<!-- vim-markdown-toc GFM -->

* [LVS](#lvs)
    * [Acronyms](#acronyms)
    * [LVS 工作模式](#lvs-工作模式)
        * [NAT](#nat)
        * [DR](#dr)
        * [TUN](#tun)
    * [Configuration](#configuration)
        * [DR](#dr-1)
        * [NAT](#nat-1)

<!-- vim-markdown-toc -->

# LVS

Linux Virtual Server.

- User Space: ipvsadm
- Kernel: ipvs

## Acronyms

- C: Client
- I: Internet
- R: Router
- VS: Virtual Server
- RS: Real Server
- RSs: Real Servers

## LVS 工作模式

1. NAT (Network Address Translation)
2. DR
3. TUN (Tunnel)

### NAT

- 集群节点 (VS, RSs) 处于同一个网络环境中.
- VS 修改网络层和传输层地址(ip, port).
- RS 将网关指向 VS (负载均衡调度器).
- RS 的 IP 通常是私有 IP, 仅用于各个集群节点通信.
- 支持端口映射.
- 进出流量都要经过 VS, 压力较大.

1. In(DNAT): C -> I -> R -> VS -> RSs
2. Out(SNAT): RS -> VS -> R -> I -> C

### DR

- VS 和 RS 处于同一广播域 (局域网) 中.
- VS 只负责处理入站请求, 压力最小.
- RS 将网关指向真实路由器.
- VS 修改数据链路层地址 (mac), 不支持端口映射.

C 发给 VS 的数据包的目标 MAC 地址被修改成
RS 的 MAC, RS 有一个仅自己可见的和 VS 一样的 IP 地址.

1. In: C -> I -> R -> VS -> RSs
2. Out: RS -> R -> I -> C

### TUN

- 为了找到 RS, VS 在数据包中附加额外的协议头.
- 集群所有节点都必须直接/间接拥有公网地址.
- RS 将网关指向真实路由器.
- 不支持端口映射.
- VS 和 RS 必须同时开启隧道功能.
- 入站由 VS 完成, 出站由 RS 完成.
- 压力较大.

1. In: C -> I -> R1 -> VS -> R1 -> I -> R2 -> RS
2. Out: RS -> R2 -> I -> C

## Configuration

### DR

- C - `10.10.10.240` (VMNet1)
- VS
    + eth0 - `10.10.10.11`
    + eth0:0 - `10.10.10.100`
- RS1
    + eth0 - `10.10.10.12`
    + lo:0 - `10.10.10.100`
- RS2
    + eth0 - `10.10.10.13`
    + lo:0 - `10.10.10.100`

为了使 RS 拥有仅自己可见的和 VS 一样的 IP,
需要修改 RS 的 ARP 通信行为 (通过调整内核参数实现).

`arp-ignore`:
- `0` - 只要本机配置有相应的 IP 地址就响应.
- `1` - 仅在请求的目标地址配置在请求到达的网络接口上时,
才响应.

`arp-announce`:
- `0` - 将本机任何网络接口上的任何地址都向外通告.
- `1` - 尽可能避免向目标网络通告与其网络不匹配的地址信息.
- `2` - 仅向目标网络通告与其网络相匹配的地址信息.

<details>
<summary>
    <code>DR 配置 - VS</code>
</summary>

```sh
# 关闭 NetworkManager
service NetworkManager stop && chkconfig NetworkManager off

# 配置网卡
cd /etc/sysconfig/network-scripts/
cp -a ifcfg-eth0 ifcfg-eth0:0
vim !$
#DEVICE=eth0:0
#IPADDR=10.10.10.100
#NETMASK=255.255.255.0
ifup eth0:0
service network restart

# 调整内核参数
vim /etc/sysctl.conf
#net.ipv4.conf.all.send_redirects = 0
#net.ipv4.conf.default.send_redirects = 0
#net.ipv4.conf.eth0.send_redirects = 0
sysctl -p

# 加载内核模块
modprobe ip_vs

# 安装 ipvsadm
yum install ipvsadm -y

ipvsadm -v
ipvsadm -A -t 10.10.10.100:80 -s rr
# -A Add VS
# -t 指定 VS 地址
# -s 选择调度算法 rr 即 round robin 算法
ipvsadm -a -t 10.10.10.100:80 -r 10.10.10.12:80 -g
# -a Add RS
# -t 指定 VS 地址
# -r 指定 RS 地址
# -g DR 模式
ipvsadm -a -t 10.10.10.100:80 -r 10.10.10.13:80 -g
ipvsadm -Ln
# 持久化
service ipvsadm save
chkconfig ipvsadm on

ipvsadm -Ln --stats
```

</details>

<details>
<summary>
    <code>DR 配置 - RS1</code>
</summary>

```sh
# 关闭 NetworkManager
service NetworkManager stop && chkconfig NetworkManager off

# 配置网卡
cd /etc/sysconfig/network-scripts/
cp -a ifcfg-lo ifcfg-lo:0
vim !$
#DEVICE=lo:0
#IPADDR=10.10.10.100
#NETMASK=255.255.255.255

# 调整内核参数
vim /etc/sysctl.conf
#net.ipv4.conf.all.arp_ignore = 1
#net.ipv4.conf.all.arp_announce = 2
#net.ipv4.conf.default.arp_ignore = 1
#net.ipv4.conf.default.arp_announce = 2
#net.ipv4.conf.lo.arp_ignore = 1
#net.ipv4.conf.lo.arp_announce = 2
sysctl -p

ifup lo:0
service network restart

# 添加入站数据包到 lo:0 接口的路由
route add -host 10.10.10.100 dev lo:0
route -n
echo "route add -host 10.10.10.100 dev lo:0" >> /etc/rc.local
```

</details>

### NAT

- C - `20.20.20.22` (VMNet1)
- VS
    + eth0 - `20.20.20.11`
    + eth1 - `10.10.10.11`
- RS1
    + eth0 - `10.10.10.12`
- RS2
    + eth0 - `10.10.10.13`

<details>
<summary>
    <code>NAT 配置 - VS</code>
</summary>

```sh
# 关闭 NetworkManager
service NetworkManager stop && chkconfig NetworkManager off

# 配置网卡
cd /etc/sysconfig/network-scripts/
vim ifcfg-eth0
#DEVICE=eth0
#ONBOOT=yes
#BOOTPROTO=static
#IPADDR=20.20.20.11
#NETMASK=255.255.255.0
vim ifcfg-eth1
#DEVICE=eth1
#ONBOOT=yes
#BOOTPROTO=static
#IPADDR=10.10.10.11
#NETMASK=255.255.255.0
service network restart

# 加载内核模块
modprobe ip_vs

# 安装 ipvsadm
yum install ipvsadm -y

# 调整内核参数
vim /etc/sysctl.conf
#net.ipv4.ip_forward = 1
sysctl -p

# 设置 iptables
service iptables start
chkconfig iptables on
iptables -F
iptables -t nat -A POSTROUTING \
  -s 10.10.10.0/24 -o eth0 \
  -j SNAT --to-source 20.20.20.11
#iptables -t nat -L
service iptables save

ipvsadm -A -t 20.20.20.11:80 -s rr
# -A Add VS
# -t 指定 VS 地址
# -s 选择调度算法 rr 即 round robin 算法
ipvsadm -a -t 20.20.20.11:80 -r 10.10.10.12:80 -m
# -a Add RS
# -t 指定 VS 地址
# -r 指定 RS 地址
# -m NAT 模式
ipvsadm -a -t 20.20.20.11:80 -r 10.10.10.13:8080 -m
ipvsadm -Ln
# 持久化
service ipvsadm save
chkconfig ipvsadm on

ipvsadm -Ln --stats
```

</details>

<details>
<summary>
    <code>NAT 配置 - RS1</code>
</summary>

```sh
# 关闭 NetworkManager
service NetworkManager stop && chkconfig NetworkManager off

# 配置网卡
cd /etc/sysconfig/network-scripts/
vim ifcfg-eth0
#DEVICE=eth0
#ONBOOT=yes
#BOOTPROTO=static
#IPADDR=10.10.10.12
#NETMASK=255.255.255.0
service network restart

route add default gw 10.10.10.11
# or
echo "GATEWAY=10.10.10.11" >> /etc/sysconfig/network-scripts/ifcfg-eth0
service network restart

route -n

# 启动对应服务如 httpd
```

</details>

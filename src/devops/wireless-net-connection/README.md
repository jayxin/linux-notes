# 配置无线网卡

步骤:

1. 确认无线网卡被识别和驱动, 即确保安装了网卡
需要的驱动 (`.ko`) 和固件 (`.bin`).
2. 检查无线接口状态 (是否被禁用).
3. 扫描可用无线网络.
4. 连接到目标网络.
5. 测试连接.

## 确认无线网卡被识别和驱动

```sh
# 对 PCE(e) 无线网卡
lspci -k | grep -iA3 net

# 对 USB 无线网卡
lsusb

# 列出所有网络接口, 查找以 wlan, wlp, wlo 等开头的
ip link

# 列出无线设备
iw dev
```

检查驱动:

- 在 `lspci -k` 的输出中, 查看无线网卡对应的行下是否有
`Kernel driver in use...`.
- 驱动正常加载, 则 `ip link` 和 `iw dev` 会显示无线接口.

没有驱动:

```sh
# 检查内核模块
lsmod | grep <driver_name>

# 加载驱动
sudo modprobe <driver_name>

# 安装驱动
# 需要根据网卡型号 (lspci, lsusb) 安装相关驱动
# 1. 安装来自发行版仓库的 firmware 包
# 2. 从源码编译和安装驱动
```

## 检查无线接口状态

检查是否被硬件/软件开关禁用.

```sh
# 列出所有无线设备及其启用状态
rfkill list
# Soft blocked: yes
# Hard blocked: yes
# Soft blocked 由软件 (如 rfkill) 禁用
# Hard blocked 由物理开关或 BIOS 设置控制

# 解除 soft blocked
# 解除所有 WiFi 设备的软件禁用
sudo rfkill unblock wifi
# 解除指定 ID 设备的禁用
sudo rfkill unblock <id>
```

使网卡可以使用协议栈:

```sh
ip link set <interface> up
```

## 扫描可用无线网络

```sh
sudo iw dev <interface> scan | grep SSID
```

## 连接到目标网络

1. 使用 `nmcli`.
2. 使用 `wpa_supplicant`.

### nmcli

```sh
# 查看已有连接
nmcli conn show

# 扫描网络
nmcli dev wifi list
# or
sudo iwlist <interface> scan

# 连接到开放网络 (无密码)
SSID_NAME="<SSID Name>"
nmcli dev wifi connect "$SSID_NAME"

# 连接到 WPA/WPA2-PSK 网络
SSID_NAME="<SSID Name>"
WIFI_PWD="<Password>"
nmcli dev wifi connect "$SSID_NAME" password "$WIFI_PWD"

# 连接到隐藏网络
CONN_NAME="<Connection Name>"
SSID_NAME="<SSID Name>"
WIFI_PWD="<Password>"
NIC_NAME="<interface>"
nmcli conn add type wifi con-name "$CONN_NAME" \
    ssid "$SSID_NAME" ifname $NIC_NAME
nmcli conn modify "$CONN_NAME" wifi-sec.key-mgmt wpa-psk
nmcli conn modify "$CONN_NAME" wifi-sec.psk "$WIFI_PWD"
nmcli conn up "$CONN_NAME"

# 管理现有连接
CONN_NAME="<Connection Name>"
nmcli conn up $CONN_NAME
nmcli conn down $CONN_NAME
nmcli conn delete $CONN_NAME
```

### wpa_supplicant

```sh
sudo vim /etc/wpa_supplicant.conf
```

`/etc/wpa_supplicant.conf`:

```conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
# 错误设置会导致扫描不到某些频段的网络
country=<Your_Country_Code>  # 例如 US, GB, CN, JP (影响可用信道)
network={
    ssid="<SSID Name>"
    psk="<Password>"  # 明文密码（不推荐）
    # 或使用加密的PSK（推荐）
    # psk=$(wpa_passphrase "<SSID Name>" "<Password>" | grep -v '#' | grep psk= | cut -d= -f2)
}
```

加密的 PSK (Pre-Shared Key) 生成:

```sh
sudo wpa_passphrase "<Your_SSID>" "<Your_Strong_Password>" | sudo tee -a /etc/wpa_supplicant.conf
```

启动 `wpa_supplicant` 关联到网络:

```sh
NIC_NAME="<interface>"
sudo wpa_supplicant -B -i $NIC_NAME -c /etc/wpa_supplicant.conf
# -B background
# -c config
```

获取 IP (通过 DHCP):
使用 `NetworkManager` 或者 `systemd-networkd`
通常不需要获取 IP, 只要 `wpa_supplicant` 关联成功,
会自动触发 DHCP.

```sh
NIC_NAME="<interface>"
sudo dhclient $NIC_NAME
# or
sudo dhcpcd $NIC_NAME
```

## 配置开机自动连接

取决于使用的网络管理服务:
- `systemd-networkd`
- `NetworkManager`
- `networking.service` + `ifupdown`

通常是配置 `/etc/network/interfaces` 或
systemd 的 Unit 文件.

在使用 NetworkManager 的系统中, `nmcli`
创建的连接默认是自动连接的.

## 测试连接

```sh
# 检查 IP
NIC_NAME="<interface>"
ip addr show $NIC_NAME

# 检查路由
ip route show

# 检查连通性
ping -c3 8.8.8.8
ping -c3 www.baidu.com
```

## 指定通过某张网卡上网

1. 使用 `ip route` 修改默认路由 (临时).
2. 修改 `metric` (永久).
3. 使用 `iptables`.
4. 禁用不需要的网卡.

### 修改默认路由

```sh
# 查看当前网卡和 IP
ip addr
# or
ifconfig

# 查看当前路由表
ip route show

# 删除原有默认路由
sudo ip route del default
# 或删除特定默认路由
GATEWAY_IP="192.168.1.1"
NIC_NAME="eth0"
sudo ip route del default via $GATEWAY_IP dev $NIC_NAME

# 添加新的默认路由
GATEWAY_IP="10.0.0.1"
NIC_NAME="wlan0"
sudo ip route add default via $GATEWAY_IP dev $NIC_NAME

# 验证
ip route show
ping -c3 8.8.8.8

# 查看出口 IP 是否变化
curl ifconfig.me
```

### 修改 metric

`metric` 越小, 优先级越高.

```sh
# 查看当前网卡 metric
ip route show

# 修改网卡 metric
CONN_NAME="<Connection Name>"
nmcli conn modify $CONN_NAME ipv4.route-metric 50
nmcli conn up $CONN_NAME
# or
sudo vim /etc/network/interfaces
# /etc/network/interfaces
#auto eth0
#iface eth0 inet dhcp
#    metric 50

# 重启网络
sudo systemctl restart networking
```

### 使用 iptables

```sh
# 添加 SNAT/MASQUERADE 规则
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

```sh
# 添加策略路由
sudo ip rule add from 192.168.1.100 table 100
sudo ip route add default via 192.168.1.1 dev eth0 table 100
```

### 禁用不需要的网卡

```sh
sudo ip link set eth0 down
# or
nmcli conn down "<Connection Name>"
```

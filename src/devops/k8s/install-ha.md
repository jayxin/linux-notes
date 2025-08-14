# Install-High Availability

k8s 高可用拓扑:

- 堆叠. etcd 部署在 Master 节点上, 每个 Master 节点上都有一个 etcd 实例,
此 etcd 实例只和该节点上的 api server 通信, 多个 Master 节点组成 etcd 集群.
    + 优点: 配置简单, 需要的机器较少, 易于管理.
    + 缺点: 存在耦合风险.
- etcd 外置. 使用单独节点部署 etcd 集群.
    + 优点: 解耦 etcd 和控制平面组件, 失去 etcd 实例或控制平面组件的影响
    较小.
    + 缺点: 配置较复杂, 需要的机器数量较多.

高可用需要提供负载均衡器, 对外提供一个统一的 VIP 给工作节点的 kubelet
访问, 负载均衡器通过反向代理到多个 master 节点上的 api server 实现
api server 的负载均衡和高可用.

这里使用堆叠拓扑部署高可用的 k8s 集群, 负载均衡器使用 haproxy+keepalive
实现.

## 环境说明

- Kubernetes: v1.33.3
- OS: Rocky Linux 9.5
    + k8s-master0 - `192.168.90.50`
    + k8s-master1 - `192.168.90.51`
    + k8s-master2 - `192.168.90.52`
    + vip - `192.168.90.55`

## 环境准备

<details>
<summary>环境准备</summary>

```sh
# ---主机名配置---
# k8s-master0
hostnamectl set-hostname k8s-master0
# k8s-master1
hostnamectl set-hostname k8s-master1
# k8s-master2
hostnamectl set-hostname k8s-master2

# ---静态 IP---
# k8s-master0, k8s-master1, k8s-master2
# 将 IP 和主机名映射关系添加到 /etc/hosts
#192.168.90.50 k8s-master0
#192.168.90.51 k8s-master1
#192.168.90.52 k8s-master2

# ---SSH 公钥认证---
# k8s-master0
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master1
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master2

# k8s-master1
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master0
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master2

# k8s-master2
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master0
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master1

# ---时间同步--- k8s-master0, k8s-master1, k8s-master2
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone
systemctl restart chronyd
systemctl enable chronyd

# ---关闭防火墙--- k8s-master0, k8s-master1, k8s-master2
systemctl stop firewalld
systemctl disable firewalld

# ---关闭selinux--- k8s-master0, k8s-master1, k8s-master2
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# ---禁用swap分区--- k8s-master0, k8s-master1, k8s-master2
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

# ---调整内核参数--- k8s-master0, k8s-master1, k8s-master2
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl -p

# ---安装常用软件--- k8s-master0, k8s-master1, k8s-master2
dnf install wget jq psmisc vim net-tools nfs-utils socat telnet \
    device-mapper-persistent-data lvm2 git tar zip unzip curl \
    conntrack ipvsadm ipset iptables sysstat libseccomp -y

# ---开启 ipvs 转发--- k8s-master0, k8s-node0, k8s-node1
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF

modprobe -- ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack
lsmod | grep -e ip_vs -e nf_conntrack

systemctl enable --now systemd-modules-load.service
# or
systemctl restart systemd-modules-load.service
systemctl enable systemd-modules-load.service

# ---重启--- k8s-master0, k8s-master1, k8s-master2
reboot

# ---check--- k8s-master0, k8s-master1, k8s-master2
# SELinux 状态
sestatus
# swap 分区状态
cat /proc/swaps
# ipvs 内核模块加载状态
lsmod | grep ip_vs
# 防火墙状态
systemctl status firewalld
# 内核参数设置
cat /proc/sys/net/ipv4/ip_forward
```

</details>

## 安装容器引擎

<details>
<summary>安装 containerd</summary>

```sh
# ---添加docker 源--- k8s-master0, k8s-master1, k8s-master2
dnf -y install dnf-plugins-core
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# ---安装 containerd--- k8s-master0, k8s-master1, k8s-master2
dnf install containerd.io -y
```

</details>

<details>
<summary>配置 containerd</summary>

```sh
# ---生成 containerd 的默认配置--- k8s-master0, k8s-master1, k8s-master2
containerd config default | sudo tee /etc/containerd/config.toml

# ---修改配置--- k8s-master0, k8s-master1, k8s-master2
vim /etc/containerd/config.toml

#disabled_plugins = []
#[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#   SystemdCgroup = true
#
#[plugins."io.containerd.grpc.v1.cri"]
#  sandbox_image = "registry.k8s.io/pause:3.10"

# ---配置代理--- k8s-master0, k8s-master1, k8s-master2
systemctl edit containerd.service
#[Service]
#Environment="ALL_PROXY=http://192.168.90.1:7890"
#Environment="HTTP_PROXY=http://192.168.90.1:7890"
#Environment="HTTPS_PROXY=http://192.168.90.1:7890"
#Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local"
```

</details>

<details>
<summary>启动 containerd</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
systemctl enable containerd
systemctl start containerd

# ---check--- k8s-master0, k8s-master1, k8s-master2
systemctl status containerd
```

</details>

## 安装 kubelet, kubeadm 和 kubectl

<details>
<summary>配置 k8s 源</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.33/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

</details>

<details>
<summary>安装 kubelet, kubeadm 和 kubectl</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
yum install kubelet kubeadm kubectl -y
```

</details>

<details>
<summary>启动 kubelet</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
# 由于 kubelet 的配置文件还未生成, 因此这里 kubelet 会启动失败
systemctl enable --now kubelet
```

</details>

## 定义负载均衡器 haproxy+keepalive 的 Pod

### haproxy

<details>
<summary>haproxy 配置</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
mkdir -p /etc/haproxy
vim /etc/haproxy/haproxy.cfg
```

</details>

<details>
<summary>
    <code>/etc/haproxy/haproxy.cfg</code>
</summary>

```conf
# k8s-master0, k8s-master1, k8s-master2
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log stdout format raw local0
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          35s
    timeout server          35s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443
    mode tcp
    option tcplog
    default_backend apiserverbackend

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserverbackend
    option httpchk

    http-check connect ssl
    http-check send meth GET uri /healthz
    http-check expect status 200

    mode tcp
    balance     roundrobin

    server k8s-master0 192.168.90.50:6443 check verify none
    server k8s-master1 192.168.90.51:6443 check verify none
    server k8s-master2 192.168.90.52:6443 check verify none
```

</details>

<details>
<summary>haproxy 静态 Pod 定义 haproxy.yaml</summary>

```yaml
# k8s-master0, k8s-master1, k8s-master2
apiVersion: v1
kind: Pod
metadata:
  name: haproxy
  namespace: kube-system
spec:
  containers:
  - image: cr.aliyuncs.com/image-infra/haproxy:2.8
    name: haproxy
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: localhost
        path: /healthz
        port: 8443
        scheme: HTTPS
    volumeMounts:
    - mountPath: /usr/local/etc/haproxy/haproxy.cfg
      name: haproxyconf
      readOnly: true
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
    name: haproxyconf
status: {}
```

</details>

### keepalive

<details>
<summary>配置 keepalive</summary>

```sh
mkdir -p /etc/keepalived
vim /etc/keepalived/keepalived.conf
```

</details>

这里将 `k8s-master0` 定义为 keepalive 的主节点.

keepalive 配置要点:
- `state`
- `priority`
- `interface`
- `virtual_ipaddress`

<details>
<summary>
    k8s-master0: <code>/etc/keepalived/keepalived.conf</code>
</summary>

```conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 101
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        192.168.90.55
    }
    track_script {
        check_apiserver
    }
}
```

</details>

<details>
<summary>
    k8s-master1: <code>/etc/keepalived/keepalived.conf</code>
</summary>

```conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        192.168.90.55
    }
    track_script {
        check_apiserver
    }
}
```

</details>

<details>
<summary>
    k8s-master2: <code>/etc/keepalived/keepalived.conf</code>
</summary>

```conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    authentication {
        auth_type PASS
        auth_pass 42
    }
    virtual_ipaddress {
        192.168.90.55
    }
    track_script {
        check_apiserver
    }
}
```

</details>

<details>
<summary>
    keepalive 健康检查脚本 <code>/etc/keepalived/check_apiserver.sh</code>
</summary>

```sh
#!/bin/sh

# k8s-master0, k8s-master1, k8s-master2
APISERVER_PORT=8443

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl -sfk --max-time 2 https://localhost:${APISERVER_PORT}/healthz -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_PORT}/healthz"
```

</details>

<details>
<summary>
    keepalive 静态 Pod 定义: <code>keepalived.yaml</code>
</summary>

```yaml
# k8s-master0, k8s-master1, k8s-master2
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: keepalived
  namespace: kube-system
spec:
  containers:
  - image: cr.aliyuncs.com/image-infra/keepalived:2.0.20
    name: keepalived
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_BROADCAST
        - NET_RAW
    volumeMounts:
    - mountPath: /usr/local/etc/keepalived/keepalived.conf
      name: config
    - mountPath: /etc/keepalived/check_apiserver.sh
      name: check
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/keepalived/keepalived.conf
    name: config
  - hostPath:
      path: /etc/keepalived/check_apiserver.sh
    name: check
status: {}
```

</details>

<details>
<summary>复制 Pod 的定义文件到指定位置</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
cp keepalived.yaml haproxy.yaml /etc/kubernetes/manifests

# kubelet 会检测此路径并创建 Pod
```

</details>

## 初始化集群

在 `k8s-master0` 创建 `kubeadm-config.yaml` 配置文件.

- `serviceSubnet` 是 Cluster IP 的地址段.
- `podSubnet` 是 Pod IP 的地址段.
- `kubernetesVersion` 指定 k8s 版本.
- `controlPlaneEndpoint` 指定上面的 vip 和端口. 端口和 `haproxy.cfg`
中配置的 haproxy 的监听端口一致.
- `localAPIEndpoint` 是本机 kube-apiserver 的监听地址和端口.

<details>
<summary>
    <code>kubeadm-config.yaml</code>
</summary>

```yaml
# k8s-master0
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789kkkkkk
  ttl: 24h0m0s
  usages:
  - signing
  - authentication

nodeRegistration:
  name: "k8s-master0"
  criSocket: "unix:///var/run/containerd/containerd.sock"
  imagePullPolicy: "IfNotPresent"

localAPIEndpoint:
  advertiseAddress: "192.168.90.50"
  bindPort: 6443

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
kubernetesVersion: "1.33.3"
controlPlaneEndpoint: "192.168.90.55:8443"
certificatesDir: "/etc/kubernetes/pki"
imageRepository: "cr.aliyuncs.com/image-infra"
clusterName: "kubernetes"
caCertificateValidityPeriod: 876000h0m0s
certificateValidityPeriod: 87600h0m0s
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
```

</details>

<details>
<summary>初始化集群</summary>

```sh
# k8s-master0
kubeadm init --config kubeadm-config.yaml --upload-certs --v=5
```

</details>

<details>
<summary>配置  kubectl</summary>

```sh
# k8s-master0
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

</details>

<details>
<summary>k8s-master1 和 k8s-master2 加入集群</summary>

```sh
# k8s-master1, k8s-master2
kubeadm join 192.168.90.55:8443 --token abcdef.0123456789kkkkkk \
        --discovery-token-ca-cert-hash sha256:53950443f3c42bbf8c6aa17289327e03555649d9ce0b11f6e5df1f87fc67956f \
        --control-plane --certificate-key 98b0dd21cd66398aa656d2724d56c583ec7131808c42845d4fb76cc51c243d9d
```

</details>

<details>
<summary>工作节点加入集群</summary>

```sh
# 前提是工作节点已完成必要的准备工作
kubeadm join 192.168.90.55:8443 --token abcdef.0123456789kkkkkk \
        --discovery-token-ca-cert-hash sha256:53950443f3c42bbf8c6aa17289327e03555649d9ce0b11f6e5df1f87fc67956f
```

</details>

<details>
<summary>check</summary>

```sh
# k8s-master0, k8s-master1, k8s-master2
kubectl get node
# 此时节点是 NotReady 状态, 因为还未安装网络插件
```

</details>

## 安装网络插件 calico

<details>
<summary>创建 calico operator</summary>

```sh
# k8s-master0
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/tigera-operator.yaml
```

</details>

<details>
<summary>
    <code>custom-resources.yaml</code>
</summary>

```yaml
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  registry: "cr.aliyuncs.com/"
  imagePath: "image-infra"
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      # 这里的 cidr 和 kubeadm-config.yaml 中的 podSubnet 保持一致
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}

---

# Configures the Calico Goldmane flow aggregator.
apiVersion: operator.tigera.io/v1
kind: Goldmane
metadata:
  name: default

---

# Configures the Calico Whisker observability UI.
apiVersion: operator.tigera.io/v1
kind: Whisker
metadata:
  name: default
```

</details>

<details>
<summary>创建 calico 资源</summary>

```sh
kubectl create -f custom-resources.yaml
```

</details>

等待一段时间即可.

<details>
<summary>check</summary>

```sh
kubectl get po -A
kubectl get no
```

</details>

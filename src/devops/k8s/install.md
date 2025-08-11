<!-- vim-markdown-toc GFM -->

* [Kubernetes 集群安装](#kubernetes-集群安装)
    * [环境说明](#环境说明)
    * [环境准备](#环境准备)
    * [安装容器引擎](#安装容器引擎)
        * [常用地址](#常用地址)
        * [docker](#docker)
        * [containerd](#containerd)
        * [cri-dockerd](#cri-dockerd)
    * [k8s](#k8s)
        * [安装](#安装)
        * [配置和启动集群](#配置和启动集群)
        * [安装网络插件](#安装网络插件)

<!-- vim-markdown-toc -->

# Kubernetes 集群安装

[k8s官网](https://kubernetes.io/)

## 环境说明

- Kubernetes: v1.33.3
- OS: RedHat Enterprise Linux 10
    - k8s-master0
    - k8s-node0
    - k8s-node1

## 环境准备

<details>
<summary>环境准备</summary>

```sh
# ---主机名配置---
# k8s-master0
hostnamectl set-hostname k8s-master0
# k8s-node0
hostnamectl set-hostname k8s-node0
# k8s-node1
hostnamectl set-hostname k8s-node1

# ---静态 IP---
# 将 IP 和主机名映射关系添加到 /etc/hosts

# ---SSH 公钥认证---
# k8s-master0
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node0
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node1

# ---时间同步--- k8s-master0, k8s-node0, k8s-node1
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo 'Asia/Shanghai' > /etc/timezone
systemctl restart chronyd
systemctl enable chronyd

# ---关闭防火墙--- k8s-master0, k8s-node0, k8s-node1
systemctl stop firewalld
systemctl disable firewalld

# ---关闭selinux--- k8s-master0, k8s-node0, k8s-node1
sed -i 's/enforcing/disabled/' /etc/selinux/config

# ---禁用swap分区--- k8s-master0, k8s-node0, k8s-node1
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

# ---调整内核参数--- k8s-master0, k8s-node0, k8s-node1
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
vm.overcommit_memory=1
fs.inotify.max_user_instances=8192
fs.inotify.max_user_instances=8192
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl -p

# ---安装常用软件--- k8s-master0, k8s-node0, k8s-node1
dnf install wget jq psmisc vim net-tools nfs-utils socat telnet \
    device-mapper-persistent-data lvm2 git tar zip unzip curl \
    conntrack ipvsadm ipset iptables sysstat libseccomp -y

# ---开启 ipvs 转发--- k8s-master0, k8s-node0, k8s-node1
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack

systemctl enable --now systemd-modules-load.service
# or
systemctl restart systemd-modules-load.service
systemctl enable systemd-modules-load.service

# ---重启--- k8s-master0, k8s-node0, k8s-node1
reboot
```

</details>

## 安装容器引擎

### 常用地址

- [阿里云](https://cr.console.aliyun.com)
- [清华](https://mirrors.tuna.tsinghua.edu.cn/docker-ce/)
- [华为](https://mirrors.huaweicloud.com/docker-ce/)
- [中科大](http://mirrors.ustc.edu.cn/)
- [中科大github](https://github.com/ustclug/mirrorrequest)
- [Azure中国](http://mirror.azure.cn/)
- [Azure中国github](https://github.com/Azure/container-service-for-azure-china)
- [DockerHub](https://hub.docker.com/)
- [Google](https://console.cloud.google.com/gcr/images/google-containers/GLOBAL)
- [CoreOS](https://quay.io/repository/)
- [RedHat](https://access.redhat.com/containers)

### docker

<details>
<summary>下载</summary>

```sh
# k8s-master0, k8s-node0, k8s-node1
wget https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/docker-28.3.3.tgz
tar -xzf docker-28.3.3.tgz
cp -rfa docker/* /usr/bin
# or
dnf config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
dnf install docker-ce containerd -y
```

</details>

<details>
<summary>
    <code>docker.service</code>
</summary>

```sh
cat > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket containerd.service

[Service]
Type=notify

ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID

TimeoutSec=0

RestartSec=2
Restart=always

StartLimitBurst=3
StartLimitInterval=60s

LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TasksMax=infinity

Delegate=yes

KillMode=process

OOMScoreAdjust=-500

[Install]
WantedBy=multi-user.target
EOF
```

</details>

<details>
<summary>
    <code>docker.socket</code>
</summary>

```sh
cat > /etc/systemd/system/docker.socket <<EOF
[Unit]
Description=Docker Socket for the API

[Socket]
ListenStream=/var/run/docker.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```

</details>

<details>
<summary>配置</summary>

```sh
cat > /etc/docker/daemon.json << EOF
{
    "builder": {
        "gc": {
            "defaultKeepStorage": "20GB",
            "enabled": true
        }
    },
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": [
        "https://docker.hpcloud.cloud",
        "https://docker.unsee.tech",
        "http://mirrors.ustc.edu.cn",
        "https://docker.chenby.cn",
        "http://mirror.azure.cn",
        "https://dockerpull.org",
        "https://hub.rat.dev",
        "https://docker.1panel.live",
        "https://docker.m.daocloud.io",
        "https://registry.dockermirror.com",
        "https://docker.aityp.com/",
        "https://docker.anyhub.us.kg",
        "https://dockerhub.icu",
        "https://docker.awsl9527.cn"
    ],
    "insecure-registries": ["https://harbor.flyfish.com"],
    "max-concurrent-downloads": 10,
    "log-driver": "json-file",
    "log-level": "warn",
    "log-opts": {
        "max-size": "10m",
        "max-file": "3"
    },
    "data-root": "/var/lib/docker"
}
EOF
```

</details>

<details>
<summary>启动</summary>

```sh
groupadd docker
systemctl enable --now docker.socket
systemctl enable --now docker.service
systemctl restart docker
docker version
```

</details>

### containerd

<details>
<summary>containerd.service</summary>

```sh
cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/bin/containerd

Type=notify

Delegate=yes

KillMode=process

Restart=always
RestartSec=5

LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=1048576

TasksMax=infinity

OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF
```

</details>

<details>
<summary>启动</summary>

```sh
systemctl enable --now containerd.service
containerd --version
```

</details>

### cri-dockerd

<details>
<summary>下载</summary>

```sh
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.15/cri-dockerd-0.3.15.amd64.tgz
tar xzf cri-dockerd-0.3.15.amd64.tgz
cp -a cri-dockerd/cri-dockerd /usr/bin
chmod +x /usr/bin/cri-dockerd
```

</details>

<details>
<summary>
    <code>cri-docker.service</code>
</summary>

```sh
cat > /etc/systemd/system/cri-docker.service <<EOF
[Unit]
Description=CRI Interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify

ExecStart=/usr/bin/cri-dockerd --network-plugin=cni --pod-infra-container-image=registry.aliyuncs.com/google_containers/pause:3.9
ExecReload=/bin/kill -s HUP $MAINPID

TimeoutSec=0

RestartSec=2
Restart=always

StartLimitBurst=3
StartLimitInterval=60s

LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

TasksMax=infinity

Delegate=yes

KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```

</details>

<details>
<summary>
    <code>cri-docker.socket</code>
</summary>

```sh
cat > /etc/systemd/system/cri-docker.socket <<EOF
[Unit]
Description=CRI Docker Socket for the API	
Partof=cri-docker.service
 
[Socket]
ListenStream=%t/cri-dockerd.sock

SocketMode=0660
SocketUser=root
SocketGroup=docker
 
[Install]
WantedBy=sockets.target
EOF
```

</details>

<details>
<summary>启动</summary>

```sh
systemctl daemon-reload
systemctl enable --now cri-docker
```

</details>

## k8s

### 安装

<details>
<summary>仓库配置</summary>

```sh
# k8s-master0, k8s-node0, k8s-node1
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.33/rpm/repodata/repomd.xml.key
EOF
```

</details>

<details>
<summary>查找所有可用版本</summary>

```sh
dnf list kubeadm.x86_64 --showduplicates | sort -r
# or
yum list kubelet.x86_64 --showduplicates | sort -r | grep 1.33
```

</details>

<details>
<summary>安装 kubeadm, kubelet, kubectl</summary>

```sh
# k8s-master0, k8s-node0, k8s-node1
# 指定要安装的版本
dnf install kubeadm-1.33* kubelet-1.33* kubectl-1.33* -y
# or
# 最新版本
yum install -y kubeadm kubelet kubectl
```

</details>

### 配置和启动集群

<details>
<summary>
    <code>/etc/sysconfig/kubelet</code>
</summary>

```sh
# k8s-master0, k8s-node0, k8s-node1
# 实现 docker 使用 cgroupdriver 和 kubelet 使用的 cgroup 的一致性
cat >> /etc/sysconfig/kubelet << EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
EOF
```

</details>

<details>
<summary>启动</summary>

```sh
systemctl enable --now kubelet

kubeadm version
```

</details>

<details>
<summary>获取各个组件的镜像</summary>

```sh
# k8s-master0, k8s-node0, k8s-node1
kubeadm config images pull \
    --image-repository registry.aliyuncs.com/google_containers \
    --cri-socket=unix:///var/run/cri-dockerd.sock
```

</details>

<details>
<summary>集群初始化</summary>

```sh
# k8s-master0
kubeadm init \
    --kubernetes-version=v1.33.3 \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.96.0.0/12 \
    --apiserver-advertise-address=192.168.250.50 \
    --image-repository registry.aliyuncs.com/google_containers \
    --cri-socket=unix:///var/run/cri-dockerd.sock

# --apiserver-advertise-address 集群通告地址
# --image-repository 默认拉取镜像地址是 k8s.gcr.io
# --kubernetes-version k8s版本, 与上面安装的版本一致
# --service-cidr Service 网络, Pod 统一访问入口
# --pod-network-cidr Pod 网络, 与下面部署的 CNI 网络组件 yaml 中保持一致

# 创建 kubectl 配置文件
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

</details>

<details>
<summary>工作节点加入 k8s 集群</summary>

```sh
# k8s-node0, k8s-node1
kubeadm join 192.168.250.50:6443 --token kkkkcb.nqqqlnaibfxvobo4 \
        --discovery-token-ca-cert-hash \
        sha256:6668a0eacc6bedad880804ae5c6e5cd0195fb9f6f4437175e9e93ff93c364c28 \
        --cri-socket=unix:///var/run/cri-dockerd.sock
```

</details>

<details>
<summary>查看集群</summary>

```sh
# k8s-master0
kubectl get nodes
kubectl get node -o wide
# 此时状态显示是 NotReady, 因为目前还未安装网络插件

kubectl get po -n kube-system
```

</details>

### 安装网络插件

<details>
<summary>安装 calico 网络插件</summary>

```sh
# Install the Tigera Calico operator and custom resource definitions.
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/tigera-operator.yaml

# Install Calico by creating the necessary custom resource
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/custom-resources.yaml
```

</details>

<details>
<summary>
    <code>calico.yaml</code>
</summary>

```sh
wget --no-check-certificate https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

vim calico.yaml
#- name: CALICO_IPV4POOL_CIDR
#  value: "10.244.0.0/16"
#- name: IP_AUTODETECTION_METHOD
#  value: "interface=ens33"

kubectl apply -f calico.yaml
```

</details>

<details>
<summary>查看状态</summary>

```sh
# k8s-master0
# 等待一段时间后查看状态
# node
kubectl get nodes

# pod
kubectl get pod -n kube-system
```

</details>

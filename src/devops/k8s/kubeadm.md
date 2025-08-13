# kubeadm

## kubeadm 初始化集群的流程

假设:
- 已安装和配置好容器引擎 (`docker` or `containerd`)
- 系统已经做了初始化的配置
- 已安装 `kubeadm`, `kubelet`, `kubectl`

<details>
<summary>帮助-kubeadm init 帮助</summary>

```sh
kubeadm init -h

#preflight                     Run pre-flight checks
#certs                         Certificate generation
#  /ca                           Generate the self-signed Kubernetes CA to provision identities for other Kubernetes components
#  /apiserver                    Generate the certificate for serving the Kubernetes API
#  /apiserver-kubelet-client     Generate the certificate for the API server to connect to kubelet
#  /front-proxy-ca               Generate the self-signed CA to provision identities for front proxy
#  /front-proxy-client           Generate the certificate for the front proxy client
#  /etcd-ca                      Generate the self-signed CA to provision identities for etcd
#  /etcd-server                  Generate the certificate for serving etcd
#  /etcd-peer                    Generate the certificate for etcd nodes to communicate with each other
#  /etcd-healthcheck-client      Generate the certificate for liveness probes to healthcheck etcd
#  /apiserver-etcd-client        Generate the certificate the apiserver uses to access etcd                      生成apisserver用来访问etcd的证书
#  /sa                           Generate a private key for signing service account tokens along with its public key   生成用于签名服务帐户令牌及其公钥的私钥
#kubeconfig                    Generate all kubeconfig files necessary to establish the control plane and the admin kubeconfig file  生成建立控制平面所需的所有kubecconfig文件和admin kubecconfig文件
#  /admin                        Generate a kubeconfig file for the admin to use and for kubeadm itself   为管理员和kubeadm本身生成一个kubecconfig文件
#  /super-admin                  Generate a kubeconfig file for the super-admin  为超级管理员生成kubecconfig文件
#  /kubelet                      Generate a kubeconfig file for the kubelet to use *only* for cluster bootstrapping purposes  为kubelet生成一个kubecconfig文件，以便*仅*用于集群引导
#  /controller-manager           Generate a kubeconfig file for the controller manager to use   生成一个kubecconfig文件供控制器管理器使用
#  /scheduler                    Generate a kubeconfig file for the scheduler to use   生成一个kubecconfig文件供调度器使用
#etcd    Generate static Pod manifest file for local etcd
#  /local Generate the static Pod manifest file for a local, single-node local etcd instance
#control-plane                 Generate all static Pod manifest files necessary to establish the control plane   生成建立控制平面所需的所有静态Pod清单文件
#  /apiserver Generates the kube-apiserver static Pod manifest   生成kube- apisserver静态Pod清单
#  /controller-manager Generates the kube-controller-manager static Pod manifest   生成kube-controller-manager静态Pod清单
#  /scheduler Generates the kube-scheduler static Pod manifest
#kubelet-start                 Write kubelet settings and (re)start the kubelet
#upload-config    Upload the kubeadm and kubelet configuration to a ConfigMap
#  /kubeadm    Upload the kubeadm ClusterConfiguration to a ConfigMap
#  /kubelet    Upload the kubelet component config to a ConfigMap
#upload-certs    Upload certificates to kubeadm-certs                                                 上传证书到kubeadm-cert
#mark-control-plane    Mark a node as a control-plane
#bootstrap-token    Generates bootstrap tokens used to join a node to a cluster
#kubelet-finalize    Updates settings relevant to the kubelet after TLS bootstrap
#  /enable-client-cert-rotation    Enable kubelet client certificate rotation                                   启用kubelet客户端证书轮换
#addon    Install required addons for passing conformance tests
#  /coredns    Install the CoreDNS addon to a Kubernetes cluster
#  /kube-proxy    Install the kube-proxy addon to a Kubernetes cluster
#show-join-command    Show the join command for control-plane and worker node
```

</details>

<details>
<summary>kubeadm 配置-生成默认配置并调整(可选)</summary>

```sh
kubeadm config print init-defaults > /etc/kubeadm-init-config.yaml
```

</details>

<details>
<summary>
    <code>/etc/kubeadm-init-config.yaml</code>
</summary>

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
# 默认证书存储路径
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
controlPlaneEndpoint: 10.62.1.180:8443
#image-repository registry.aliyuncs.com/google_containers
imageRepository: 172.16.4.39:8090/google_containers
kubernetesVersion: v1.32.2
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
scheduler: {}
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 0h0m0s
  usages:
  - signing
  - authentication
nodeRegistration:
  # 使用 containerd 作为 CRI
  criSocket: unix:///var/run/containerd/containerd.sock
  # 使用 docker 作为CRI
  #criSocket: /var/run/dockershim.sock
  # 使用 cri-docker 作为CRI
  #criSocket: unix:///var/run/cri-dockerd.sock 
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
# 优化 IPVS 性能
ipvs:
  scheduler: "rr"  # 轮询调度算法
  minSyncPeriod: "1s"
  syncPeriod: "30s"
  # 启用 IPVS 的严格 ARP 模式
  strictARP: true
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 使用 systemd 作为 Cgroup 驱动
cgroupDriver: systemd
# 启用 kubelet 的自动证书轮换
rotateCertificates: true
# 设置 kubelet 的最大 Pod 数量
maxPods: 250
# 优化 kubelet 的资源分配
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
systemReserved:
  cpu: "1000m"
  memory: "1Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
# 启用 kubelet 的日志轮换
logging:
  format: "json"
  flushFrequency: "5s"
  verbosity: 2
```

</details>

<details>
<summary>镜像下载</summary>

```sh
# 下载 scheduler, apiserver 等集群组件的镜像
kubeadm config images pull [--config /etc/kubeadm-init-config.yaml]
```

</details>

<details>
<summary>检查运行环境</summary>

```sh
# 检查容器引擎、端口占用情况、内核参数的设置等
kubeadm init phase preflight [--config /etc/kubeadm-init-config.yaml]
```

</details>

<details>
<summary>证书生成</summary>

```sh
# 生成所有
kubeadm init phase certs all [--config /etc/kubeadm-init-config.yaml]

# 根证书
kubeadm init phase certs/ca
# apiserver 的证书
kubeadm init phase certs/apiserver
kubeadm init phase certs/apiserver-kubelet-client
kubeadm init phase certs/front-proxy-ca
kubeadm init phase certs/front-proxy-client
kubeadm init phase certs/etcd-ca
kubeadm init phase certs/etcd-server
kubeadm init phase certs/etcd-peer
kubeadm init phase certs/etcd-healthcheck-client
kubeadm init phase certs/apiserver-etcd-client
kubeadm init phase certs/sa
```

</details>

<details>
<summary>kubeconfig 文件生成</summary>

```sh
# 配置各组件和 apiserver 的通信认证, 需要确保权限为 600
kubeadm init phase kubeconfig all [--config /etc/kubeadm-init-config.yaml]

# 集群管理员凭证
kubeadm init phase kubeconfig/admin
# kubelet 和 apiserver 通信凭证
kubeadm init phase kubeconfig/kubelet
# controller-manager 和 apiserver 通信凭证
kubeadm init phase kubeconfig/controller-manager
# scheduler 和 apiserver 通信凭证
kubeadm init phase kubeconfig/scheduler
```

</details>

<details>
<summary>Pod 定义-为本地 etcd 生成静态 Pod 清单文件</summary>


```sh
# 在 /etc/kubernetes/manifests/etcd.yaml 定义 etcd 的 Pod 配置
kubeadm init phase etcd/local [--config /etc/kubeadm-init-config.yaml]
```

</details>

<details>
<summary>Pod 定义-生成控制平面的静态 Pod 清单文件</summary>

```sh
kubeadm init phase control-plane all [--config /etc/kubeadm-init-config.yaml]

kubeadm init phase control-plane/apiserver
kubeadm init phase control-plane/controller-manager
kubeadm init phase control-plane/scheduler
```

</details>

<details>
<summary>配置 kubelet 并启动之</summary>

```sh
# kubelet 的作用:
# - Static Pod 管理: kubelet 会监控 /etc/kubernetes/manifests 目录,
#   自动加载和运行其中的 Pod 定义文件
# - 自托管机制: 控制平面(apiserver, scheduler 等)组件以 Pod 形式运行,
#   但是不受 Kubernetes 自身调度管理
# /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# --pod-manifest-path 用于指定静态 Pod 的路径
kubeadm init phase kubelet-start [--config /etc/kubeadm-init-config.yaml]
# 生成运行时配置 /var/lib/kubelet/config.yaml
# 生成环境变量 /var/lib/kubelet/kubeadm-flags.env, 比如 Pod 的 CIDR 等

# 检查
crictl ps -a
systemctl status kubelet
```

</details>

<details>
<summary>上传 kubeadm 和 kubelet 配置到 ConfigMap</summary>

```sh
kubeadm init phase upload-config all [--config /etc/kubeadm-init-config.yaml]

kubeadm init phase upload-config/kubeadm
kubeadm init phase upload-config/kubelet

# 检查
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get pods
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get cm
```

</details>

<details>
<summary>证书上传到 kubeadm-certs</summary>

```sh
# Upload control-plane certificates to the kubeadm-certs Secret.
# 需要保存 certificate key, 后续其他控制平面节点加入需要使用
kubeadm init phase upload-certs --upload-certs [--config /etc/kubeadm-init-config.yaml]

# 检查
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get secret
```

</details>

<details>
<summary>将节点标记为控制平面</summary>

```sh
# 添加标签 node-role.kubernetes.io/control-plane
# 添加污点 node-role.kubernetes.io/control-plane:NoSchedule
kubeadm init phase mark-control-plane [--config /etc/kubeadm-init-config.yaml]

# 检查
kubectl --kubeconfig /etc/kubernetes/admin.conf describe node k8s-master0
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get node
```

</details>

<details>
<summary>生成用于向集群加入节点的引导令牌</summary>

```sh
# bootstrap token 是一种用于新的节点加入集群时进行身份验证的令牌
kubeadm init phase bootstrap-token [--config /etc/kubeadm-init-config.yaml]

# check
kubeadm token list --kubeconfig /etc/kubernetes/admin.conf
# bootstrap-token 有过期时间, 确保在其过期前使用将节点加入集群
# 可用 kubeadm token create 命令创建 bootstrap-token
```

</details>

<details>
<summary>更新 kubelet 配置</summary>

```sh
# 确保 kubelet 的客户端证书可以自动轮换
kubeadm init phase kubelet-finalize/enable-client-cert-rotation [--config /etc/kubeadm-init-config.yaml]
```

</details>

<details>
<summary>安装附加组件</summary>

```sh
# 常见的 Kubernetes 附加组件
# CoreDNS: 提供 dns 服务, 使 Pod 可通过名称解析其他 Pod 的 IP
# kube-proxy: 服务代理, 负责将 service 请求转发到正确的 Pod
# dashboard: Kubernetes 的 Web UI, 用于管理和监控集群
# Network plugin: Calico, Flannel， 提供 Pod 之间的网络通信
# Metrics Server: 收集集群中节点的资源使用情况
kubeadm init phase addon all [--config /etc/kubeadm-init-config.yaml]

kubeadm init phase addon/coredns
kubeadm init phase addon/kube-proxy

# check
kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get pod
```

</details>

<details>
<summary>显示控制平面节点加入的命令</summary>

```sh
# Generate a new certificate key
kubeadm init phase upload-certs --upload-certs [--config /etc/kubeadm-init-config.yaml]
# Create join command
kubeadm token create --print-join-command --certificate-key "<上面命令得到的 key>"
```

</details>

<details>
<summary>显示工作节点加入的命令</summary>

```sh
kubeadm token create --print-join-command [--config /etc/kubeadm-init-config.yaml]
```

</details>

<details>
<summary>配置 kubectl</summary>

```sh
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# check
kubectl get node
```

</details>

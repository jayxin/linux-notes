<!-- vim-markdown-toc GFM -->

* [Trouble Shooting](#trouble-shooting)
    * [Fix namespace stuck in the terminating state](#fix-namespace-stuck-in-the-terminating-state)
        * [Problem](#problem)
        * [Causes](#causes)
        * [Solution](#solution)
    * [Fix pod stuck in the terminating state](#fix-pod-stuck-in-the-terminating-state)
        * [Problem](#problem-1)
        * [Solution](#solution-1)

<!-- vim-markdown-toc -->

# Trouble Shooting

## Fix namespace stuck in the terminating state

### Problem

删除 namespace 时进程卡住。

```bash
kubectl delete namespace tackle-operator
#namespace "tackle-operator" deleted

kubectl get namespace
#NAME               	   STATUS    	   AGE
#default            	                Active    	   148d
#ingress-nginx      	    Active    	   148d
#kube-node-lease    	    Active    	   148d
#kube-public        	    Active    	   148d
#kube-system        	    Active    	   148d
#kubernetes-dashboard   Active    	   148d
#tackle-operator    	   Terminating       10d
```

### Causes

<details>
<summary>获取 namespace 定义</summary>

```bash
kubectl get namespace ${NAMESPACE} -o yaml
#apiVersion: v1
#kind: Namespace
#metadata:
#  annotations:
#	kubectl.kubernetes.io/last-applied-configuration: |
#	kubernetes.io/metadata.name: tackle-operator
#spec:
#  finalizers:
#  - kubernetes
#status:
#  conditions:
# - lastTransitionTime: "2022-01-19T19:05:31Z"
#	message: 'Some content in the namespace has finalizers remaining: tackles.tackle.io/finalizer in 1 resource instances'
#	reason: SomeFinalizersRemain
#	status: "True"
#	type: NamespaceFinalizersRemaining
#  phase: Terminating
```

</details>

`finalizers` 字段指示 K8S 根据某些条件是否满足来决定是否删除一个资源。
若条件不满足则 K8S 不会删除资源。所以 `namespace` 资源会进入 `terminating`
状态，无法被删除。

### Solution

移除 `finalizers` 字段。

<details>
<summary>Solution</summary>

```bash
# 获取 namespace 的定义
kubectl get namespace ${NAMESPACE} -o json > tmp.json

# 删除 finalizers 字段
vim tmp.json

# 更新 namespace 的定义
curl -k \
  -H "Content-Type: application/json" \
  -X PUT --data-binary @tmp.json \
  http://127.0.0.1:6443/api/v1/namespaces/${NAMESPACE}/finalize

# 检查结果
kubectl get namespace ${NAMESPACE}

# 或者
(
NAMESPACE=your-stuck-namespace
kubectl proxy &
kubectl get namespace $NAMESPACE -o json |jq '.spec = {"finalizers":[]}' >tmp.json
curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json 127.0.0.1:8001/api/v1/namespaces/$NAMESPACE/finalize
)
# kubectl proxy 会在本地 127.0.0.1:8001 创建一个到集群 master 节点的代理服务器
```

</details>

## Fix pod stuck in the terminating state

### Problem

Pod 处于 Terminating 状态。

### Solution

<details>
<summary>强制删除 Pod</summary>

```bash
kubectl delete pod ${PODNAME} \
    --grace-period=0 --force --namespace ${NAMESPACE}
```

</details>

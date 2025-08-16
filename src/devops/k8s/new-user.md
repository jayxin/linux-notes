# K8S 新建用户

在默认情况下，K8S 集群中一般使用 `kubernetes-admin` 用户来管理集群.
这里新建一个用户.

<details>
<summary>K8S 新建用户</summary>

```sh
NEW_USER="tom"
CA_CERT="/etc/kubernetes/pki/ca.crt"
CA_KEY="/etc/kubernetes/pki/ca.key"

# 生成私钥
(umask 077; openssl genrsa -out $NEW_USER.key 2048)
# 创建证书签名请求
openssl req -new -key $NEW_USER.key -out $NEW_USER.csr -subj "/CN=$NEW_USER"
# 签发证书
openssl x509 -req -in $NEW_USER.csr \
    -CA $CA_CERT -CAkey $CA_KEY -CAcreateserial \
    -out $NEW_USER.crt -days 365
# 查看签发的证书
openssl x509 -in $NEW_USER.crt -text -noout

# 设置新用户的证书
kubectl config set-credentials $NEW_USER \
    --client-certificate=./$NEW_USER.crt --client-key=./$NEW_USER.key --embed-certs=true
kubectl config view

# 设置上下文
kubectl config set-context $NEW_USER@kubernetes --cluster=kubernetes --user=NEW_USER
kubectl config view

# 使用上下文 (切换到新的用户)
kubectl config use-context $NEW_USER@kubernetes
kubectl get pods --all-namespaces

# 切回默认用户
kubectl config use-context kubernetes-admin@kubernetes
kubectl get pods --all-namespaces
```

</details>

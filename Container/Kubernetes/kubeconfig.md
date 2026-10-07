minikube sshkubeconfig——Kubernetes 的“身份证 + 地图”

kubeconfig 不是“第三方专用”文件，而是所有客户端（包括 kubectl 命令行）连接集群的必需品
kubectl 本身就是一个客户端程序，和 GoLand、Python SDK 地位相同
   
没有 kubeconfig，kubectl 连集群在哪里都不知道

## kubeconfig文件结构
```yaml
apiVersion: v1
kind: Config
current-context: minikube              # 👈 当前使用的集群

clusters:                              # 👈 集群列表
- cluster:
    server: https://192.168.49.2:8443  # 👈 API Server 地址（唯一入口）
    certificate-authority: /home/xxx/.minikube/ca.crt
  name: minikube

users:                                 # 👈 用户列表（含认证凭证）
- name: minikube
  user:
    client-certificate: /home/xxx/.minikube/client.crt
    client-key: /home/xxx/.minikube/client.key

contexts:                              # 👈 上下文（把集群和用户绑定）
- context:
    cluster: minikube
    user: minikube
  name: minikube
```
一个context包括了
- Cluster	要连接的 Kubernetes 集群地址和证书	https://192.168.49.2:8443
- User	用于认证的用户凭证	证书、token 或用户名密码
- Namespace	默认使用的命名空间	default, production

查看context
```bash
howardhe@ASUS-HE:~/note$ kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default
```


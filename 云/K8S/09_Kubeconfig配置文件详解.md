[【运维干货分享】Kubeconfig配置文件详解 (qq.com)](https://mp.weixin.qq.com/s/c13i0BTMBQQddsZIV8UMPw?version=4.1.26.6018&platform=win&nwr_flag=1#wechat_redirect)

## Kubeconfig配置文件详解

- Kubeconfig 是一个YAML文件，其中包含所有Kubernetes集群详细信息，证书和秘密令牌，用于对集群进行身份验证。
- 如果使用的是托管的kubernetes集群，则可以直接从集群管理员处获取此配置文件，也可以从云平台获取此配置文件。
- 当使用时，**它会使用kubeconfig文件**中的信息连接到kubernetes集群api。
  - kubeconfig 文件的默认位置是$HOME/.kube/config
- 此外，**控制器管理器、调度器和kubelet等kubernetes集群组件使用kubeconfig文件与API服务器进行交互。**

## Kubeconfig 文件示例

- 通过kubeconfig文件中的配置信息，各组件就能连接到k8s集群。
  - certificate-authority-data：集群 CA
  - server：集群端点（主节点的 IP/DNS）
  - name：集群名称
  - user：用户/服务帐户的名称。
  - token：用户/服务帐户的秘密令牌。

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <ca-data-here>
    server: https://your-k8s-cluster.com
  name: <cluster-name>
contexts:
- context:
    cluster:  <cluster-name>
    user:  <cluster-name-user>
  name:  <cluster-name>
current-context:  <cluster-name>
kind: Config
preferences: {}
users:
- name:  <cluster-name-user>
  user:
    token: <secret-token-here>
```

## 将K8s集群与kubeconfig文件连接的不同方法

- 可以通过不同的方式使用kubeconfig，每种方式都有自己的优先级。
- 下面的各种方式的优先级的先后顺序：

1. kubectl 上下文：带有kubectl 的kubeconfig会覆盖所有其他配置，它具有最高的优先级
2. 环境变量：KUBECONFIG环境变量会覆盖当前上下文
3. 命令行参考：当前上下文的优先级最少，低于内联配置引用和env变量

### 使用kubeconfig kubectl context 连接到k8s集群

- 要连接到kubernetes集群，基本先决条件kubectl CLI插件。如果您尚未安装CLI，请按照此处给出的说明进行操作。
- 然后按照下面的步骤使用kubeconfig文件与集群进行交互

1. 将kubeconfig 移动到.kube目录

   - kubectl 使用kubeconfig文件中提供的详细信息与kubernetes集群进行交互。默认情况下，kubectl会在该位置查找配置文件 ~/.kube

   - 让我们将 kubeconfig 文件移动到 .kube 目录。替换为您的 kubeconfig 当前路径。/path/to/kubeconfig

     mv /path/to/kubeconfig ~/.kube

2. 列出所有集群上下文
   - 目录中可以有任意数量的kubeconfig。每个配置都有一个唯一的上下文名称（即集群的名称）。
   - 可以列出上下文来验证kubeconfig文件。

   ```
   kubectl config get-contexts -o=name
   ```

3. 设置当前上下文

   - 要将当前上下文设置为您的 kubeconfig 文件。您可以使用以下命令进行设置。替换为列出的上下文名称。

   ```
   kubectl config use-context
   # 例如
   kubectl config use-context my-dev-cluster
   ```

4. 验证Kubernetes集群连接

   - 要验证集群连通性，我们可以执行以下kubectl来列出集群节点

   ```
   kubectl get nodes
   ```

### 连接kubeconfig环境变量

- 可以设置环境变量与文件路径以连接到集群。在终端中使用 kubectl 命令的哪个位置，env 变量都应该可用。如果设置此变量，它将覆盖当前集群上下文。KUBECONFIGkubeconfigKUBECONFIG。

- 可以使用以下命令设置变量。文件名在哪里。dev_cluster_config kubeconfig

  ```
  export KUBECONFIG=$HOME/.kube/dev_cluster_config
  ```

### **在 kubectl 中使用 kubeconfig 文件**

- 可以使用 Kubectl 命令传递 Kubeconfig 文件，以覆盖当前上下文和 KUBECONFIG 环境变量。

下面是一个获取节点的示例。

```
kubectl get nodes --kubeconfig=$HOME/.kube/dev_cluster_config
```

- 此外，您可以使用，

```
KUBECONFIG=$HOME/.kube/dev_cluster_config kubectl get nodes
```
































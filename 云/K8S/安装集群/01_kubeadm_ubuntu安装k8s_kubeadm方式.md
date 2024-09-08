## ubuntu22.04使用kubeadm部署k8s集群

### 准备工作

- 需要在所有节点上运行

  1. 基础配置

  ```
  # 时间同步
  sudo apt -y install chrony
  sudo systemctl enable chrony && sudo systemctl start chrony
  sudo chronyc sources -v
  
  # 设置时区
  sudo timedatectl set-timezone Asia/Shanghai
  
  # 设置hosts文件
  vim /etc/hosts
  #添加如下内容：
  10.121.218.50 node01
  10.121.218.49 node02
  10.121.218.48 node03
  
  # 免密登录node01执行
  ssh-keygen
  ssh-copy-id 10.121.218.50
  ssh-copy-id 10.121.218.49
  ssh-copy-id 10.121.218.48
  
  # 禁用swap
  sudo swapoff -a && sudo sed -i '/swap/s/^/#/' /etc/fstab
  sudo swapon --show
  
  # 禁用防火墙
  sudo ufw disable
  sudo ufw status
  ```

  2. 内核参数调整

  ```
  cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
  overlay
  br_netfilter
  EOF
  
  # 加载模块
  sudo modprobe overlay
  sudo modprobe br_netfilter
  
  # 设置所需的sysctl 参数
  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
  net.bridge.bridge-nf-call-iptables  = 1
  net.bridge.bridge-nf-call-ip6tables = 1   # 将桥接的IPv4 流量传递到iptables 的链
  net.ipv4.ip_forward                 = 1   # 启用 IPv4 数据包转发
  EOF
  
  # 应用 sysctl 参数
  sudo sysctl --system
  
  # 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载
  sudo lsmod | grep br_netfilter
  sudo lsmod | grep overlay
  
  # 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1
  sudo sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
  ```

  3. 配置ipvs

  ```
  # 安装
  sudo apt install -y ipset ipvsadm
  
  # 内核加载ipvs
  cat <<EOF | sudo tee /etc/modules-load.d/ipvs.conf
  ip_vs
  ip_vs_rr
  ip_vs_wrr
  ip_vs_sh
  nf_conntrack
  EOF
  
  # 加载模块
  sudo modprobe ip_vs
  sudo modprobe ip_vs_rr
  sudo modprobe ip_vs_wrr
  sudo modprobe ip_vs_sh
  sudo modprobe nf_conntrack
  
  ```

  4. 安装容器
     - 选用containerd作为容器运行时

  ```
  # 安装containerd
  sudo apt install -y containerd
  ```

  5. 修改containerd的配置文件
     - 配置containerd使用[cgroup](https://so.csdn.net/so/search?q=cgroup&spm=1001.2101.3001.7020)的驱动为systemd，并修改沙箱镜像源：
 ```shell
# 生成containetd的配置文件
sudo mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
# 修改/etc/containerd/config.toml，修改SystemdCgroup为true
sudo sed -i "s#SystemdCgroup\ \=\ false#SystemdCgroup\ \=\ true#g" /etc/containerd/config.toml
sudo cat /etc/containerd/config.toml | grep SystemdCgroup

# 修改沙箱镜像源
sudo sed -i "s#registry.k8s.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
sudo cat /etc/containerd/config.toml | grep sandbox_image

# 配置containerd代理 可以不做
vim /etc/containerd/config.toml
# 添加如下内容：
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
            endpoint = ["http://地址:8443"]
         [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
            endpoint = ["http://地址:8443"]
# 重启containerd
systemctl restart containerd.service

 ```

> cgroup驱动说明：
>
> > cgroup驱动有两个，cgroupfs和systemd。文使用的ubuntu使用systemd作为初始化系统程序，因此将kubelet和容器运行时的cgroup驱动都配置为systemd。

6. 安装kubeadm、kubelet和kubectl

   ```shell
   # 安装依赖
   sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gpg
   
   # 添加kubernetes的key
   curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
   
   # 添加kubernetes apt仓库，使用阿里云镜像源
   echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main' | sudo tee /etc/apt/sources.list.d/kubernetes.list
   
   # 更新apt索引
   sudo apt update
   
   # 查看版本列表
   apt-cache madison kubeadm
   
   # 不带版本默认会安装最新版本，本文安装的版本为1.28.2
   sudo apt-get install -y kubelet kubeadm kubectl
   
   # 锁定版本，不随 apt upgrade 更新
   sudo apt-mark hold kubelet kubeadm kubectl
   
   # kubectl命令补全
   sudo apt install -y bash-completion
   kubectl completion bash | sudo tee /etc/profile.d/kubectl_completion.sh > /dev/null
   . /etc/profile.d/kubectl_completion.sh
   
   ```

   ###  安装k8s集群

   - master上执行

   1. 准备镜像
   
   ```shell
   # 查看镜像版本
   kubeadm config images list
   
   # 查看阿里云镜像
   kubeadm config images list --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers
   
   # 下载阿里云镜像
   # 指定版本：kubeadm config images pull --kubernetes-version=v1.28.2 --image-repository registry.aliyuncs.com/google_containers
   
   kubeadm config images pull  --image-repository registry.aliyuncs.com/google_containers
   ```
   
   - 备注：
   
     阿里云有两个镜像仓库可用
   
     - registry.aliyuncs.com/google_containers
     - registry.cn-hangzhou.aliyuncs.com/google_containers
   
   2. 初始化kubernetes集群
   
   - 初始化支持命令行和配置文件两种方式
     - 配置文件
   
   生成配置文件模块：
   
   ```
   kubeadm config print init-defaults > init.default.yaml- 
   ```
   
   - init.default.yaml文件内容如下，根据当前环境信息修改：

```shell
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.121.218.50
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: 1.28.2
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```

3. 初始化**控制台节点**

   - 初始化控制节点，配置文件方式：

   ```shell
   sudo kubeadm init --config init.default.yaml --upload-certs
   # 初始化完成后的输出
   Your Kubernetes control-plane has initialized successfully!
   
   To start using your cluster, you need to run the following as a regular user:
   
     mkdir -p $HOME/.kube
     sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
     sudo chown $(id -u):$(id -g) $HOME/.kube/config
   
   Alternatively, if you are the root user, you can run:
   
     export KUBECONFIG=/etc/kubernetes/admin.conf
   
   You should now deploy a pod network to the cluster.
   Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
     https://kubernetes.io/docs/concepts/cluster-administration/addons/
   
   Then you can join any number of worker nodes by running the following on each as root:
   # 工作节点加入
   kubeadm join 10.121.218.50:6443 --token abcdef.0123456789abcdef \
           --discovery-token-ca-cert-hash sha256:d7c612e50d63f81d15ad205eedd722a0338bfdcb6663a5799bbfb86d5fd155a4
   
   ```

- 部署成功后配置kubeconfig文件:

   ```shell
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   ```


- **命令行方式**

```shell
sudo kubeadm init \
--kubernetes-version=v1.28.2  \
--apiserver-advertise-address=10.128.118.241 \
--image-repository registry.aliyuncs.com/google_containers --v=5 \
--upload-certs \
--service-cidr=10.96.0.0/12
```

4. 安装网络插件

- 部署网络插件，选用calico

```
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/custom-resources.yaml
vim custom-resources.yaml
......
 11     ipPools:
 12     - blockSize: 26
 13       cidr: 10.244.0.0/16  # --pod-network-cidr对应的IP地址段
 14       encapsulation: VXLANCrossSubnet
......
 kubectl  create -f tigera-operator.yaml
 kubectl create -f custom-resources.yaml

```

5. 加入worker节点：

   master节点生成新的令牌
   新的worker节点的基础环境准备完成之后，接下来就需要准备加入节点的命令了，在master节点上执行kubeadm token查看加入节点的令牌信息（默认保留24小时，24小时候，令牌信息失效并且会被删除，所以也就意味着初始部署之后的令牌信息，在24小时候就失效并且消失了，所以需要重新创建）。
   
   在control-plane, master节点上查看令牌信息，如果没有则重新创建令牌，具体如下所示：
   
   
   ```
   # kubeadm token list
   # kubeadm token create --help
   # kubeadm token create --print-join-command
   kubeadm join 192.168.122.24:6443 --token hj1sax.moy397qh7e6ba298 --discovery-token-ca-cert-hash sha256:accd7731cb8fa8061f4b6cf3996d81329bab29c610110a8d75bd130c112bf3ac 
   
   ```
   
   
   
   
   
   ```
   kubeadm join 10.121.218.50:6443 --token abcdef.0123456789abcdef \
           --discovery-token-ca-cert-hash sha256:d7c612e50d63f81d15ad205eedd722a0338bfdcb6663a5799bbfb86d5fd155a4
   # 查看集群节点
   kubectl  get nodes
   NAME     STATUS   ROLES           AGE     VERSION
   node     Ready    control-plane   76m     v1.28.2
   node02   Ready    <none>          7m13s   v1.28.2
   node03   Ready    <none>          6m56s   v1.28.2
   
   ```


### 卸载Kubernetes、docker相关

```csharp
sudo apt-get purge kubelet kubeadm kubectl # 卸载
apt remove docker-ce docker-ce-cli containerd.io
```



> 参考
>
> https://blog.csdn.net/qq_42895490/article/details/139564373
>
> https://mp.weixin.qq.com/s/ZfszXdgX_r-zXoh9TE_83A
>
> https://www.znwx.cn/ops/f0imtut8.html
>
> https://blog.csdn.net/lilinhai548/article/details/141780525
>
> https://developer.aliyun.com/article/1070897

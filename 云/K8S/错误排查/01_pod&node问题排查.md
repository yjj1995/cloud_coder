| **问题类型** | **问题描述**                  | **排查步骤**                                                 |
| ------------ | ----------------------------- | ------------------------------------------------------------ |
| **Pod问题**  | Pod无法启动                   | 1. `kubectl describe pod [pod_name] -n [namespace_name]`<br>2. `kubectl logs [pod_name] -n [namespace_name]`<br>3. `kubectl get events --field-selector involvedobject.name=[pod_name] -n [namespace_name]` |
|              | Pod无法连接到其他服务         | 1. `kubectl exec -it [pod_name] -n [namespace_name] -/bin/bash`，测试网络连接<br>2. 检查Pod的NetworkPolicy<br>3. `kubectl describe service [service_name] -n [namespace_name]` |
|              | Pod运行缓慢或异常             | 1. `kubectl top pod [pod_name] -n [namespace_name]`<br>2. 进入容器使用`top`或`htop`查看进程资源使用情况<br>3. `kubectl logs [pod_name] -n [namespace_name]` |
|              | Pod无法被调度到节点上运行     | 1. `kubectl describe pod [pod_name] -n [namespace_name]`<br>2. `kubectl get nodes` 和 `kubectl describe node [node_name]`<br>3. 检查Pod所需的标签和注释是否与节点匹配 |
|              | Pod状态一直是Pending          | 1. `kubectl describe pod [pod_name]`<br>2. 查看节点资源利用率<br>3. 调整Pod的调度策略 |
|              | Pod无法访问外部服务           | 1. 检查Pod的DNS配置<br>2. 确认Pod具有网络访问权限<br>3. 检查Pod所在节点的对外访问权限<br>4. 检查网络策略 |
|              | Pod启动后立即退出             | 1. `kubectl describe pod [pod_name]`<br>2. `kubectl logs [pod_name]`<br>3. 检查容器镜像、环境变量和入口脚本 |
|              | Pod启动后无法正确运行应用程序 | 1. `kubectl logs [pod_name]`<br>2. 检查应用程序配置文件和依赖<br>3. 尝试在本地运行容器 |
|              | Service不可访问               | 1. 检查Service定义<br>2. 检查endpoint<br>3. 检查网络插件配置<br>4. 确保防火墙配置允许Service对外开放 |
| **Node问题** | Node状态异常                  | 1. `kubectl get nodes`<br>2. `kubectl describe node [node_name]`<br>3. `kubectl get pods --all-namespaces -o wide` |
|              | Node上运行的Pod无法访问网络   | 1. `kubectl describe node [node_name]`<br>2. `kubectl describe pod [pod_name] -n [namespace_name]`<br>3. `kubectl logs [pod_name] -n [namespace_name]` |
|              | Node上的Pod无法访问存储       | 1. `kubectl describe pod [pod_name] -n [namespace_name]`<br>2. 进入Pod容器尝试访问文件系统<br>3. `kubectl describe pvc [pvc_name] -n [namespace_name]` |
|              | 存储卷挂载失败                | 1. `kubectl describe pod [pod_name] -n [namespace_name]`<br>2. `kubectl describe pvc [pvc_name] -n [namespace_name]`<br>3. 检查网络存储连接和服务状态 |
|              | Node节点加入集群后无法被调度  | 1. 检查taints和tolerations是否匹配<br>2. 检查节点资源使用情况<br>3. 确保节点与API server连接正常 |
|              | PersistentVolume挂载失败      | 1. 检查PV和Pod之间的匹配关系<br>2. 检查PVC中的storageClassName是否匹配<br>3. 检查节点存储配置 |
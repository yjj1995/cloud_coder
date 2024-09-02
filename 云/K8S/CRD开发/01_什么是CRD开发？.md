## K8S可扩展性

### CRD&Controller 机制

- 自己定义业务的CRD和实现Controller 来实现扩展



#### 命令式API VS 声明式API

- 命令式API：直接向接收者发送指令
  - 简单直接
  - 多条指令会使得代码复杂
- 声明式API:   发布信息
  - 清晰明了
  - 

> 声明式API
>
> 资源                         观察者 （kube-controller manager）
>
> Node
>
> Pod
>
> Deployment 

​     CRD                        自定义Controller

> 资源：是K8S API的endpoint ,  可以通过api访问。

#### 自定义资源 CRD

- 自定义资源（Custom Resource Definition、CRD）应该是 Kubernetes 最常见的扩展方式，它是扩展 Kubernetes API 的方式之一。
  - Kubernetes 的 API 就是我们向集群提交的 YAML，系统中的各个组件会根据提交的 YAML 启动应用、创建网络路由规则以及运行工作负载。









#### Controller





#### 架构

![modular-kubernetes-api](https://img.draveness.me/modular-kubernetes-api-2021-03-24-16165170057450.png)







![Symbolic representation of seven numbered extension points for Kubernetes](https://kubernetes.ac.cn/docs/concepts/extend-kubernetes/extension-points.png)

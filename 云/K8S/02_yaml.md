# K8S中的 yaml 文件

[yaml语法学习](https://www.jianshu.com/p/36f6acedf378)

[Kubernetes（k8s） YAML文件详解 (qq.com)](https://mp.weixin.qq.com/s/Meh1EGG3HJPygiGxFfuTGg?version=4.1.26.6018&platform=win&nwr_flag=1#wechat_redirect)

**Kubernetes 支持 YAML 和 JSON格式 管理资源对象**

- **JSON** 格式：主要用于 api 接口之间消息的传递

- YAML

  格式：用于配置和管理，YAML是一种简洁的非标记性语言，内容格式人性化，较易读。

  

**YAML语法格式：**

- **大小写敏感；**

- 使用缩进表示层级关系；**不支持Tab键制表符缩进，只使用空格缩进；**

- 缩进的空格数目不重要，只要相同层级的元素左侧对齐即可，通常开头缩进两个空格；

- 字符后缩进一个空格，如冒号，逗号，短横杆（-) 等

- **`"---"` 表示YAML格式，一个文件的开始，用于分隔文件；** 可以将创建多个资源写在同一个 yaml 文件中，用 `---` 隔开，就不用写多个 yaml 文件了。

- "#” 表示注释；

  

**yaml 文件的学习方法：**

- 多看别人(官方)写的，能读懂
- 能照着现场的文件改着用
- 遇到不懂的，善用 `kubectl explain ...`命令查.



# deployment资源

## 简述

Deployment 为 Pod 和 ReplicaSet 提供了一个声明式定义 (declarative) 方法，用来替代以前的 ReplicationController 更方便的管理应用。

作为最常用的 Kubernetes 对象，Deployment 经常会用来创建 ReplicaSet 和 Pod，我们往往不会直接在集群中使用 ReplicaSet 部署一个新的微服务，一方面是因为 ReplicaSet 的功能其实不够强大，一些常见的更新、扩容和缩容运维操作都不支持，Deployment 的引入就是为了支持这些复杂的操作。



**典型用例如下：**

- 使用 Deployment 来创建 ReplicaSet。ReplicaSet 在后台创建 pod。检查启动状态，看它是成功还是失败。
- 然后，通过更新 Deployment 的 PodTemplateSpec 字段来声明 Pod 的新状态。这会创建一个新的 ReplicaSet，Deployment 会按照控制的速率将 pod 从旧的 ReplicaSet 移动到新的 ReplicaSet 中。
- 如果当前状态不稳定，回滚到之前的 Deployment revision。每次回滚都会更新 Deployment 的 revision。
- 扩容 Deployment 以满足更高的负载。
- 暂停 Deployment 来应用 PodTemplateSpec 的多个修复，然后恢复上线。
- 根据 Deployment 的状态判断上线是否 hang 住了。
- 清除旧的不必要的 ReplicaSet。



## deployment 原理

- **控制器模型**

在Kubernetes架构中，有一个叫做kube-controller-manager的组件。这个组件，是一系列控制器的集合。其中每一个控制器，都以独有的方式负责某种编排功能。而Deployment正是这些控制器中的一种。它们都遵循Kubernetes中一个通用的编排模式，即：控制循环

用一段go语言伪代码，描述这个控制循环

```go
for {
    实际状态 := 获取集群中对象X的实际状态
    期望状态 := 获取集群中对象X的期望状态
    if 实际状态 == 期望状态 {
        什么都不做
    }else{
        执行编排动作，将实际状态调整为期望状态
    }
}
```

在具体实现中，实际状态往往来自于Kubernetes集群本身。比如Kubelet通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它感兴趣的信息，这些都是常见的实际状态的来源；期望状态一般来自用户提交的YAML文件，这些信息都保存在Etcd中

对于Deployment，它的控制器简单实现如下：

1. Deployment Controller从Etcd中获取到所有携带 “app：nginx”标签的Pod，然后统计它们的数量，这就是实际状态
2. Deployment对象的replicas的值就是期望状态
3. Deployment Controller将两个状态做比较，然后根据比较结果，确定是创建Pod，还是删除已有Pod

- **滚动更新**

Deployment滚动更新的实现，依赖的是Kubernetes中的ReplicaSet

Deployment控制器实际操纵的，就是Replicas对象，而不是Pod对象。对于Deployment、ReplicaSet、Pod它们的关系如下图：

![81ae804be8abed9502d30c4b564961c6.jpg](assets/format,webp.webp)

ReplicaSet负责通过“控制器模式”，保证系统中Pod的个数永远等于指定的个数。这也正是Deployment只允许容器的restartPolicy=Always的主要原因：只有容器能保证自己始终是running状态的前提下，ReplicaSet调整Pod的个数才有意义。

Deployment同样通过控制器模式，操作ReplicaSet的个数和属性，进而实现“水平扩展/收缩”和“滚动更新”两个编排动作对于“水平扩展/收缩”的实现，Deployment Controller只需要修改replicas的值即可。用户执行这个操作的指令如下：

```shell
kubectl scale deployment nginx-deployment --replicas=4
```



## Deployment.yaml 文件解析

Deployment yaml文件包含四个部分：

- apiVersion: 表示版本。版本查看命令：kubectl api-versions
- kind: 表示资源
- metadata: 表示元信息
- spec: 资源规范字段

Deployment yaml 详解：

```yaml
apiVersion: apps/v1        # 指定api版本，此值必须在kubectl api-versions中。业务场景一般首选”apps/v1“
kind: Deployment        # 指定创建资源的角色/类型   
metadata:          # 资源的元数据/属性 
  name: demo      # 资源的名字，在同一个namespace中必须唯一
  namespace: default     # 部署在哪个namespace中。不指定时默认为default命名空间
  labels:          # 自定义资源的标签
    app: demo
    version: stable
  annotations:  # 自定义注释列表
    name: string
spec:     # 资源规范字段，定义deployment资源需要的参数属性，诸如是否在容器失败时重新启动容器的属性
  replicas: 1     # 声明副本数目
  revisionHistoryLimit: 3     # 保留历史版本
  selector:     # 标签选择器
    matchLabels:     # 匹配标签，需与上面的标签定义的app保持一致
      app: demo
      version: stable
  strategy:     # 策略
    type: RollingUpdate     # 滚动更新策略
                            # ecreate：删除所有已存在的pod,重新创建新的
                            # RollingUpdate：滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，
                                # 例如设置最大不可用pod数量，最小升级间隔时间等等
    rollingUpdate:             # 滚动更新
      maxSurge: 1             # 滚动升级时最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 0     # 在更新过程中进入不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  template:     # 定义业务模板，如果有多个副本，所有副本的属性会按照模板的相关配置进行匹配
    metadata:     # 资源的元数据/属性 
      annotations:         # 自定义注解列表
        sidecar.istio.io/inject: "false"     # 自定义注解名字
      labels:     # 自定义资源的标签
        app: demo    # 模板名称必填
        version: stable
    spec:     # 资源规范字段
      restartPolicy: Always        # Pod的重启策略。[Always | OnFailure | Nerver]
                                  # Always ：在任何情况下，只要容器不在运行状态，就自动重启容器。默认
                                  # OnFailure ：只在容器异常时才自动容器容器。
                                    # 对于包含多个容器的pod，只有它里面所有的容器都进入异常状态后，pod才会进入Failed状态
                                  # Nerver ：从来不重启容器
      nodeSelector:       # 将该Pod调度到包含这个label的node上，仅能基于简单的等值关系定义标签选择器
        caas_cluster: work-node
      containers:        # Pod中容器列表
      - name: demo         # 容器的名字   
        image: demo:v1         # 容器使用的镜像地址   
        imagePullPolicy: IfNotPresent     # 每次Pod启动拉取镜像策略
                                            # IfNotPresent ：如果本地有就不检查，如果没有就拉取。默认 
                                            # Always : 每次都检查
                                            # Never : 每次都不检查（不管本地是否有）
        command: [string]     # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
        args: [string]         # 容器的启动命令参数列表
            # 如果command和args均没有写，那么用Docker默认的配置
            # 如果command写了，但args没有写，那么Docker默认的配置会被忽略而且仅仅执行.yaml文件的command（不带任何参数的）
            # 如果command没写，但args写了，那么Docker默认配置的ENTRYPOINT的命令行会被执行，但是调用的参数是.yaml中的args
            # 如果如果command和args都写了，那么Docker默认的配置被忽略，使用.yaml的配置
        workingDir: string      # 容器的工作目录
        volumeMounts:        # 挂载到容器内部的存储卷配置
        - name: string         # 引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
          mountPath: string        # 存储卷在容器内mount的绝对路径，应少于512字符
          readOnly: boolean        # 是否为只读模式
        - name: string
          configMap:         # 类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
            name: string
            items:
            - key: string
              path: string
        ports:    # 需要暴露的端口库号列表
          - name: http     # 端口号名称
            containerPort: 8080     # 容器开放对外的端口 
          # hostPort: 8080    # 容器所在主机需要监听的端口号，默认与Container相同
            protocol: TCP     # 端口协议，支持TCP和UDP，默认TCP
        env:    # 容器运行前需设置的环境变量列表
        - name: string     # 环境变量名称
          value: string    # 环境变量的值
        resources:     # 资源管理。资源限制和请求的设置
          limits:     # 资源限制的设置，最大使用
            cpu: "1"         # CPU，"1"(1核心) = 1000m。将用于docker run --cpu-shares参数
            memory: 500Mi     # 内存，1G = 1024Mi。将用于docker run --memory参数
          requests:  # 资源请求的设置。容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行
            cpu: 100m
            memory: 100Mi
        livenessProbe:     # pod内部的容器的健康检查的设置。当探测无响应几次后将自动重启该容器
                          # 检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
          httpGet: # 通过httpget检查健康，返回200-399之间，则认为容器正常
            path: /healthCheck     # URI地址。如果没有心跳检测接口就为/
            port: 8089         # 端口
            scheme: HTTP     # 协议
            # host: 127.0.0.1     # 主机地址
        # 也可以用这两种方法进行pod内容器的健康检查
        # exec:         # 在容器内执行任意命令，并检查命令退出状态码，如果状态码为0，则探测成功，否则探测失败容器重启
        #   command:   
        #     - cat   
        #     - /tmp/health   
        # 也可以用这种方法   
        # tcpSocket: # 对Pod内容器健康检查方式设置为tcpSocket方式
        #   port: number 
          initialDelaySeconds: 30     # 容器启动完成后首次探测的时间，单位为秒
          timeoutSeconds: 5     # 对容器健康检查等待响应的超时时间，单位秒，默认1秒
          periodSeconds: 30     # 对容器监控检查的定期探测间隔时间设置，单位秒，默认10秒一次
          successThreshold: 1     # 成功门槛
          failureThreshold: 5     # 失败门槛，连接失败5次，pod杀掉，重启一个新的pod
        readinessProbe:         # Pod准备服务健康检查设置
          httpGet:
            path: /healthCheck    # 如果没有心跳检测接口就为/
            port: 8089
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 5
        lifecycle:        # 生命周期管理  
          postStart:    # 容器运行之前运行的任务  
            exec:  
              command:  
                - 'sh'  
                - 'yum upgrade -y'  
          preStop:        # 容器关闭之前运行的任务  
            exec:  
              command: ['service httpd stop']
      initContainers:        # 初始化容器
      - command:
        - sh
        - -c
        - sleep 10; mkdir /wls/logs/nacos-0
        env:
        image: {{ .Values.busyboxImage }}
        imagePullPolicy: IfNotPresent
        name: init
        volumeMounts:
        - mountPath: /wls/logs/
          name: logs
      volumes:
      - name: logs
        hostPath:
          path: {{ .Values.nfsPath }}/logs
      volumes:     # 在该pod上定义共享存储卷列表
      - name: string         # 共享存储卷名称 （volumes类型有很多种）
        emptyDir: {}         # 类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      - name: string
        hostPath:          # 类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
          path: string    # Pod所在宿主机的目录，将被用于同期中mount的目录
      - name: string
        secret:             # 类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
          scretname: string  
          items:     
          - key: string
            path: string
      imagePullSecrets:     # 镜像仓库拉取镜像时使用的密钥，以key：secretkey格式指定
      - name: harbor-certification
      hostNetwork: false    # 是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
      terminationGracePeriodSeconds: 30     # 优雅关闭时间，这个时间内优雅关闭未结束，k8s 强制 kill
      dnsPolicy: ClusterFirst    # 设置Pod的DNS的策略。默认ClusterFirst
              # 支持的策略：[Default | ClusterFirst | ClusterFirstWithHostNet | None]
              # Default : Pod继承所在宿主机的设置，也就是直接将宿主机的/etc/resolv.conf内容挂载到容器中
              # ClusterFirst : 默认的配置，所有请求会优先在集群所在域查询，如果没有才会转发到上游DNS
              # ClusterFirstWithHostNet : 和ClusterFirst一样，不过是Pod运行在hostNetwork:true的情况下强制指定的
              # None : 1.9版本引入的一个新值，这个配置忽略所有配置，以Pod的dnsConfig字段为准
      affinity:  # 亲和性调试
        nodeAffinity:     # 节点亲和性。满足特定条件的pod对象运行在同一个node上。效果同nodeSelector，但功能更强大
          requiredDuringSchedulingIgnoredDuringExecution:     # 硬亲和性
            nodeSelectorTerms:         # 节点满足任何一个条件就可以
            - matchExpressions:     # 有多个选项时，则只有同时满足这些逻辑选项的节点才能运行 pod
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
        podAffinity:     # pod亲和性。满足特定条件的pod对象运行在同一个node上
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector: 
              matchExpressions: 
              - key: app
                operator: In
                values: 
                - nginx
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:     # pod反亲和性。满足特定条件（相同pod标签）的pod对象不能运行在同一个node上
          requiredDuringSchedulingIgnoredDuringExecution: 
          - labelSelector: 
              matchExpressions: 
              - key: app
                operator: In
                values: 
                - nginx
            topologyKey: kubernetes.io/hostname        # 反亲和性的节点标签 key
      tolerations:        # 污点容忍度。允许pod可以运行在匹配的污点上
      - operator: "Equal"        # 匹配类型。支持[Exists | Equal(默认值)]。Exists为容忍所有污点
        key: "key1"
        value: "value1"
        effect: "NoSchedule"        # 污点类型：[NoSchedule | PreferNoSchedule | NoExecute]
                                        # NoSchedule ：不会被调度 
                                        # PreferNoSchedule：尽量不调度
                                        # NoExecute：驱逐节点
```



## Deployment.yaml 配置项说明

- **livenessProbe：**存活指针

用于判断 Pod（中的应用容器）是否健康，可以理解为健康检查。使用 livenessProbe 来定期的去探测，如果探测成功，则 Pod 状态可以判定为 Running；如果探测失败，可kubectl会根据Pod的重启策略来重启容器。

如果未给Pod设置 livenessProbe，则默认探针永远返回 Success。

执行 kubectl get pods 命令，输出信息中 STATUS 一列可以看到Pod是否处于Running状态。

livenessProbe使用场景：有些后端应用在出现某些异常的时候会有假死的情况，这种情况容器依然是running状态，但是应用是无法访问的，所以需要加入存活探测livenessProbe来避免这种情况的发生。

参考：https://blog.csdn.net/qq_41980563/article/details/122139923

- **readinessProbe：**就绪指针

就绪的意思是已经准备好了，Pod 的就绪可以理解为这个 Pod 可以接受请求和访问。使用 readinessProbe 来定期的去探测，如果探测成功，则 Pod 的 Ready 状态判定为 True；如果探测失败，Pod 的 Ready 状态判定为 False。

与 livenessProbe 不同的是，kubelet 不会对 readinessProbe 的探测情况有重启操作。

当执行 kubectl get pods 命令，输出信息中 READY 一列可以看到 Pod 的 READY 状态是否为 True。

readinessProbe 使用场景：k8s 应用更新虽然是滚动升级方式，但是很多后端程序启动都比较久，容器起来了，但是服务未起来，而 k8s 只要容器起来了就会移除掉旧的容器，这种情况就会导致在更新发版的时候应用访问失败。这时候就需要配置 readinessProbe 就绪检测，保证新的 pod 已经能正常使用了才会移除掉旧的 pod。

- **initContainers：**初始化容器

用于主容器启动时先启动可一个或多个初始化容器，如果有多个，那么这几个 Init Container 按照定义的顺序依次执行，只有所有的 Init Container 执行完后，主容器才会启动。由于一个 Pod 里的存储卷是共享的，所以 Init Container 里产生的数据可以被主容器使用到。

Init Container 可以在多种K8S资源里被使用到，如 Deployment、Daemon Set、Pet Set、Job 等，但归根结底都是在 Pod 启动时，在主容器启动前执行，做初始化工作。

- **tolerations：**污点容忍度

节点污点：类似节点上的标签或注解信息，用来描述对应节点的元数据信息；污点定义的格式和标签、注解的定义方式很类似，都是用一个 key-value 数据来表示，不同于节点标签，污点的键值数据中包含对应污点的 effect，污点的 effect 是用于描述对应节点上的污点有什么作用；在 k8s 上污点有三种 effect（效用）策略，第一种策略是 NoSchedule，表示拒绝 pod 调度到对应节点上运行；第二种策略是 PreferSchedule，表示尽量不把 pod 调度到此节点上运行；第三种策略是 NoExecute，表示拒绝将 pod 调度到此节点上运行；该效用相比 NoSchedule 要严苛一点；从上面的描述来看，对应污点就是来描述拒绝pod运行在对应节点的节点属性。

pod 对节点污点的容忍度：pod 要想运行在对应有污点的节点上，对应 pod 就要容忍对应节点上的污点；pod 对节点污点的容忍度就是在对应 pod 中定义怎么去匹配节点污点；通常匹配节点污点的方式有两种，一种是等值（Equal）匹配，一种是存在性（Exists）匹配；等值匹配表示对应pod的污点容忍度，必须和节点上的污点属性相等，所谓污点属性是指污点的 key、value 以及 effect；即容忍度必须满足和对应污点的key、value 和 effect 相同，这样表示等值匹配关系，其操作符为 Equal；存在性匹配是指对应容忍度只需要匹配污点的 key 和 effect 即可，value 不纳入匹配标准，即容忍度只要满足和对应污点的 key 和 effect 相同就表示对应容忍度和节点污点是存在性匹配，其操作符为 Exists；



# service资源

## 简述

Service是Kubernetes的核心概念，通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并将请求负载分发到后端各个容器应用上。



## Service.yaml 文件解析

```yaml
apiVersion: v1     # 指定api版本，此值必须在kubectl api-versions中 
kind: Service     # 指定创建资源的角色/类型 
metadata:     # 资源的元数据/属性
  name: demo     # 资源的名字，在同一个namespace中必须唯一
  namespace: default     # 部署在哪个namespace中。不指定时默认为default命名空间
  labels:         # 设定资源的标签
  - app: demo
  annotations:  # 自定义注解属性列表
  - name: string
spec:     # 资源规范字段
  type: ClusterIP     # service的类型，指定service的访问方式，默认ClusterIP。
      # ClusterIP类型：虚拟的服务ip地址，用于k8s集群内部的pod访问，在Node上kube-porxy通过设置的iptables规则进行转发
      # NodePort类型：使用宿主机端口，能够访问各个Node的外部客户端通过Node的IP和端口就能访问服务器
      # LoadBalancer类型：使用外部负载均衡器完成到服务器的负载分发，需要在spec.status.loadBalancer字段指定外部负载均衡服务器的IP，并同时定义nodePort和clusterIP用于公有云环境。
  clusterIP: string        #虚拟服务IP地址，当type=ClusterIP时，如不指定，则系统会自动进行分配，也可以手动指定。当type=loadBalancer，需要指定
  sessionAffinity: string    #是否支持session，可选值为ClietIP，默认值为空。ClientIP表示将同一个客户端(根据客户端IP地址决定)的访问请求都转发到同一个后端Pod
  ports:
    - port: 8080     # 服务监听的端口号
      targetPort: 8080     # 容器暴露的端口
      nodePort: int        # 当type=NodePort时，指定映射到物理机的端口号
      protocol: TCP     # 端口协议，支持TCP或UDP，默认TCP
      name: http     # 端口名称
  selector:     # 选择器。选择具有指定label标签的pod作为管理范围
    app: demo
status:    # 当type=LoadBalancer时，设置外部负载均衡的地址，用于公有云环境    
  loadBalancer:    # 外部负载均衡器    
    ingress:
      ip: string    # 外部负载均衡器的IP地址
      hostname: string    # 外部负载均衡器的主机名
```

注：

- port和nodePort都是service的端口，前者暴露给k8s集群内部服务访问，后者暴露给k8s集群外部流量访问。从上两个端口过来的数据都需要经过反向代理kube-proxy，流入后端pod的targetPort上，最后到达pod内的容器。NodePort类型的service可供外部集群访问是因为service监听了宿主机上的端口，即监听了（所有节点）nodePort，该端口的请求会发送给service，service再经由负载均衡转发给Endpoints的节点。



## ingress.yaml 文件详解

```yaml
apiVersion: extensions/v1beta1         # 创建该对象所使用的 Kubernetes API 的版本     
kind: Ingress         # 想要创建的对象的类别，这里为Ingress
metadata:
  name: showdoc        # 资源名字，同一个namespace中必须唯一
  namespace: op     # 定义资源所在命名空间
  annotations:         # 自定义注解
    kubernetes.io/ingress.class: nginx        # 声明使用的ingress控制器
spec:
  rules:
  - host: showdoc.example.cn     # 服务的域名
    http:
      paths:
      - path: /      # 路由路径
        backend:     # 后端Service
          serviceName: showdoc        # 对应Service的名字
          servicePort: 80           # 对应Service的端口
```
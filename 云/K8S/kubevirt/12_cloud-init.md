## cloud-init 介绍

- [Cloud-init的认识和应用-CSDN博客](https://blog.csdn.net/enweitech/article/details/106076025/)

### Cloud-init 是什么？

- Cloud-init是开源的云初始化程序，能够对新创建的弹性云服务器中指定的自定义信息（主机名，密钥和用户数据等）进行初始化配置。通过Cloud-init进行弹性云服务器的初始化配置，将对使用弹性云服务器，镜像服务和弹性伸缩产生影响。
- cloud-init是一个Linux虚拟机的初始化工具，被广泛应用在AWS和OpenStack等云平台中。
  - **用于在新建的虚拟机中进行时间设置，密码设置扩展分区，安装软件包等初始化设置**

#### 对镜像服务的影响

- 为了保证使用私有镜像新创建的弹性服务器可以自定义配置，需要在创建私有镜像前先安装Cloud-init（linux）/Cloudbase-init(windows)。

### 使用技巧

- 对于linux镜像，cloud-init负责instance的初始化工作。
- cloud-init 功能很强大，我们可以修改配置定制cloud-init
- cloud-init的配置文件为/etc/cloud/cloud.cfg,

1. 配置 root 能够直接登录Instance(默认不允许root登录)，设置：

   ```
   disable_root: 0
   ```

2. 配置ssh passwd方式登录（默认只能通知private key登录），设置：

   ```
   ssh_pwauth: 1
   ```

3. 修改instance的hostname（默认instance每次重启后cloud-init都会重新将hostname恢复成初始值），将cloud_init_modules列表中 下面两项删除或注释掉：

   ```
   ```

   


### 清除缓存

```
cloud-init clean 
```




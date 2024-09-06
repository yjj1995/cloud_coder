## **Kubevirt如何管理虚拟机？**

[Kubernetes Kubevirt 虚拟机管理 | Infvie Envoy](https://www.infvie.com/ops-notes/kubernetes-kubevirt-virtual-machine-management.html)

[KubeVirt创建虚拟机 – Mr晨 (mrlch.cn)](https://mrlch.cn/archives/1579)



[KubeVirt 03：部署一个简单的 VM – 小菜园 (imxcai.com)](https://www.imxcai.com/k8s/kubevirt/kubevirt-03-deploy-simple-vm.html)

### **虚拟机镜像制作与管理**



![img](assets/cbdc5893eda4f3c64d85a495a4c3dce7.png)

- 虚拟机镜像采用容器镜像形式存放在[镜像仓库](https://cloud.tencent.com/product/tcr?from_column=20065&from=20065)中。创建原理如上图所示，将Linux发行版本的镜像文件存放到基础镜像的/disk目录内，镜像格式支持qcow2、raw、img。

- 通过Dockerfile文件将虚拟机镜像制作成容器镜像，然后分别推送到不同的registry镜像仓库中。客户在创建虚拟机时，根据配置的优先级策略拉取registry中的虚拟机容器镜像，如果其中一台registry故障，会另一台健康的registry拉取镜像。





> 参考
>
> [KVM qcow2、raw、vmdk等镜像格式和转换](https://blog.csdn.net/weixin_41738417/article/details/102662491)


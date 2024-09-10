[QEMU安装ubuntu方式--使用云镜像-CSDN博客](https://blog.csdn.net/swhz234/article/details/141785044)

[使用qemu-system-x86_64和cloud-init修改qcow2镜像密码 - 武平宁 - 博客园 (cnblogs.com)](https://www.cnblogs.com/dewan/p/18194350)

[Cloud-init - 风继续吹 (zhang21.cn)](https://www.zhang21.cn/cloud-init/)

```
$ virt-customize --format qcow2 -a /var/lib/libvirt/images/golden-devstation-centos7-disk-10G.qcow2 \
                                --install cloud-init,mod_ssl,httpd,mariadb-server,php,openssh-server \
                                --memsize 4096  --hostname devstation  --selinux-relabel --timezone Europe/Madrid \
                                --root-password password:toor --password centos:password:developer123 \
                                --run-command 'systemctl enable httpd' --run-command 'systemctl enable mariadb' \
                                --mkdir /var/www/html/manual --upload ~/lorax/index.html:/var/www/html/manual/index.html
```


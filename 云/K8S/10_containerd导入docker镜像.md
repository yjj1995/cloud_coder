\# 拉取原始镜像
docker pull registry.aliyuncs.com/google_containers/pause:3.9

\# 重新标记为目标镜像
docker tag registry.aliyuncs.com/google_containers/pause:3.9 registry.k8s.io/pause:3.6

\# 推送到目标仓库（如果需要的话）
docker push registry.k8s.io/pause:3.9

\# 从 Docker Hub 拉取 pause:3.9 镜像
sudo docker pull registry.k8s.io/pause:3.9

\# 保存镜像到 tar 文件
sudo docker save -o pause-3.9.tar registry.k8s.io/pause:3.9

\# 使用 ctr 导入离线镜像
sudo ctr -n k8s.io images import pause-3.9.tar

\#查看镜像
sudo ctr -n k8s.io images ls

\#标记（Tag）镜像
sudo ctr images tag my-source-image:latest my-target-image:v1.0

\#删除镜像
sudo ctr images remove my-target-image:v1.0

\#查看正在运行中的镜像
ctr -n k8s.io containers list
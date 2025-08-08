### 1 安装 Minikube
**Minikube** 是一个轻量级的工具，用于在本地快速搭建单节点的 **Kubernetes 集群**，帮助开发者在本地环境中开发、测试和学习 Kubernetes。
官网链接: https://minikube.sigs.k8s.io/docs/
Mac 安装 Minikube:
```sh
curl -LO https://github.com/kubernetes/minikube/releases/download/v1.35.0/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
minikube start --driver=docker # 启动
```
卸载步骤
```sh
sudo rm /usr/local/bin/minikube
rm -rf ~/.minikube
```
![[Minikube 启动案例.png|800]]

执行 `minikube kubectl --` 安装 k8s 命令行工具
### 2 应用容器化
创建访问镜像仓库的凭证
```sh
kubectl create secret docker-registry my-secret \
  --docker-server=http://docker.io \
  --docker-username=kkq13 \
  --docker-password=yjs123456YJS \
  --docker-email=yjs215434298@gmail.com
```
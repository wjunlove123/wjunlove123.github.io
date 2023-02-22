Kind 是 Kubernetes in Docker 的简写，是一个使用 Docker 容器作为 Node 节点，在本地创建和运行Kubernetes 集群的工具。适用于在本机创建 Kubernetes 集群环境进行开发和测试。使用 Kind 搭建的集群无法在生产中使用，但是如果你只是想在本地简单的玩玩 K8s，不想占用太多的资源，那么使用 Kind 是你不错的选择。

Kind 内部也是使用 Kubeadm 创建和启动集群节点，并使用 Containerd 作为容器运行时，所以弃用 dockershim对 Kind 没有什么影响。

Kind将 Docker 容器作为 Kubernetes 的 Node 节点，并在该 Node 中安装 Kubernetes组件，包括一个或者多个 Control Plane 和一个或者多个 Work Nodes。这就解决了在本机运行多个 Node 的问题，而不需要虚拟化。

####安装
要使用 Kind 的前提是提供一个 Docker 环境，可以使用下面的命令快速安装。
```bash
sudo sh -c "$(curl -fsSL https:"(get.docker.com)"
```
当然也可以使用桌面版，Docker 安装过后接下来可以安装一个 kubectl 工具，Kind 本身不需要 kubectl，安装 kubectl 可以在本机直接管理 Kubernetes 集群

**Linux系统**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s
https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
#验证版本
kubectl version --client
```
接下来就可以安装 Kind 了，最简单的方式是在 Github Release 页面 直接下载对应的安装包即可，比如我们这里安装最新的 v0.17.0 版本
```bash
wget https://github.com/kubernetes-sigs/kind/releases/download/v0.17.0/kind-darwin-amd64
# 可以使用下面的命令加速
# wget https://ghps.cc/https://github.com/kubernetes-sigs/kind/releases/download/v0.17.0/kind-darwin-amd64
# chmod +x kind-darwin-amd64
# sudo mv kind-darwin-amd64 /usr/local/bin/kind
# 验证版本
# kind version
kind v0.17.0 go1.19.2 darwin/amd64
```

####操作
创建指定名称的集群
```bash
#kind create cluster --name kind-2
```
使用 kind get clusters 命令来获取集群列表
```bash
#kind get clusters
kind
kind-2
kind-3
```
当有多个集群的时候，我们可以使用 kubectl 来切换要管理的集群：
```bash
#切换到集群 `kind`
kubectl config use-context kind-kind
# 切换到群集`kind-2`
kubectl config use-context kind-kind-2
```
删除集群
```bash
kind delete cluster --name kind-2
```

Kubeadm 是一个安装 K8s 集群的工具包，可帮助你以更简单、合理安全和可扩展的方式引导最佳实践 Kubernetes 群
集。它还支持为你管理 Bootstrap Tokens 并升级/降级集群

Kubeadm 的目标是建立一个通过 Kubernetes 一致性测试的最小可行集群 ，但不会安装其他功能插件。在设计上并未安
装网络解决方案，所以需要用户自行安装第三方符合 CNI 的网络解决方案（如 flanal，calico 等）。此外 Kubeadm
可以在多种设备上运行，可以是 Linux 笔记本电脑、虚拟机、物理/云服务器或 Raspberry Pi，这使得 Kubeadm 非常
适合与不同种类的配置系统（例如 Terraform，Ansible 等）集成

####环境准备
3 个节点，都是 Centos 7.6 系统，内核版本：3.10.0-1160.71.1.el7.x86_64，在每个节点上添加 hosts 信息
```bash
cat /etc/hosts
172.21.0.2 master
172.21.0.3 node1
172.21.0.4 node2
```
>结点的 hostname 必须使用标准的 DNS 命名，另外千万别用默认 localhost 的 hostname，会导致各种错误出现的。在 Kubernetes 项目里，机器的名字以及一切存储在 Etcd 中的 API 对象，都必须使用标准的 DNS 命名（RFC 1123）。可以使用命令 hostnamectl set-hostname xxx 来修改 hostname  

下面是一些环境准备工作，需要在所有节点配置。
首先禁用防火墙：
```bash
systemctl stop firewalld
systemctl disable firewalld
```
禁用 SELINUX ：
```bash
setenforce 0
#使用下面命令验证是否禁用成功
cat /etc/selinux/config
SELINUX=disabled
```
由于开启内核 ipv4 转发需要加载 br_netfilter 模块，所以加载下该模块：
```bash
modprobe br_netfilter
```
最好将上面的命令设置成开机启动，因为重启后模块失效，下面是开机自动加载模块的方式，在 etc/rc.d/rc.local 文件末尾添加如下脚本内容：
```bash
for file in /etc/sysconfig/modules/*.modules ; do
[ -x $file ] && $file
done
```
然后在 /etc/sysconfig/modules/ 目录下新建如下文件：
```bash
 mkdir -p /etc/sysconfig/modules/
 vi /etc/sysconfig/modules/br_netfilter.modules
 modprobe br_netfilter
```
增加权限：
```bash
chmod 755 br_netfilter.modules
```
然后重启后，模块就可以自动加载了：
```bash
lsmod |grep br_netfilter
br_netfilter 22256 0
bridge 151336 1 br_netfilter
```
然后创建 /etc/sysctl.d/k8s.conf 文件，添加如下内容：
```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
# 下面的内核参数可以解决ipvs模式下长连接空闲超时的问题
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
net.ipv4.tcp_keepalive_time = 600
```
执行如下命令使修改生效：
```bash
sysctl -p /etc/sysctl.d/k8s.conf
```
安装 ipvs：
```bash
cat > /etc/sysconfig/modules/ipvs.modules "%EOF
#/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules &&  bash
/etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```
上面脚本创建了 **/etc/sysconfig/modules/ipvs.module**s 文件，保证在节点重启后能自动加载所需模块。使用
**lsmod | grep -e ip_vs -e nf_conntrack_ipv**4 命令查看是否已经正确加载所需的内核模块。
接下来还需要确保各个节点上已经安装了 ipset 软件包，为了便于查看 ipvs 的代理规则，最好安装一下管理工具
ipvsadm：
```bash
yum install ipset
yum install ipvsadm
```
关闭 swap 分区：
```bash
swapoff -a
```
修改 /etc/fstab 文件，注释掉 SWAP 的自动挂载，使用 free -m 确认 swap 已经关闭。swappiness 参数调整，修改 /etc/sysctl.d/k8s.conf 添加下面一行：
```bash
vm.swappiness=0
```
执行 sysctl -p /etc/sysctl.d/k8s.conf 使修改生效。

####安装 Containerd
首先需要在节点上安装 seccomp 依赖，这一步很重要：
```bash
rpm -qa |grep libseccomp
libseccomp-2.3.1-4.el7.x86_64
# 如果没有安装 libseccomp 包则可以执行下面的命令安装依赖
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/libseccomp-2.3.1-4.el7.x86_64.rpm
yum install libseccomp-2.3.1-4.el7.x86_64.rpm -y
```
由于 Containerd 需要依赖底层的 runc 工具，所以我们也需要先安装 runc，不过 Containerd 提供了一个包含相关依赖的压缩包 cri-containerd-cni-${VERSION}.${OS}-${ARCH}.tar.gz，可以直接使用这个包来进行安装，强烈建议使用该安装包，不然可能因为 runc 版本问题导致不兼容

首先从 release 页面下载最新的 1.6.10 版本的压缩包：
```bash
wget https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-1.6.10-linux-amd64.tar.gz
# 如果有限制，也可以替换成下面的 URL 加速下载
wget https://ghdl.feizhuqwq.cf/https://github.com/containerd/containerd/releases/download/v1.6.10/cri-containerd-1.6.10-linux-amd64.tar.gz
```
直接将压缩包解压到系统的各个目录中：
```bash
#tar -C / -xzf cri-containerd-1.6.10-linux-amd64.tar.gz
```
记得将 /usr/local/bin 和 /usr/local/sbin 追加到 PATH 环境变量中：
```bash
#echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin
#containerd -v
containerd github.com/containerd/containerd v1.6.10
770bd0108c32f3fb5c73ae1264f7e503fe7b2661
#runc -h
runc: symbol lookup error: runc: undefined symbol: seccomp_notify_respond
```
可以正常执行 containerd -v 命令证明 Containerd 安装成功了，但是执行 runc -h 命令的时候却出现了类似runc: undefined symbol: seccomp_notify_respond 的错误，这是因为我们当前系统默认安装的 libseccomp是 2.3.1 版本，该版本已经不能满足我们这里的 v1.6.10 版本的 Containerd 了（从 1.5.7 版本开始就不兼容了），需要 2.4 以上的版本，所以我们需要重新安装一个高版本的 libseccomp
```bash
# rpm -qa | grep libseccomp
libseccomp-2.3.1-4.el7.x86_64
# 下载高于 2.4 以上的包
# wget http://rpmfind.net/linux/centos/8-stream/BaseOS/x86_64/os/Packages/libseccomp-2.5.1-1.el8.x86_64.rpm
#rpm -ivh libseccomp-2.5.1-1.el8.x86_64.rpm
#rpm -qa | grep libseccomp
libseccomp-2.5.1-1.el8.x86_64
```
现在 runc 命令就可以正常使用了：
```bash
#runc -v
runc version 1.1.4
commit: v1.1.4-0-g5fd4c4d1
spec: 1.0.2-dev
go: go1.18.8
libseccomp: 2.5.1
```
Containerd 的默认配置文件为 /etc/containerd/config.toml，我们可以通过如下所示的命令生成一个默认的配置：
```bash
#mkdir -p /etc/containerd
#containerd config default > /etc/containerd/config.toml
```
对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为容器的 cgroup driver 可以确保节点在资源紧张的情况更加稳定，所以推荐将 containerd 的 cgroup driver 配置为 systemd。
修改前面生成的配置文件 **/etc/containerd/config.toml**，在**plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options** 配置块下面将**SystemdCgroup** 设置为 **true** ：
```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
"")
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
SystemdCgroup = true
....
```
然后再为镜像仓库配置一个加速器，需要在 cri 配置块下面的 registry 配置块下面进行配置 registry.mirrors ：
```bash
[plugins."io.containerd.grpc.v1.cri"]
"")
# sandbox_image = "registry.k8s.io/pause:3.6"
sandbox_image = "registry.aliyuncs.com/k8sxio/pause:3.8"
"")
[plugins."io.containerd.grpc.v1.cri".registry]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
endpoint = ["https:"(bqr1dr1n.mirror.aliyuncs.com"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
endpoint = ["https:"(registry.aliyuncs.com/k8sxio"]
```
由于上面我们下载的 Containerd 压缩包中包含一个 etc/systemd/system/containerd.service 的文件，这样我们就可以通过 systemd 来配置 Containerd 作为守护进程运行了，现在我们就可以启动 Containerd 了，直接执行下面的命令即可：
```bash
systemctl daemon-reload
systemctl enable containerd -- now
```
启动完成后就可以使用 Containerd 的 CLI 工具 ctr 和 crictl 了，比如查看版本：
```bash
#ctr version
Client:
 Version: v1.6.10
 Revision: 770bd0108c32f3fb5c73ae1264f7e503fe7b2661
 Go version: go1.18.8
Server:
 Version: v1.6.10
 Revision: 770bd0108c32f3fb5c73ae1264f7e503fe7b2661
 UUID: 9b89c1b6-27d0-434b-8c71-243c7af750c5
#crictl version
Version: 0.1.0
RuntimeName: containerd
RuntimeVersion: v1.6.10
RuntimeApiVersion: v1
```

####初始化集群
上面的相关环境配置完成后，接着我们就可以来安装 Kubeadm 了，我们这里是通过指定 yum 源的方式来进行安装的：
```bash
#cat  /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
```
然后安装 kubeadm、kubelet、kubectl：
```bash
# --disableexcludes 禁掉除了kubernetes之外的别的仓库
#yum makecache fast
#yum install -y kubelet-1.25.4 kubeadm-1.25.4 kubectl-1.25.4  --disableexcludes=kubernetes
#kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.4",
GitCommit:"872a965c6c6526caa949f0c6ac028ef7aff3fb78", GitTreeState:"clean",
BuildDate:"2022-11-09T13:35:06Z", GoVersion:"go1.19.3", Compiler:"gc",
Platform:"linux/amd64"}
```
可以看到我们这里安装的是 v1.25.4 版本，然后将 master 节点的 kubelet 设置成开机启动：
```bash
systemctl enable --now kubelet
```
接下来我们可以通过下面的命令在 master 节点上输出集群初始化默认使用的配置：
```bash
kubeadm config print init-defaults --component-configs KubeletConfiguration >
kubeadm.yaml
```
然后根据我们自己的需求修改配置，比如修改 imageRepository 指定集群初始化时拉取 Kubernetes 所需镜像的地址，kube-proxy 的模式为 ipvs，另外需要注意的是我们这里准备安装 flannel 网络插件的，需要将networking.podSubnet 设置为 10.244.0.0/16：
```yaml
#kubeadm.yaml
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
 advertiseAddress: 172.21.0.2
 bindPort: 6443
nodeRegistration:
 criSocket: unix:""*var/run/containerd/containerd.sock
 imagePullPolicy: IfNotPresent
 name: master
 taints:
 - effect: "NoSchedule"
 key: "node-role.kubernetes.io/master"
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
imageRepository: registry.aliyuncs.com/k8sxio
kind: ClusterConfiguration
kubernetesVersion: 1.25.4
networking:
 dnsDomain: cluster.local
 serviceSubnet: 10.96.0.0/12
 podSubnet: 10.244.0.0/16 # 指定 pod 子网
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs # kube-proxy 模式
---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
 anonymous:
 enabled: false
 webhook:
 cacheTTL: 0s
 enabled: true
 x509:
 clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
 mode: Webhook
 webhook:
 cacheAuthorizedTTL: 0s
 cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
 - 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
```
在开始初始化集群之前可以使用 **kubeadm config images pull --config kubeadm.yaml** 预先在各个服务器节点上拉取所 k8s 需要的容器镜像。
配置文件准备好过后，可以使用如下命令先将相关镜像 pull 下面：
```bash
#kubeadm config images pull "'config kubeadm.yaml
```
上面在拉取 coredns 镜像的时候出错了，没有找到这个镜像，我们可以手动 pull 该镜像，然后重新 tag 下镜像地址即可：
```bash
ctr -n k8s.io i pull docker.io/coredns/coredns:1.9.3
ctr -n k8s.io i tag docker.io/coredns/coredns:1.9.3
```
然后就可以使用上面的配置文件在 master 节点上进行初始化：
```bash
#kubeadm init --config kubeadm.yaml
```
根据安装提示拷贝 kubeconfig 文件：
```bash
 #mkdir -p $HOME/.kube
#sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
然后可以使用 kubectl 命令查看 master 节点是否已经初始化成功了：
```bash
#kubectl get nodes
NAME STATUS ROLES AGE VERSION
master NotReady control-plane 104s v1.25.4
```
现在节点还处于 NotReady 状态，是因为还没有安装 CNI 插件，我们可以先添加一个 Node 节点，再部署网络插件。

####添加节点
记住初始化集群上面的配置和操作要提前做好，将 master 节点上面的 $HOME/.kube/config 文件拷贝到 node 节点对应的文件中（如果想在 node 节点上操作 kubectl），安装 kubeadm、kubelet、kubectl（可选），然后执行上面初始化完成后提示的 join 命令即可
```bash
kubeadm join 172.21.0.2:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash
```
>如果忘记了上面的 join 命令可以使用命令 kubeadm token create --print-join-command 重新获取。

执行成功后运行 get nodes 命令：
```bash
#kubectl get nodes
NAME STATUS ROLES AGE VERSION
master NotReady control-plane 15m v1.25.4
node1 NotReady <none> 98s v1.25.4
```
这个时候其实集群还不能正常使用，因为还没有安装网络插件，接下来安装网络插件，可以在文档
https:"(kubernetes.io/docs/concepts/cluster-administration/addons 中选择我们自己的网络插件，这里我们安装**flannel**:
```yaml
#wget https://raw.githubusercontent.com/flannel-io/flannel/v0.20.1/Documentation/kube-flannel.yml
# 如果有节点是多网卡，则需要在资源清单文件中指定内网网卡
# 搜索到名为 kube-flannel-ds 的 DaemonSet，在kube-flannel容器下面

#vi kube-flannel.yml
......
containers:
- name: kube-flannel
#image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may
apply)
image: docker.io/rancher/mirrored-flannelcni-flannel:v0.20.1
command:
- /opt/bin/flanneld
args:
- --ip-masq
- --kube-subnet-mgr
- --iface=eth0 # 如果是多网卡的话，指定内网网卡的名称
......
# kubectl apply -f kube-flannel.yml # 安装 flannel 网络插件
```
用同样的方法添加另外一个节点即可。

####清理
如果你的集群安装过程中遇到了其他问题，我们可以使用下面的命令来进行重置：
```bash
 kubeadm reset
ifconfig cni0 down && ip link delete cni0
ifconfig flannel.1 down && ip link delete flannel.1
rm -rf /var/lib/cni/
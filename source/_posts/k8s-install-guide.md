---
title: 【私有云】Kubernetes（k8s）部署指南（v1.24.12）_
date: 2023-05-13 00:33:25
tags: [K8S, linux,Kubernetes,docker ]
---

阅读本篇文章时您可能需要如下技能：

1.基础的Linux技能

2.基础的docker技能

3.基础的文档日志查找能力

4.一个会用百度谷歌等搜索引擎的技能

# 一、基础环境部署

## 1）前期准备（所有节点）

### 1、修改主机名并配置hosts

修改master节点与node节点的主机名

```BASH
# 在master节点上执行
hostnamectl set-hostname k8s-master
# 在node节点上执行
hostnamectl set-hostname k8s-node
```

配置hosts

```BASH
# 请将ip修改为自己的ip
cat >> /etc/hosts<<EOF
192.168.10.100 k8s-master
192.168.10.200 k8s-node1
EOF
```

### 2、配置ssh免密访问

```BASH
# 直接一直回车就行
ssh-keygen

ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-master
ssh-copy-id -i ~/.ssh/id_rsa.pub root@k8s-node1
```

### 3、关闭防火墙

注意：如果您是在公网环境下搭建k8s建议开启防火墙并通过指定开放端口以确保服务器安全性。

```BASH
systemctl stop firewalld
systemctl disable firewalld
```

### 4、关闭swap

> 官方文档中建议将swap关闭以提高系统稳定性和系统运行性能

```BASH
# 临时关闭；关闭swap主要是为了性能考虑
swapoff -a
# 可以通过这个命令查看swap是否关闭了
free
# 永久关闭        
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 5、禁用selinux

```BASH
# 临时关闭
setenforce 0
# 永久禁用
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

### 6、允许iptables检查桥接流量（可选）

若要显式加载此模块，请运行 `sudo modprobe br_netfilter`，通过运行 `lsmod | grep br_netfilter` 来验证 br_netfilter 模块是否已加载，

```BASH
sudo modprobe br_netfilter
lsmod | grep br_netfilter
```

为了让 Linux 节点的 iptables 能够正确查看桥接流量，请确认 sysctl 配置中的 `net.bridge.bridge-nf-call-iptables` 设置为 1。 例如：

```BASH
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system
```

## 2）安装docker（所有节点）

> v1.24 之前的 Kubernetes 版本包括与 Docker Engine 的直接集成，使用名为 dockershim 的组件。 这种特殊的直接整合不再是 Kubernetes 的一部分 （这次删除被作为 v1.20 发行版本的一部分宣布）。 你可以阅读检查 Dockershim 弃用是否会影响你 以了解此删除可能会如何影响你。 要了解如何使用 dockershim 进行迁移，请参阅[从 dockershim 迁移](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/migrating-from-dockershim/)。

```BASH
# 安装yum配置工具
yum install -y yum-utils
# 配置yum源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 安装docker-ce版本
yum install -y docker-ce
# 启动
systemctl start docker
# 开机自启
systemctl enable docker


# Docker镜像源设置
# 修改文件 /etc/docker/daemon.json，没有这个文件就创建
# 添加以下内容后，重启docker服务：
cat >/etc/docker/daemon.json<<EOF
{
   "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
EOF
# 加载与重启docker
systemctl reload docker && systemctl restart docker
# 查看
systemctl status docker containerd
```

## 5）配置k8s yum 源（所有节点）

```BASH
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[k8s]
name=k8s
enabled=1
gpgcheck=0
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
EOF
```

## 6）将 sandbox_image 镜像源设置为阿里云google_containers镜像源（所有节点）

```BASH
# 导出默认配置，config.toml这个文件默认是不存在的
containerd config default > /etc/containerd/config.toml
grep sandbox_image  /etc/containerd/config.toml
sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/google_containers/pause#g"       /etc/containerd/config.toml
grep sandbox_image  /etc/containerd/config.toml
```

## 7)配置containerd cgroup 驱动程序 systemd（所有节点）

> kubernets自ｖ1.24.0后，就不再使用docker.shim，替换采用containerd作为容器运行时端点。因此需要安装containerd（在docker的基础下安装），上面安装docker的时候就自动安装了containerd了。这里的docker只是作为客户端而已。容器引擎还是containerd。

```BASH
sed -i 's#SystemdCgroup = false#SystemdCgroup = true#g' /etc/containerd/config.toml
# 应用所有更改后,重新启动containerd
systemctl restart containerd
```

## 8）开始安装kubeadm、kubelet、kubectl（master节点）

```BASH
# 不指定版本就是最新版本
yum install -y kubelet-1.24.12 kubeadm-1.24.12  kubectl-1.24.12 --disableexcludes=kubernetes
# disableexcludes=kubernetes：禁掉除了这个kubernetes之外的别的仓库
# 设置为开机自启并现在立刻启动服务 --now：立刻启动服务
systemctl enable --now kubelet

# 查看状态，这里需要等待一段时间再查看服务状态，启动会有点慢
systemctl status kubelet
```

查看版本

```BASH
kubectl version
yum info kubeadm
```

## 9)使用 kubeadm 初始化集群（master节点)

### 1、运行如下命令生成配置文件

```BASH
kubeadm config print init-defaults > kubeadm-init.yaml
```

### 2、修改 kubeadm-init.yaml

该文件有三处需要修改:

- 将`advertiseAddress: 1.2.3.4`修改为本机地址
- 将`imageRepository: k8s.gcr.io`修改为`imageRepository:registry.aliyuncs.com/google_containers`
- 将`kubernetesVersion: 1.24.0`修改为`kubernetesVersion: 1.24.12`
- 添加 `podSubnet: "10.244.0.0/16"`

修改完成的文件如下

[![初始化脚本](https://s2.loli.net/2023/03/31/omLfdOU9V6HrnuQ.png)](https://s2.loli.net/2023/03/31/omLfdOU9V6HrnuQ.png)



### 3、下载镜像

```BASH
kubeadm config images pull --config kubeadm-init.yaml
```

### 4、开始初始化

```BASH
kubeadm init --config kubeadm-init.yaml
```

### 5、初始化完成

初始化完成后会有一个提示

[![image-20230331140633606](https://s2.loli.net/2023/03/31/IYft2K9PQdRxmbn.png)](https://s2.loli.net/2023/03/31/IYft2K9PQdRxmbn.png)





将这行命令保存下来，备用

### 6、配置环境变量

```BASH
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 临时生效（退出当前窗口重连环境变量失效）
export KUBECONFIG=/etc/kubernetes/admin.conf
# 永久生效（推荐）
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile
source  ~/.bash_profile

```

## 10）配置网络插件

> 你必须部署一个基于 Pod 网络插件的 容器网络接口 (CNI)，以便你的 Pod 可以相互通信。

首先下载flannel的配置文件

```BASH
curl https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

```

然后运行

```BASH
kubectl apply -f kube-flannel.yml
```

查看节点状态

[![image-20230331174959406](https://s2.loli.net/2023/03/31/UAuMT819CYwBF7p.png)](https://s2.loli.net/2023/03/31/UAuMT819CYwBF7p.png)



这个时候节点状态就已经准备完成了

## 11）node节点加入集群

先安装kubelet

```BASH
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
# 设置为开机自启并现在立刻启动服务 --now：立刻启动服务
systemctl enable --now kubelet
systemctl status kubelet
```

如果没有令牌，可以通过在控制平面节点上运行以下命令来获取令牌：

```BASH
kubeadm token list

```

默认情况下，`令牌会在24小时后过期`。如果要在当前令牌过期后将节点加入集群， 则可以通过在控制平面节点上运行以下命令来创建新令牌：

```BASH
kubeadm token create
# 再查看
kubeadm token list

```

如果你没有 `–discovery-token-ca-cert-hash` 的值，则可以通过在控制平面节点上执行以下命令链来获取它：

```BASH
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

```

如果执行kubeadm init时没有记录下加入集群的命令，可以通过以下命令重新创建`（推荐）`一般不用上面的分别获取token和ca-cert-hash方式，执行以下命令一气呵成：

```BASH
kubeadm token create --print-join-command

```

这里需要等待一段时间，再查看节点节点状态，因为需要安装kube-proxy和flannel。

```BASH
kubectl get pods -A
kubectl get nodes

```

[![查看pod](https://s2.loli.net/2023/03/31/VSJ1scq3t96KTLP.png)](https://s2.loli.net/2023/03/31/VSJ1scq3t96KTLP.png)

查看pod



[![节点](https://s2.loli.net/2023/03/31/syjgzJLbK8ux2WG.png)](https://s2.loli.net/2023/03/31/syjgzJLbK8ux2WG.png)

节点



# 二、搭建部署Dashboard

## 1）部署

在master节点上输入

```BASH
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml

```

部署成功之后，可以看到 kubernetes-dashboard 相关的两个pod：[![dashboard pod](https://s2.loli.net/2023/03/31/UxY7Z9KfluaJt8S.png)](https://s2.loli.net/2023/03/31/UxY7Z9KfluaJt8S.png)



和 kubernetes-dashboard 相关的两个service：[![dashboard services](https://s2.loli.net/2023/03/31/PpvygGjBKNQa9fD.png)](https://s2.loli.net/2023/03/31/PpvygGjBKNQa9fD.png)



## 2）获取Token

根据[dashboard/creating-sample-user.md at master · kubernetes/dashboard (github.com)](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)来创建示例账号。

创建一个服务角色

```YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard


```

保存到admin-user.yaml。

在创建一个角色绑定

```YAML
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard

```

保存到clusterrolebinding.yaml。

然后输入

```BASH
kubectl -n kubernetes-dashboard create token admin-user

```

来获取token 复制下来。

## 3）修改为NodePort

执行:

```BASH
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard

```

修改 `type: ClusterIP` 为 `type: NodePort`:

```YAML
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP

```

看一下具体分配的 node port 是哪个：

```BASH
$ kubectl -n kubernetes-dashboard get service kubernetes-dashboard
NAME                   TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.106.3.227   <none>        443:32212/TCP   9h

```

可以看到这里分配的是 32212 端口。

然后通过nodeip:即可访问了

[![dashboard登录](https://s2.loli.net/2023/03/31/Z6VkYycI5FAhe13.png)](https://s2.loli.net/2023/03/31/Z6VkYycI5FAhe13.png)

dashboard登录



[![完事](https://s2.loli.net/2023/03/31/nbgqcdD5QAIixGu.png)](https://s2.loli.net/2023/03/31/nbgqcdD5QAIixGu.png)

完事



# 参考资料:

[1]: https://www.cnblogs.com/liugp/p/16614645.html “【云原生】Kubernetes（k8s）最新版最完整版环境部署+master高可用实现（k8sV1.24.1+dashboard+harbor）”
[2]: https://skyao.io/learning-kubernetes/docs/installation/kubeadm/dashboard.html “部署并访问Dashboard”
[3]: https://kubernetes.io/zh-cn/docs “K8S官方文档”

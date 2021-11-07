---
layout:     post
title:      "使用 kubeadm、containerd、gvisor 部署 Akash Provider"
subtitle:   ""
date:       2021-11-07
author:     "jiangydev"
header-img: "img/post-bg-docker.jpg"
header-mask: 0.5
catalog:    true
tags:
    - Akash
    - Kubernetes
---

[TOC]

> 译者注：
>
> - 本文为译文，原文地址：https://nixaid.com/deploy-akash-provider-with-kubeadm/?0.14；
> - 为提高文章的可读性，在保留原作者步骤的前提下，内容可能做了略微补充说明和调整；



这篇文章将指导您使用 kubeadm 完成 Akash Provider 部署。

>**文章更新记录**：
>
>- 2021 年 7 月 12 日：最初使用 Akash 0.12.0 发布。
>- 2021 年 10 月 30 日：针对 Akash 0.14.0 更新，多 master/worker节点设置。还通过非常简单的轮询 DNS A 记录方式，添加了 HA 支持。



## 一、介绍

这篇文章将指导您完成，在您自己的 Linux 发行版系统上运行 Akash Provider 所需的必要配置和设置步骤。（我使用的 x86_64 Ubuntu Focal）。
步骤中还包括注册和激活 Akash Provider。

我们将使用 **containerd**，因此你**无需**安装**docker**！

我没有像官方文档所建议的那样使用`kubespray`。因为我想对系统中的每个组件有更多的控制权，也不想安装 docker。

> 译者注：
>
> **Q**：containerd 和 docker 是什么？他们有什么区别？
>
> **A**：他们都是运行时组件，负责管理镜像和容器的生命周期。
>
> 作为 K8s 容器运行时，调用链的区别：
>
> - kubelet -> docker shim （在 kubelet 进程中） -> dockerd -> containerd
>
> - kubelet -> cri plugin（在 containerd 进程中） -> containerd
>
> 
>
> 更详细的描述可以看这篇文章，很好的描述了他们之间的区别：https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci

## 二、准备工作

### 2.1 设置主机名

设置一个有意义的主机名：

```shell
hostnamectl set-hostname akash-single.domainXYZ.com
```

如果您打算使用推荐的方式与 3 个 master 节点（控制平面）和 N 个 worker 节点，那么您可以像下面这样设置主机名：

> **强烈推荐**：如果您要部署多 master 节点和 worker 节点，您可以使用如下主机名。

```shell
# master 节点（控制平面）
akash-master-01.domainXYZ.com
akash-master-02.domainXYZ.com
akash-master-03.domainXYZ.com
# worker 节点
akash-worker-01.domainXYZ.com
...
akash-worker-NN.domainXYZ.com
```

> 在下面的示例中，我使用了我在 Akash Provider 上的实际地址 `*.ingress.nixaid.com`。在您的操作中，您需要将其替换为您的域名。

### 2.2 启用 netfilter 和内核 IP 转发（路由）

> 注：kube-proxy 需要启用 `net.bridge.bridge-nf-call-iptables`；

```shell
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```

### 2.3 禁用 swap 分区

建议禁用并删除交换文件。

```shell
swapon -s
swapoff -a
sed -i '/swap/d' /etc/fstab
rm /swapfile
```

> 译者注：
>
> **Q**：什么是 swap 分区？为什么要禁用 swap 分区？
>
> **A**：swap 分区是将一部分磁盘空间用作内存使用，可以暂时解决内存不足的问题，Linux 系统默认会开启 swap 分区。
>
> 个人观点：swap 分区虽然能解决内存暂时不足的问题，但是与磁盘交互 IO 会影响应用程序的性能和稳定性，也不是长久之计。若考虑服务质量，服务提供商应该禁用 swap 分区。客户在内存资源不够时，可以临时申请更大的内存。
>
> 目前 K8s 版本是不支持 swap 的，经过漫长的讨论，最终 K8s 社区确实打算支持 swap，但还是实验版。
>
> K8s 社区对开启 swap 功能的讨论：https://github.com/kubernetes/kubernetes/issues/53533

### 2.4 安装 containerd

```shell
wget https://github.com/containerd/containerd/releases/download/v1.5.7/containerd-1.5.7-linux-amd64.tar.gz
tar xvf containerd-1.5.7-linux-amd64.tar.gz -C /usr/local/

wget -O /etc/systemd/system/containerd.service https://raw.githubusercontent.com/containerd/containerd/v1.5.7/containerd.service

mkdir /etc/containerd

systemctl daemon-reload
systemctl start containerd
systemctl enable containerd
```

### 2.5 安装 CNI 插件

> 容器网络接口 (CNI) ：大多数 pod 网络都需要。

```shell
cd
mkdir -p /etc/cni/net.d /opt/cni/bin
CNI_ARCH=amd64
CNI_VERSION=1.0.1
CNI_ARCHIVE=cni-plugins-linux-${CNI_ARCH}-v${CNI_VERSION}.tgz
wget https://github.com/containernetworking/plugins/releases/download/v${CNI_VERSION}/${CNI_ARCHIVE}
tar -xzf $CNI_ARCHIVE -C /opt/cni/bin
```

> 译者注：
>
> CNI 作为容器网络的统一标准，可以让各个容器管理平台（k8s、mesos等）都可以通过相同的接口调用各式各样的网络插件（flannel,calico,weave 等）来为容器配置网络。

### 2.6 安装 crictl

> Kubelet 容器运行时接口 (CRI) ：kubeadm、kubelet 需要的。

```shell
INSTALL_DIR=/usr/local/bin
mkdir -p $INSTALL_DIR
CRICTL_VERSION="v1.22.0"
CRICTL_ARCHIVE="crictl-${CRICTL_VERSION}-linux-amd64.tar.gz"
wget "https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VERSION}/${CRICTL_ARCHIVE}"
tar -xzf $CRICTL_ARCHIVE -C $INSTALL_DIR
chown -Rh root:root $INSTALL_DIR
```

更新`/etc/crictl.yaml`文件内容：

```shell
cat > /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
#debug: true
pull-image-on-create: true
disable-pull-on-run: false
EOF
```



### 2.7 安装 runc

> runc 是非 Akash 部署使用的默认 OCF 运行时（即标准的 kubernetes 容器，如 kube、etcd、calico pods）。

```shell
apt install -y runc
```

> 译者注：
>
> - runc：是一个根据OCI标准创建并运行容器的 CLI 工具。简单地说，容器的创建、运行、销毁等操作最终都是通过调用 runc 完成的。
>
> - OFC：开放式计算设施。

### 2.8 （仅在 worker 节点上操作）安装 gVisor (runsc) 和 runc

>gVisor (runsc) ：是容器的应用程序内核，可在任何地方提供有效的纵深防御。
>这里对比了几个容器运行时（Kata、gVisor、runc、rkt 等）：https://gist.github.com/miguelmota/8082507590d55c400c5dc520a43e14a1

```shell
apt -y install software-properties-common
curl -fsSL https://gvisor.dev/archive.key | apt-key add -
add-apt-repository "deb [arch=amd64,arm64] https://storage.googleapis.com/gvisor/releases release main"
apt update
apt install -y runsc
```

### 2.9 配置 containerd 以使用 gVisor

> 现在 Kubernetes 将使用 containerd（稍后当我们使用 kubeadm 引导它时，您将看到这一点），我们需要将其配置为使用 gVisor 运行时。
>
> 删除 NoSchedule master 节点上的“runsc”（最后两行）。   

更新`/etc/containerd/config.toml`：

```shell
cat > /etc/containerd/config.toml << 'EOF'
# version MUST present, otherwise containerd won't pick the runsc !
version = 2

#disabled_plugins = ["cri"]

[plugins."io.containerd.runtime.v1.linux"]
  shim_debug = true
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF
```

并重启 containerd 服务：

```shell
systemctl restart containerd
```

>gVisor (runsc) 还不能与 systemd-cgroup 或 cgroup v2 一起工作，如果您想跟进，这有两个未解决的问题：
>[systemd-cgroup 支持 #193](https://github.com/google/gvisor/issues/193)
>[在 runc 中支持 cgroup v2 #3481](https://github.com/google/gvisor/issues/3481)

## 三、安装 Kubernetes

> 译者注：
>
> **Q**：什么是 Kubernetes？
>
> **A**：[Kubernetes](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/)，也称为 K8s，是一个自动部署、扩展和管理容器化应用的开源系统。

### 3.1 安装最新稳定版 kubeadm、kubelet、kubectl 并添加 kubelet systemd 服务

```shell
INSTALL_DIR=/usr/local/bin
RELEASE="$(curl -sSL https://dl.k8s.io/release/stable.txt)"
cd $INSTALL_DIR
curl -L --remote-name-all https://storage.googleapis.com/kubernetes-release/release/${RELEASE}/bin/linux/amd64/{kubeadm,kubelet,kubectl}
chmod +x {kubeadm,kubelet,kubectl}

RELEASE_VERSION="v0.9.0"
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service" | sed "s:/usr/bin:${INSTALL_DIR}:g" | tee /etc/systemd/system/kubelet.service
mkdir -p /etc/systemd/system/kubelet.service.d
curl -sSL "https://raw.githubusercontent.com/kubernetes/release/${RELEASE_VERSION}/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf" | sed "s:/usr/bin:${INSTALL_DIR}:g" | tee /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

cd
systemctl enable kubelet
```

### 3.2 使用 kubeadm 部署 Kubernetes 集群

>您可以根据需要随意调整 pod 子网和 service 子网以及其他控制平面配置。
>更多内容请参阅：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/
>
>确保将`kubernetesVersion`环境变量设置为您下载的二进制文件的版本：https://dl.k8s.io/release/stable.txt
>
>你只需要在 1 个 master 节点上运行 `kubeadm init` 命令！
>稍后您将使用`kubeadm join`加入其他 master 节点（控制平面）和 worker 节点。
>
>如果你打算扩展你的 master 节点，取消多 master 部署的`controlPlaneEndpoint`注释。
>把`controlPlaneEndpoint`设置为和`--cluster-public-hostname`相同的值。该主机名应解析为 Kubernetes 集群的公共 IP。
>
>**专业提示**：您可以多次注册相同的 DNS A 记录，指向多个 Akash master 节点。然后设置`controlPlaneEndpoint`为该 DNS A 记录，这样它就能做到均衡的 DNS 轮询！

```shell
cat > kubeadm-config.yaml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: cgroupfs
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock # --cri-socket=unix:///run/containerd/containerd.sock
  ##kubeletExtraArgs:
    ##root-dir: /mnt/data/kubelet
  imagePullPolicy: "Always"
localAPIEndpoint:
  advertiseAddress: "0.0.0.0"
  bindPort: 6443
---
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: "stable"
#controlPlaneEndpoint: "akash-master-lb.domainXYZ.com:6443"
networking:
  podSubnet: "10.233.64.0/18" # --pod-network-cidr, taken from kubespray
  serviceSubnet: "10.233.0.0/18" # --service-cidr, taken from kubespray
EOF
```

在您的 master 节点（控制平面）上下载必要的依赖包，执行预检并预拉镜像：

```shell
apt -y install ethtool socat conntrack
kubeadm init phase preflight --config kubeadm-config.yaml
kubeadm config images pull --config kubeadm-config.yaml
```

现在，如果您准备好初始化单节点配置（您只有一个 master 节点也将运行 Pod）：

```shell
kubeadm init --config kubeadm-config.yaml
```

如果您计划运行多 master 部署，请确保添加`--upload-certs`到 kubeadm init 命令，如下所示：

```shell
kubeadm init --config kubeadm-config.yaml --upload-certs
```

>您可以先运行 `kubeadm init phase upload-certs --upload-certs --config kubeadm-config.yaml`，然后运行`kubeadm token create --config kubeadm-config.yaml --print-join-command`来获取`kubeadm join`命令！
>
>单节点 master 部署不需要运行`upload-certs`命令。  

如果您将看到“**Your Kubernetes control-plane has initialized successfully!** ”，则一切顺利，现在您的 Kubernetes 控制平面节点就在服务中了！

您还将看到 kubeadm 输出了带有`--token`的`kubeadm join`命令。请妥善保管这条命令，因为如果您想根据您需要的架构类型来加入更多节点（worker 节点、数据节点），那么需要该命令。

通过多 master 节点部署，您将看到带有`--control-plane --certificate-key`额外参数的`kubeadm join`命令！确保在将更多 master 节点加入集群时使用它们！

### 3.3 检查您的节点

> 当你要使用`kubectl`命令与 Kubernetes 集群通信时，您可以设置`KUBECONFIG`变量，也可以创建指向`~/.kube/config`的软链接。
>确保您的`/etc/kubernetes/admin.conf` 是**安全**的，因为它是您的 Kuberentes 管理密钥，可让您对 K8s 集群执行所有操作。
>
>Akash Provider 服务将使用该配置，稍后您将看到如何使用。
>
>（多 master 部署）新加入的 master 节点将自动从源 master 节点接收`admin.conf`文件。为您提供更多备份！

```shell
mkdir ~/.kube
ln -sv /etc/kubernetes/admin.conf ~/.kube/config
kubectl get nodes -o wide
```

### 3.4 安装 Calico 网络

缺少网络插件，Kubernetes 是无法使用的：

```shell
$ kubectl describe node akash-single.domainXYZ.com | grep -w Ready
  Ready            False   Wed, 28 Jul 2021 09:47:09 +0000   Wed, 28 Jul 2021 09:46:52 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

```shell
cd
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

> 译者注：
>
> **Q**：什么是 Calico？
>
> **A**：`Calico`是一套开源的网络和网络安全方案，用于容器、虚拟机、宿主机之间的网络连接，可以用在kubernetes、OpenShift、DockerEE、OpenStrack等PaaS或IaaS平台上。

### 3.5 （可选）允许 master 节点调度 POD

默认情况下，出于**安全原因**，您的 K8s 集群不会在控制平面节点（master 节点）上调度 Pods。要么删除 master 节点上的 Taints（污点），以便您可以使用`kubectl taint nodes`命令在其上调度 pod，或者用`kubectl join`命令加入将运行 calico 的 worker 节点（但首先要确保执行了这些准备步骤：安装 CNI 插件、安装 crictl、配置 Kubernetes 以使用 gVisor）。

如果您正在运行单 master 部署，请删除 master 节点上的你 Taints：

```shell
$ kubectl describe node akash-single.domainXYZ.com |grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule

$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

> 译者注：
>
> - Taint 是设置在节点上的，可以用来避免 pod 被分配到这些节点上，除非 pod 指定了 Toleration（容忍），并和 Taint 匹配。
>
> - 示例1（新增）：`kubectl taint nodes nodeName key1=value1:NoSchedule`。
> - 示例2（删除）：`kubectl taint nodes nodeName key1-`。
> - `NoSchedule`：一定不能被调度。

### 3.6 检查您的节点和 pods

```shell
$ kubectl get nodes -o wide --show-labels
NAME          STATUS   ROLES                  AGE     VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME    LABELS
akash-single.domainXYZ.com   Ready    control-plane,master   4m24s   v1.22.1   149.202.82.160   <none>        Ubuntu 20.04.2 LTS   5.4.0-80-generic   containerd://1.4.8   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=akash-single.domainXYZ.com,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=

$ kubectl describe node akash-single.domainXYZ.com | grep -w Ready
  Ready                True    Wed, 28 Jul 2021 09:51:09 +0000   Wed, 28 Jul 2021 09:51:09 +0000   KubeletReady                 kubelet is posting ready status. AppArmor enabled
```

```shell
$ kubectl get pods -A 
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-78d6f96c7b-kkszw   1/1     Running   0          3m33s
kube-system   calico-node-ghgz8                          1/1     Running   0          3m33s
kube-system   coredns-558bd4d5db-2shqz                   1/1     Running   0          4m7s
kube-system   coredns-558bd4d5db-t9r75                   1/1     Running   0          4m7s
kube-system   etcd-akash-single.domainXYZ.com                           1/1     Running   0          4m26s
kube-system   kube-apiserver-akash-single.domainXYZ.com                 1/1     Running   0          4m24s
kube-system   kube-controller-manager-akash-single.domainXYZ.com        1/1     Running   0          4m23s
kube-system   kube-proxy-72ntn                           1/1     Running   0          4m7s
kube-system   kube-scheduler-akash-single.domainXYZ.com                 1/1     Running   0          4m21s 
```

### 3.7 （可选）安装 NodeLocal DNSCache

>如果您使用带有此补丁的 akash 版本，则不必安装 NodeLocal DNSCache。
>
>补丁：https://github.com/arno01/akash/commit/5c81676bb8ad9780571ff8e4f41e54565eea31fd
>
>PR：https://github.com/ovrclk/akash/pull/1440
>问题：[https://github.com/ovrclk/akash/issues/1339#issuecomment-889293170](https://github.com/ovrclk/akash/issues/1339#issuecomment-889293170)
>
>使用 NodeLocal DNSCache 以获得更好的性能，[https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) 

NodeLocal DNSCache 服务安装很简单：

```shell
kubedns=$(kubectl get svc kube-dns -n kube-system -o 'jsonpath={.spec.clusterIP}')
domain="cluster.local"
localdns="169.254.25.10"

wget https://raw.githubusercontent.com/kubernetes/kubernetes/v1.22.1/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml

sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/__PILLAR__DNS__SERVER__/$kubedns/g" nodelocaldns.yaml

kubectl create -f nodelocaldns.yaml
```

修改`/var/lib/kubelet/config.yaml`文件中的`clusterDNS`，使用`169.254.25.10`(NodeLocal DNSCache) 而不是默认值`10.233.0.10`并重新启动 Kubelet 服务：

```shell
systemctl restart kubelet
```

为了确保您使用的是 NodeLocal DNSCache，您可以创建一个 POD 并检查内部域名服务器应该是`169.254.25.10`：

```shell
$ cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 169.254.25.10
options ndots:5
```

### 3.8 （可选）IPVS 模式

>**注意**：由于“akash-deployment-restrictions”网络策略中这行代码[https://github.com/ovrclk/akash/blob/7c39ea403/provider/cluster/kube/builder.go#L599](https://github.com/ovrclk/akash/blob/7c39ea403/provider/cluster/kube/builder.go#L599)，导致跨服务通信（容器 X 到同一 POD 内的服务 Y）在 IPVS 模式下不起作用。不过，可能还有另一种方法可以使其工作，您可以尝试启用`kube_proxy_mode`切换的`kubespray`部署，看看它是否能够以这种方式工作。
>
>https://www.linkedin.com/pulse/iptables-vs-ipvs-kubernetes-vivek-grover/
>[https://forum.akash.network/t/akash-provider-support-ipvs-kube-proxy-mode/720](https://forum.akash.network/t/akash-provider-support-ipvs-kube-proxy-mode/720)  

如果你有一天想在 IPVS 模式下运行 kube-proxy（而不是默认的 IPTABLES 模式），你需要重复上面“安装 NodeLocal DNSCache”部分的步骤，除了`nodelocaldns.yaml`使用以下命令修改文件：

```shell
sed -i "s/__PILLAR__LOCAL__DNS__/$localdns/g; s/__PILLAR__DNS__DOMAIN__/$domain/g; s/,__PILLAR__DNS__SERVER__//g; s/__PILLAR__CLUSTER__DNS__/$kubedns/g" nodelocaldns.yaml
```

通过设置`mode:`为`ipvs`，将 kube-proxy 切换到 IPVS 模式：

```shell
kubectl edit configmap kube-proxy -n kube-system
```

并重启 kube-proxy：

```shell
kubectl -n kube-system delete pod -l k8s-app=kube-proxy
```

> 译者注：
>
> kube-proxy 有三种代理模式：userspace、iptables、ipvs。
>
> IPVS（IP Virtual Server）是构建于Netfilter之上，作为Linux内核的一部分提供传输层负载均衡的模块。

### 3.9 配置 Kubernetes 以使用 gVisor

设置 gvisor (runsc) Kubernetes RuntimeClass。
Akash Provider 创建的部署将默认使用它。

```shell
cat <<'EOF' | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
EOF
```

### 3.10 检查您的 gVisor 和 K8s DNS 是否如期工作

```shell
cat > dnstest.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  runtimeClassName: gvisor
  containers:
  - name: dnsutils
    image: gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
    command:
      - sleep
      - "3600"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
EOF

$ kubectl apply -f dnstest.yaml

$ kubectl exec -i -t dnsutils -- sh
/ # dmesg 
[   0.000000] Starting gVisor...
[   0.459332] Reticulating splines...
[   0.868906] Synthesizing system calls...
[   1.330219] Adversarially training Redcode AI...
[   1.465972] Waiting for children...
[   1.887919] Generating random numbers by fair dice roll...
[   2.302806] Accelerating teletypewriter to 9600 baud...
[   2.729885] Checking naughty and nice process list...
[   2.999002] Granting licence to kill(2)...
[   3.116179] Checking naughty and nice process list...
[   3.451080] Creating process schedule...
[   3.658232] Ready!
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope global dynamic 
    inet6 ::1/128 scope global dynamic 
2: eth0: <UP,LOWER_UP> mtu 1480 
    link/ether 9e:f1:a0:ee:8a:55 brd ff:ff:ff:ff:ff:ff
    inet 10.233.85.133/32 scope global dynamic 
    inet6 fe80::9cf1:a0ff:feee:8a55/64 scope global dynamic 
/ # ip r
127.0.0.0/8 dev lo 
::1 dev lo 
169.254.1.1 dev eth0 
fe80::/64 dev eth0 
default via 169.254.1.1 dev eth0 
/ # netstat -nr
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
169.254.1.1     0.0.0.0         255.255.255.255 U         0 0          0 eth0
0.0.0.0         169.254.1.1     0.0.0.0         UG        0 0          0 eth0
/ # cat /etc/resolv.conf 
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.233.0.10
options ndots:5
/ # ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=42 time=5.671 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 5.671/5.671/5.671 ms
/ # ping google.com
PING google.com (172.217.13.174): 56 data bytes
64 bytes from 172.217.13.174: seq=0 ttl=42 time=85.075 ms
^C
--- google.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 85.075/85.075/85.075 ms
/ # nslookup kubernetes.default.svc.cluster.local
Server:		10.233.0.10
Address:	10.233.0.10#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.233.0.1

/ # exit

$ kubectl delete -f dnstest.yaml
```

如果您看到“Starting gVisor...”，则表示 Kubernetes 能够使用 gVisor (runsc) 运行容器。

> *如果您使用 NodeLocal DNSCache，您将看到 169.254.25.10 而不是 10.233.0.10 的域名服务器。*

> 当你应用 network-policy-default-ns-deny.yaml 后，网络测试将不起作用（即 ping 8.8.8.8 将失败），这是意料之中的。

### 3.11 （可选）加密 etcd

>etcd 是一个一致性且高可用的键值存储，用作 Kubernetes 对所有集群数据的后备存储。
>Kubernetes 使用 etcd 来存储它的所有数据——它的配置数据、状态和元数据。Kubernetes 是一个分布式系统，所以它需要一个像 etcd 这样的分布式数据存储。etcd 允许 Kubernetes 集群中的任何节点读写数据。

⚠️与没有加密相比，在 EncryptionConfig 中存储原始加密密钥只能**适度**改善您的安全状况。请使用[kms provider](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/)以提高安全性。

```shell
$ mkdir /etc/kubernetes/encrypt

ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
cat > /etc/kubernetes/encrypt/config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

按以下方式更新您的`/etc/kubernetes/manifests/kube-apiserver.yaml`，以便 kube-apiserver 知道从哪里读取密钥：

```shell
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
$ diff -Nur kube-apiserver.yaml.orig /etc/kubernetes/manifests/kube-apiserver.yaml
--- kube-apiserver.yaml.orig	2021-07-28 10:05:38.198391788 +0000
+++ /etc/kubernetes/manifests/kube-apiserver.yaml	2021-07-28 10:13:51.975308872 +0000
@@ -41,6 +41,7 @@
     - --service-cluster-ip-range=10.233.0.0/18
     - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
     - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
+    - --encryption-provider-config=/etc/kubernetes/encrypt/config.yaml
     image: k8s.gcr.io/kube-apiserver:v1.22.1
     imagePullPolicy: IfNotPresent
     livenessProbe:
@@ -95,6 +96,9 @@
     - mountPath: /usr/share/ca-certificates
       name: usr-share-ca-certificates
       readOnly: true
+    - mountPath: /etc/kubernetes/encrypt
+      name: k8s-encrypt
+      readOnly: true
   hostNetwork: true
   priorityClassName: system-node-critical
   volumes:
@@ -122,4 +126,8 @@
       path: /usr/share/ca-certificates
       type: DirectoryOrCreate
     name: usr-share-ca-certificates
+  - hostPath:
+      path: /etc/kubernetes/encrypt
+      type: DirectoryOrCreate
+    name: k8s-encrypt
 status: {}
```

当你保存这个文件时`/etc/kubernetes/manifests/kube-apiserver.yaml`，`kube-apiserver`会*自动*重启。（这可能需要一两分钟，请耐心等待。）

```shell
$ crictl ps | grep apiserver
10e6f4b409a4b       106ff58d43082       36 seconds ago       Running             kube-apiserver            0                   754932bb659c5
```

不要忘记在所有 Kubernetes 节点上执行相同的操作！

使用您刚刚添加的密钥加密所有的信息：

```shell
kubectl get secrets -A -o json | kubectl replace -f -
```

### 3.12 （可选）IPv6 支持

如果您希望在 Kubernetes 集群中启用 IPv6 支持，请参阅这个地址：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/

## 四、为 Akash Provider 服务配置 Kubernetes

> 如果您要将 Akash 提供程序从 0.12 更新到 0.14，请确保按照以下步骤：[https://github.com/ovrclk/akash/blob/9e1a7aa5ccc894e89d84d38485b458627a287bae/script/provider_migrate_to_hostname_operator.md](https://github.com/ovrclk/akash/blob/9e1a7aa5ccc894e89d84d38485b458627a287bae/script/provider_migrate_to_hostname_operator.md)

```shell
mkdir akash-provider
cd akash-provider

wget https://raw.githubusercontent.com/ovrclk/akash/mainnet/main/pkg/apis/akash.network/v1/crd.yaml
kubectl apply -f ./crd.yaml

wget https://raw.githubusercontent.com/ovrclk/akash/mainnet/main/pkg/apis/akash.network/v1/provider_hosts_crd.yaml
kubectl apply -f ./provider_hosts_crd.yaml

wget https://raw.githubusercontent.com/ovrclk/akash/mainnet/main/_docs/kustomize/networking/network-policy-default-ns-deny.yaml
kubectl apply -f ./network-policy-default-ns-deny.yaml

wget https://raw.githubusercontent.com/ovrclk/akash/mainnet/main/_run/ingress-nginx-class.yaml
kubectl apply -f ./ingress-nginx-class.yaml

wget https://raw.githubusercontent.com/ovrclk/akash/mainnet/main/_run/ingress-nginx.yaml
kubectl apply -f ./ingress-nginx.yaml

# NOTE: in this example the Kubernetes node is called "akash-single.domainXYZ.com" and it's going to be the ingress node too.
# In the perfect environment that would not be the master (control-plane) node, but rather the worker nodes!

kubectl label nodes akash-single.domainXYZ.com akash.network/role=ingress

# Check the label got applied:

kubectl get nodes -o wide --show-labels 
NAME          STATUS   ROLES                  AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME    LABELS
akash-single.domainXYZ.com   Ready    control-plane,master   10m   v1.22.1   149.202.82.160   <none>        Ubuntu 20.04.2 LTS   5.4.0-80-generic   containerd://1.4.8   akash.network/role=ingress,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=akash-single.domainXYZ.com,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node-role.kubernetes.io/master=,node.kubernetes.io/exclude-from-external-load-balancers=
```

```shell
git clone --depth 1 -b mainnet/main https://github.com/ovrclk/akash.git
cd akash
kubectl apply -f _docs/kustomize/networking/namespace.yaml
kubectl kustomize _docs/kustomize/akash-services/ | kubectl apply -f -

cat >> _docs/kustomize/akash-hostname-operator/kustomization.yaml <<'EOF'
images:
  - name: ghcr.io/ovrclk/akash:stable
    newName: ghcr.io/ovrclk/akash
    newTag: 0.14.0
EOF

kubectl kustomize _docs/kustomize/akash-hostname-operator | kubectl apply -f -
```

### 4.1 获取 DNS 泛解析

在我的案例中，我将使用`<anything>.ingress.nixaid.com`，它将解析为我的 Kubernetes 节点的 IP 的位置。最好只有 worker 节点！

```shell
A *.ingress.nixaid.com resolves to 149.202.82.160
```

并且`akash-provider.nixaid.com`将解析到我将要运行的 Akash Provider 服务的 IP。（Akash Provider 服务正在监听`8443/tcp`端口）

**专业提示**：您可以多次注册相同的 DNS 泛 A 记录，指向多个 Akash 工作节点，这样它将获得均衡的 DNS 轮询！

## 五、在 Akash 区块链上创建 Akash Provider

现在我们已经配置、启动并运行了我们的 Kubernetes，是时候让 Akash Provider 运行了。

注意：您不必直接在 Kubernetes 集群上运行 Akash Provider 服务。你可以在任何地方运行它。它只需要能够通过互联网访问您的 Kubernetes 集群。

### 5.1 创建 Akash 用户

我们将在`akash`用户下运行`akash provider`。

```shell
useradd akash -m -U -s /usr/sbin/nologin
mkdir /home/akash/.kube
cp /etc/kubernetes/admin.conf /home/akash/.kube/config
chown -Rh akash:akash /home/akash/.kube
```

### 5.2 安装 Akash 客户端

```shel
su -s /bin/bash - akash

wget https://github.com/ovrclk/akash/releases/download/v0.14.0/akash_0.14.0_linux_amd64.zip
unzip akash_0.14.0_linux_amd64.zip
mv /home/akash/akash_0.14.0_linux_amd64/akash /usr/bin/
chown root:root /usr/bin/akash
```

### 5.3 配置 Akash 客户端

```shell
su -s /bin/bash - akash

mkdir ~/.akash

export KUBECONFIG=/home/akash/.kube/config
export PROVIDER_ADDRESS=akash-provider.nixaid.com
export AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/mainnet"
export AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | grep -Ev 'forbole|skynetvalidators|162.55.94.246' | shuf -n 1)"
export AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
export AKASH_KEYRING_BACKEND=file
export AKASH_PROVIDER_KEY=default
export AKASH_FROM=$AKASH_PROVIDER_KEY
```

检查变量：

```shell
$ set |grep ^AKASH
AKASH_CHAIN_ID=akashnet-2
AKASH_FROM=default
AKASH_KEYRING_BACKEND=file
AKASH_NET=https://raw.githubusercontent.com/ovrclk/net/master/mainnet
AKASH_NODE=http://135.181.181.120:28957
AKASH_PROVIDER_KEY=default
```

现在创建`default`密钥：

```shell
$ akash keys add $AKASH_PROVIDER_KEY --keyring-backend=$AKASH_KEYRING_BACKEND

Enter keyring passphrase:
Re-enter keyring passphrase:

- name: default
  type: local
  address: akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
...
```

确保将助记词种子保存在安全的地方，因为这是恢复帐户和资金的唯一方法！

> 如果您想从助记词种子中恢复您的密钥，请在`akash keys add ...`命令后添加`--recover`参数。

### 5.4 配置 Akash provider

```shell
$ cat provider.yaml 
host: https://akash-provider.nixaid.com:8443
attributes:
  - key: region
    value: europe  ## change this to your region!
  - key: host
    value: akash   ## feel free to change this to whatever you like
  - key: organization  # optional
    value: whatever-your-Org-is  ## change this to your org.
  - key: tier          # optional
    value: community
```

### 5.5 为您的 Akash 提供商的钱包提供资金

您将需要大约`10 AKT`(Akash Token) 才能开始使用。

您的钱包必须有足够的资金，因为在区块链上竞标订单需要 5 AKT 存款。该押金在中标/中标后全额退还。

在此处提到的交易所之一购买 AKT：https://akash.network/token/

查询钱包余额：

```shell
# Put here your address which you've got when created one with "akash keys add" command.
export AKASH_ACCOUNT_ADDRESS=akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0

$ akash \
  --node "$AKASH_NODE" \
  query bank balances "$AKASH_ACCOUNT_ADDRESS"
```

> *换算：1 akt = 1000000 uakt (akt\*10^6)*

### 5.6 在 Akash 网络上注册您的 provider

```shell
$ akash tx provider create provider.yaml \
  --from $AKASH_PROVIDER_KEY \
  --keyring-backend=$AKASH_KEYRING_BACKEND \
  --node=$AKASH_NODE \
  --chain-id=$AKASH_CHAIN_ID \
  --gas-prices="0.025uakt" \
  --gas="auto" \
  --gas-adjustment=1.15
```

> 如果你想更改`provider.yaml`，请在更新后再执行带有如上参数的`akash tx provider update`命令。

在 Akash 网络上注册您的 provider 后，就能看到你的主机：

```shell
$ akash \
  --node "$AKASH_NODE" \
  query provider list -o json | jq -r '.providers[] | [ .attributes[].value, .host_uri, .owner ] | @csv' | sort -d
"australia-east-akash-provider","https://provider.akashprovider.com","akash1ykxzzu332txz8zsfew7z77wgsdyde75wgugntn"
"equinix-metal-ams1","akash","mn2-0","https://provider.ams1p0.mainnet.akashian.io:8443","akash1ccktptfkvdc67msasmesuy5m7gpc76z75kukpz"
"equinix-metal-ewr1","akash","mn2-0","https://provider.ewr1p0.mainnet.akashian.io:8443","akash1f6gmtjpx4r8qda9nxjwq26fp5mcjyqmaq5m6j7"
"equinix-metal-sjc1","akash","mn2-0","https://provider.sjc1p0.mainnet.akashian.io:8443","akash10cl5rm0cqnpj45knzakpa4cnvn5amzwp4lhcal"
"equinix-metal-sjc1","akash","mn2-0","https://provider.sjc1p1.mainnet.akashian.io:8443","akash1cvpefa7pw8qy0u4euv497r66mvgyrg30zv0wu0"
"europe","nixaid","https://akash-provider.nixaid.com:8443","akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0"
"us-west-demo-akhil","dehazelabs","https://73.157.111.139:8443","akash1rt2qk45a75tjxzathkuuz6sq90jthekehnz45z"
"us-west-demo-caleb","https://provider.akashian.io","akash1rdyul52yc42vd8vhguc0t9ryug9ftf2zut8jxa"
"us-west-demo-daniel","https://daniel1q84.iptime.org","akash14jpkk4n5daspcjdzsrylgw38lj9xug2nznqnu2"
"us-west","https://ssymbiotik.ekipi365.com","akash1j862g3efcw5xcvn0402uwygrwlzfg5r02w9jw5"
```

### 5.7 创建 provider 证书

您必须向区块链发起一个交易，来创建和您的 provider 关联的证书：

```shell
akash tx cert create server $PROVIDER_ADDRESS \
  --chain-id $AKASH_CHAIN_ID \
  --keyring-backend $AKASH_KEYRING_BACKEND \
  --from $AKASH_PROVIDER_KEY \
  --node=$AKASH_NODE \
  --gas-prices="0.025uakt" --gas="auto" --gas-adjustment=1.15
```

## 六、启动 Akash Provider

Akash 提供程序将需要 Kubernetes 管理员配置。之前我们已经把它移到了`/home/akash/.kube/config`。

创建`start-provider.sh`文件来启动 Akash Provider。
但在此之前，请使用您在创建 provider 密钥时设置的密码来创建`key-pass.txt`文件。

```shell
echo "Your-passWoRd" | tee /home/akash/key-pass.txt
```

> 确保将`--cluster-public-hostname`设置为解析到 Kubernetes 集群公共 IP 的主机名。正如您将看到的那样，您还要将设置`controlPlaneEndpoint`为该主机名。

```shell
cat > /home/akash/start-provider.sh << 'EOF'
#!/usr/bin/env bash

export AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/mainnet"
export AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | grep -Ev 'forbole|skynetvalidators|162.55.94.246' | shuf -n 1)"

cd /home/akash
( sleep 2s; cat key-pass.txt; cat key-pass.txt ) | \
  /home/akash/bin/akash provider run \
  --chain-id akashnet-2 \
  --node $AKASH_NODE \
  --keyring-backend=file \
  --from default \
  --fees 1000uakt \
  --kubeconfig /home/akash/.kube/config \
  --cluster-k8s true \
  --deployment-ingress-domain ingress.nixaid.com \
  --deployment-ingress-static-hosts true \
  --bid-price-strategy scale \
  --bid-price-cpu-scale 0.0011 \
  --bid-price-memory-scale 0.0002 \
  --bid-price-storage-scale 0.0011 \
  --bid-price-endpoint-scale 0 \
  --bid-deposit 5000000uakt \
  --balance-check-period 24h \
  --minimum-balance 5000000 \
  --cluster-node-port-quantity 1000 \
  --cluster-public-hostname akash-master-lb.domainXYZ.com \
  --bid-timeout 10m0s \
  --log_level warn
```

确保它是可执行的：

```shell
chmod +x /home/akash/start-provider.sh
```

创建`akash-provider.service`systemd 服务，以便 Akash provider 能自启动：

```shell
cat > /etc/systemd/system/akash-provider.service << 'EOF'
[Unit]
Description=Akash Provider
After=network.target

[Service]
User=akash
Group=akash
ExecStart=/home/akash/start-provider.sh
KillSignal=SIGINT
Restart=on-failure
RestartSec=15
StartLimitInterval=200
StartLimitBurst=10
#LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

启动 Akash provider：

```shell
systemctl daemon-reload
systemctl start akash-provider
systemctl enable akash-provider
```

检查日志：

```shell
journalctl -u akash-provider --since '5 min ago' -f
```

Akash 节点检测如下：

```shell
D[2021-06-29|11:33:34.190] node resources                               module=provider-cluster cmp=service cmp=inventory-service node-id=akash-single.domainXYZ.com available-cpu="units:<val:\"7050\" > attributes:<key:\"arch\" value:\"amd64\" > " available-memory="quantity:<val:\"32896909312\" > " available-storage="quantity:<val:\"47409223602\" > "
```

**CPU**：`7050 / 1000`= **7 CPU**（服务器实际上有8个CPU，它必须为提供者节点运行的任何东西保留1个CPU，这是一件聪明的事情）
**可用内存**：32896909312 /（1024^3）= **30.63Gi**（服务器有 32Gi RAM)
**可用存储空间**：47409223602 / (1024^3) = **44.15Gi**（这里**有点奇怪**，我在 rootfs 根目录上只有 32Gi 可用）

## 七、在我们自己的 Akash provider 上部署

> 为了在您的客户端配置 Akash 客户端，请参考https://nixaid.com/solana-on-akashnet/中的前 4 个步骤或[https://docs.akash.network/guides/deploy](https://docs.akash.network/guides/deploy)

现在我们已经运行了自己的 Akash Provider，让我们尝试在其上部署一些东西。
我将部署`echoserver`服务，一旦通过 HTTP/HTTPS 端口查询，该服务就可以向客户端返回有趣的信息。

```shell
$ cat echoserver.yml
---
version: "2.0"

services:
  echoserver:
    image: gcr.io/google_containers/echoserver:1.10
    expose:
      - port: 8080
        as: 80
        to:
          - global: true
        #accept:
        #  - my.host123.com

profiles:
  compute:
    echoserver:
      resources:
        cpu:
          units: 0.1
        memory:
          size: 128Mi
        storage:
          size: 128Mi
  placement:
    akash:
      #attributes:
      #  host: nixaid
      #signedBy:
      #  anyOf:
      #    - "akash1365yvmc4s7awdyj3n2sav7xfx76adc6dnmlx63" ## AKASH
      pricing:
        echoserver: 
          denom: uakt
          amount: 100

deployment:
  echoserver:
    akash:
      profile: echoserver
      count: 1
```

请注意，我已经注释了`signedBy`指令，这个指令用来确保他们部署在受信任的 provider 上。注释掉之后，意味着您可以部署在您想要的任何 Akash provider 上，而不必签名。

如果您想使用`signedBy`指令，您可以使用`akash tx audit attr create`命令在您的 Akash Provider 上签署属性。

```shell
akash tx deployment create echoserver.yml \
  --from default \
  --node $AKASH_NODE \
  --chain-id $AKASH_CHAIN_ID \
  --gas-prices="0.025uakt" --gas="auto" --gas-adjustment=1.15
```

现在已经将部署发布到了 Akash 网络，让我们看看我们的 Akash Provider。

从 Akash provider 的角度来看，成功的预订是这样的：

> `Reservation fulfilled` 信息是我们要找的。

```shell
Jun 30 00:00:46 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:46.122] syncing sequence                             cmp=client/broadcaster local=31 remote=31
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.837] order detected                               module=bidengine-service order=order/akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.867] group fetched                                module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.867] requesting reservation                       module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: D[2021-06-30|00:00:53.868] reservation requested                        module=provider-cluster cmp=service cmp=inventory-service order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1 resources="group_id:<owner:\"akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h\" dseq:1585829 gseq:1 > state:open group_spec:<name:\"akash\" requirements:<signed_by:<> > resources:<resources:<cpu:<units:<val:\"100\" > > memory:<quantity:<val:\"134217728\" > > storage:<quantity:<val:\"134217728\" > > endpoints:<> > count:1 price:<denom:\"uakt\" amount:\"2000\" > > > created_at:1585832 "
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.868] Reservation fulfilled                        module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: D[2021-06-30|00:00:53.868] submitting fulfillment                       module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1 price=357uakt
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.932] broadcast response                           cmp=client/broadcaster response="Response:\n  TxHash: BDE0FE6CD12DB3B137482A0E93D4099D7C9F6A5ABAC597E17F6E94706B84CC9A\n  Raw Log: []\n  Logs: []" err=null
Jun 30 00:00:53 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:53.932] bid complete                                 module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:00:56 akash1 start-provider.sh[1029866]: I[2021-06-30|00:00:56.121] syncing sequence                             cmp=client/broadcaster local=32 remote=31
```

现在 Akash provider 的预订已完成，我们可以将其视为客户端的出价：

```shell
$ akash query market bid list   --owner=$AKASH_ACCOUNT_ADDRESS   --node $AKASH_NODE   --dseq $AKASH_DSEQ   
...
- bid:
    bid_id:
      dseq: "1585829"
      gseq: 1
      oseq: 1
      owner: akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h
      provider: akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
    created_at: "1585836"
    price:
      amount: "357"
      denom: uakt
    state: open
  escrow_account:
    balance:
      amount: "50000000"
      denom: uakt
    id:
      scope: bid
      xid: akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
    owner: akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
    settled_at: "1585836"
    state: open
    transferred:
      amount: "0"
      denom: uakt
...
```

现在让我们创建租约（接受 Akash Provider 提供的出价）：

```shell
akash tx market lease create \
  --chain-id $AKASH_CHAIN_ID \
  --node $AKASH_NODE \
  --owner $AKASH_ACCOUNT_ADDRESS \
  --dseq $AKASH_DSEQ \
  --gseq $AKASH_GSEQ \
  --oseq $AKASH_OSEQ \
  --provider $AKASH_PROVIDER \
  --from default \
  --gas-prices="0.025uakt" --gas="auto" --gas-adjustment=1.15
```

现在我们可以在提供商的站点上看到“ **lease won** ”：

```shell
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: D[2021-06-30|00:03:42.479] ignoring group                               module=bidengine-order order=akash15yd3qszmqausvzpj7n0y0e4pft2cu9rt5gccda/1346631/1/1 group=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: I[2021-06-30|00:03:42.479] lease won                                    module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1 lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: I[2021-06-30|00:03:42.480] shutting down                                module=bidengine-order order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: I[2021-06-30|00:03:42.480] lease won                                    module=provider-manifest lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: I[2021-06-30|00:03:42.480] new lease                                    module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: D[2021-06-30|00:03:42.480] emit received events skipped                 module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 data=<nil> leases=1 manifests=0
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: I[2021-06-30|00:03:42.520] data received                                module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 version=77fd690d5e5ec8c320a902da09a59b48dc9abd0259d84f9789fee371941320e7
Jun 30 00:03:42 akash1 start-provider.sh[1029866]: D[2021-06-30|00:03:42.520] emit received events skipped                 module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 data="deployment:<deployment_id:<owner:\"akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h\" dseq:1585829 > state:active version:\"w\\375i\\r^^\\310\\303 \\251\\002\\332\\t\\245\\233H\\334\\232\\275\\002Y\\330O\\227\\211\\376\\343q\\224\\023 \\347\" created_at:1585832 > groups:<group_id:<owner:\"akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h\" dseq:1585829 gseq:1 > state:open group_spec:<name:\"akash\" requirements:<signed_by:<> > resources:<resources:<cpu:<units:<val:\"100\" > > memory:<quantity:<val:\"134217728\" > > storage:<quantity:<val:\"134217728\" > > endpoints:<> > count:1 price:<denom:\"uakt\" amount:\"2000\" > > > created_at:1585832 > escrow_account:<id:<scope:\"deployment\" xid:\"akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829\" > owner:\"akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h\" state:open balance:<denom:\"uakt\" amount:\"5000000\" > transferred:<denom:\"uakt\" amount:\"0\" > settled_at:1585859 > " leases=1 manifests=0
```

发送清单以最终在您的 Akash Provider 上部署`echoserver`服务！

```shell
akash provider send-manifest echoserver.yml \
  --node $AKASH_NODE \
  --dseq $AKASH_DSEQ \
  --provider $AKASH_PROVIDER \
  --from default
```

Provider 已经获得了部署的清单 => “ **manifest received** ”，并且`kube-builder`模块在命名空间`c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76`下已经“ **created service** ” ：

```shell
Jun 30 00:06:16 akash1 start-provider.sh[1029866]: I[2021-06-30|00:06:16.122] syncing sequence                             cmp=client/broadcaster local=32 remote=32
Jun 30 00:06:21 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:21.413] inventory fetched                            module=provider-cluster cmp=service cmp=inventory-service nodes=1
Jun 30 00:06:21 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:21.413] node resources                               module=provider-cluster cmp=service cmp=inventory-service node-id=akash-single.domainXYZ.com available-cpu="units:<val:\"7050\" > attributes:<key:\"arch\" value:\"amd64\" > " available-memory="quantity:<val:\"32896909312\" > " available-storage="quantity:<val:\"47409223602\" > "
Jun 30 00:06:26 akash1 start-provider.sh[1029866]: I[2021-06-30|00:06:26.122] syncing sequence                             cmp=client/broadcaster local=32 remote=32
Jun 30 00:06:35 akash1 start-provider.sh[1029866]: I[2021-06-30|00:06:35.852] manifest received                            module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829
Jun 30 00:06:35 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:35.852] requests valid                               module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 num-requests=1
Jun 30 00:06:35 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:35.853] publishing manifest received                 module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 num-leases=1
Jun 30 00:06:35 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:35.853] publishing manifest received for lease       module=manifest-manager deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829 lease_id=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
Jun 30 00:06:35 akash1 start-provider.sh[1029866]: I[2021-06-30|00:06:35.853] manifest received                            module=provider-cluster cmp=service
Jun 30 00:06:36 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:36.023] provider/cluster/kube/builder: created service module=kube-builder service="&Service{ObjectMeta:{echoserver      0 0001-01-01 00:00:00 +0000 UTC <nil> <nil> map[akash.network:true akash.network/manifest-service:echoserver akash.network/namespace:c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76] map[] [] []  []},Spec:ServiceSpec{Ports:[]ServicePort{ServicePort{Name:0-80,Protocol:TCP,Port:80,TargetPort:{0 8080 },NodePort:0,AppProtocol:nil,},},Selector:map[string]string{akash.network: true,akash.network/manifest-service: echoserver,akash.network/namespace: c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76,},ClusterIP:,Type:ClusterIP,ExternalIPs:[],SessionAffinity:,LoadBalancerIP:,LoadBalancerSourceRanges:[],ExternalName:,ExternalTrafficPolicy:,HealthCheckNodePort:0,PublishNotReadyAddresses:false,SessionAffinityConfig:nil,IPFamily:nil,TopologyKeys:[],},Status:ServiceStatus{LoadBalancer:LoadBalancerStatus{Ingress:[]LoadBalancerIngress{},},},}"
Jun 30 00:06:36 akash1 start-provider.sh[1029866]: I[2021-06-30|00:06:36.121] syncing sequence                             cmp=client/broadcaster local=32 remote=32
Jun 30 00:06:36 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:36.157] provider/cluster/kube/builder: created rules module=kube-builder rules="[{Host:623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com IngressRuleValue:{HTTP:&HTTPIngressRuleValue{Paths:[]HTTPIngressPath{HTTPIngressPath{Path:/,Backend:IngressBackend{Resource:nil,Service:&IngressServiceBackend{Name:echoserver,Port:ServiceBackendPort{Name:,Number:80,},},},PathType:*Prefix,},},}}}]"
Jun 30 00:06:36 akash1 start-provider.sh[1029866]: D[2021-06-30|00:06:36.222] deploy complete                              module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
```

让我们从客户端看租约状态：

```shelll
akash provider lease-status \
  --node $AKASH_NODE \
  --dseq $AKASH_DSEQ \
  --provider $AKASH_PROVIDER \
  --from default

{
  "services": {
    "echoserver": {
      "name": "echoserver",
      "available": 1,
      "total": 1,
      "uris": [
        "623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com"
      ],
      "observed_generation": 1,
      "replicas": 1,
      "updated_replicas": 1,
      "ready_replicas": 1,
      "available_replicas": 1
    }
  },
  "forwarded_ports": {}
}
```

我们搞定了！
我们查询一下：

```shell
$ curl 623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com


Hostname: echoserver-5c6f84887-6kh9p

Pod Information:
  -no pod information available-

Server values:
  server_version=nginx: 1.13.3 - lua: 10008

Request Information:
  client_address=10.233.85.136
  method=GET
  real path=/
  query=
  request_version=1.1
  request_scheme=http
  request_uri=http://623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com:8080/

Request Headers:
  accept=*/*
  host=623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com
  user-agent=curl/7.68.0
  x-forwarded-for=CLIENT_IP_REDACTED
  x-forwarded-host=623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com
  x-forwarded-port=80
  x-forwarded-proto=http
  x-real-ip=CLIENT_IP_REDACTED
  x-request-id=8cdbcd7d0c4f42440669f7396e206cae
  x-scheme=http

Request Body:
  -no body in request-
```

我们在自己的 Akash provider 上的部署正如期工作！

让我们从 Kubernetes 的角度看看，我们在 Akash Provider 的部署实际上是怎样的？

```shell
$ kubectl get all -A -l akash.network=true
NAMESPACE                                       NAME                      READY   STATUS    RESTARTS   AGE
c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   pod/echoserver-5c6f84887-6kh9p   1/1     Running   0          2m37s

NAMESPACE                                       NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   service/echoserver   ClusterIP   10.233.47.15   <none>        80/TCP    2m37s

NAMESPACE                                       NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   deployment.apps/echoserver   1/1     1            1           2m38s

NAMESPACE                                       NAME                            DESIRED   CURRENT   READY   AGE
c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   replicaset.apps/echoserver-5c6f84887   1         1         1       2m37s

$ kubectl get ing -A 
NAMESPACE                                       NAME   CLASS    HOSTS                                                  ADDRESS     PORTS   AGE
c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   echoserver    <none>   623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com   localhost   80      8m47s

$ kubectl -n c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76 describe ing echoserver
Name:             echoserver
Namespace:        c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76
Address:          localhost
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                                                  Path  Backends
  ----                                                  ----  --------
  623n1u4k2hbiv6f1kuiscparqk.ingress.nixaid.com  
                                                        /   echoserver:80 (10.233.85.137:8080)
Annotations:                                            <none>
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    8m9s (x2 over 9m5s)  nginx-ingress-controller  Scheduled for sync


$ crictl pods
POD ID              CREATED             STATE               NAME                                        NAMESPACE                                       ATTEMPT             RUNTIME
4c22dba05a2c0       5 minutes ago       Ready               echoserver-5c6f84887-6kh9p                         c9mdnf8o961odir96rdcflt9id95rq2a2qesidpjuqd76   0                   runsc
...
```

客户端也可以读取他的部署日志：

```shell
akash \
  --node "$AKASH_NODE" \
  provider lease-logs \
  --dseq "$AKASH_DSEQ" \
  --gseq "$AKASH_GSEQ" \
  --oseq "$AKASH_OSEQ" \
  --provider "$AKASH_PROVIDER" \
  --from default \
  --follow

[echoserver-5c6f84887-6kh9p] Generating self-signed cert
[echoserver-5c6f84887-6kh9p] Generating a 2048 bit RSA private key
[echoserver-5c6f84887-6kh9p] ..............................+++
[echoserver-5c6f84887-6kh9p] ...............................................................................................................................................+++
[echoserver-5c6f84887-6kh9p] writing new private key to '/certs/privateKey.key'
[echoserver-5c6f84887-6kh9p] -----
[echoserver-5c6f84887-6kh9p] Starting nginx
[echoserver-5c6f84887-6kh9p] 10.233.85.136 - - [30/Jun/2021:00:08:00 +0000] "GET / HTTP/1.1" 200 744 "-" "curl/7.68.0"
[echoserver-5c6f84887-6kh9p] 10.233.85.136 - - [30/Jun/2021:00:27:10 +0000] "GET / HTTP/1.1" 200 744 "-" "curl/7.68.0"
```

完成测试后，是时候关闭部署了：

```shell
akash tx deployment close \
  --node $AKASH_NODE \
  --chain-id $AKASH_CHAIN_ID \
  --dseq $AKASH_DSEQ \
  --owner $AKASH_ACCOUNT_ADDRESS \
  --from default \
  --gas-prices="0.025uakt" --gas="auto" --gas-adjustment=1.15
```

Provider 将其视为预期的“**deployment closed**”、“**teardown request**”等：

```shell
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.828] deployment closed                            module=provider-manifest deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.828] manager done                                 module=provider-manifest deployment=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.829] teardown request                             module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.830] shutting down                                module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash cmp=deployment-monitor
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.830] shutdown complete                            module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash cmp=deployment-monitor
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.837] teardown complete                            module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.837] waiting on dm.wg                             module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.838] waiting on withdrawal                        module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.838] shutting down                                module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash cmp=deployment-withdrawal
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.838] shutdown complete                            module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash cmp=deployment-withdrawal
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.838] shutdown complete                            module=provider-cluster cmp=service cmp=deployment-manager lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0 manifest-group=akash
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.838] manager done                                 module=provider-cluster cmp=service lease=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1/akash1nxq8gmsw2vlz3m68qvyvcf3kh6q269ajvqw6y0
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: D[2021-06-30|00:28:44.838] unreserving capacity                         module=provider-cluster cmp=service cmp=inventory-service order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.838] attempting to removing reservation           module=provider-cluster cmp=service cmp=inventory-service order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.838] removing reservation                         module=provider-cluster cmp=service cmp=inventory-service order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:28:44 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:44.838] unreserve capacity complete                  module=provider-cluster cmp=service cmp=inventory-service order=akash1h24fljt7p0nh82cq0za0uhsct3sfwsfu9w3c9h/1585829/1/1
Jun 30 00:28:46 akash1 start-provider.sh[1029866]: I[2021-06-30|00:28:46.122] syncing sequence                             cmp=client/broadcaster local=36 remote=36
```

## 八、关闭集群

以防万一，如果您想关闭 Kubernetes 集群：

```shell
systemctl disable akash-provider
systemctl stop akash-provider

kubectl drain <node name> --delete-local-data --force --ignore-daemonsets

###kubectl delete node <node name>
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t nat -X && iptables -t mangle -F && iptables -t mangle -X  && iptables -t raw -F && iptables -t raw -X && iptables -X
ip6tables -F && ip6tables -t nat -F && ip6tables -t nat -X && ip6tables -t mangle -F && ip6tables -t mangle -X && ip6tables -t raw -F && ip6tables -t raw -X && ip6tables -X
ipvsadm -C
conntrack -F

## if Weave Net was used:
weave reset (if you  used)  (( or "ip link delete weave" ))

## if Calico was used:
ip link
ip link delete cali*
ip link delete vxlan.calico

modprobe -r ipip
```

一些排错场景：

```shell
## if getting during "crictl rmp -a" (deleting all pods using crictl)
removing the pod sandbox "f89d5f4987fbf80790e82eab1f5634480af814afdc82db8bca92dc5ed4b57120": rpc error: code = Unknown desc = sandbox network namespace "/var/run/netns/cni-65fbbdd0-8af6-8c2a-0698-6ef8155ca441" is not fully closed

ip netns ls
ip -all netns delete

ps -ef|grep -E 'runc|runsc|shim'
ip r
pidof runsc-sandbox |xargs -r kill
pidof /usr/bin/containerd-shim-runc-v2 |xargs -r kill -9
find /run/containerd/io.containerd.runtime.v2.task/ -ls

rm -rf /etc/cni/net.d

systemctl restart containerd
###systemctl restart docker
```

## 九、水平扩展您的 Akash provider

如果您想为新部署添加更多空间，您可以扩展您的 Akash Provider。

为此，获取新的裸机或 VPS 主机并重复所有步骤，直到“**3.2 使用 kubeadm 部署 Kubernetes 集群**”（不包括）。
在新的 master 节点（控制平面）或 worker 节点上运行以下命令：

```shell
apt update
apt -y dist-upgrade
apt autoremove

apt -y install ethtool socat conntrack

mkdir -p /etc/kubernetes/manifests

## If you are using NodeLocal DNSCache
sed -i -s 's/10.233.0.10/169.254.25.10/g' /var/lib/kubelet/config.yaml
```

在您现有的 master 节点（控制平面）上生成 token。您将需要它来加入新的 master 节点/ worker 节点。

如果添加新的 master 节点，请确保运行以下`upload-certs`阶段：

> 这是为了避免从您的 master 节点手动复制`/etc/kubernetes/pki`到新的 master 节点。

```shell
kubeadm init phase upload-certs --upload-certs --config kubeadm-config.yaml
```

生成用于将新 master 节点或 worker 节点加入 kubernetes 集群的 Token：

```shell
kubeadm token create --config kubeadm-config.yaml --print-join-command
```

要加入任意数量的 master 节点（控制平面），请运行以下命令：

```shell
kubeadm join akash-master-lb.domainXYZ.com:6443 --token REDACTED.REDACTED --discovery-token-ca-cert-hash sha256:REDACTED --control-plane --certificate-key REDACTED
```

要加入任意数量的 worker 节点，请运行以下命令：

```shell
kubeadm join akash-master-lb.domainXYZ.com:6443 --token REDACTED.REDACTED --discovery-token-ca-cert-hash sha256:REDACTED
```

### 9.1 扩展 Ingress

现在您有 1 个以上的 worker 节点，您可以扩展`ingress-nginx-controller`以提高服务可用性。
为此，您只需要运行以下命令。

用`akash.network/role=ingress`给所有 worker 节点打标：

```shell
kubectl label nodes akash-worker-<##>.nixaid.com akash.network/role=ingress
```

将`ingress-nginx-controller`扩展到您拥有的 woker 节点数量：

```shell
kubectl -n ingress-nginx scale deployment ingress-nginx-controller --replicas=<number of worker nodes>
```

现在，给 worker 节点 IP 的注册新的 DNS A 泛域名，解析到运行着`ingress-nginx-controller`的`*.ingress.nixaid.com`。

示例：

```shell
$ dig +noall +answer anything.ingress.nixaid.com
anything.ingress.nixaid.com. 1707 IN  A 167.86.73.47
anything.ingress.nixaid.com. 1707 IN  A 185.211.5.95
```

## 十、已知问题和解决方案

- Akash Provider 的收益在损失：https://github.com/ovrclk/akash/issues/1363
- 当 tag 相同时，provider 不会拉新的镜像：https://github.com/ovrclk/akash/issues/1354
- 当部署清单出现错误时，容器挂起（僵尸程序）：https://github.com/ovrclk/akash/issues/1353
- [netpol] akash-deployment-restrictions 阻止 POD 在 pod 子网通过 53/udp、53/tcp 访问 kube-dns ：https://github.com/ovrclk/akash/issues/1339

## 十一、参考

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
[https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
[https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet -cgroup-driver](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet-cgroup-driver)
https://storage.googleapis.com/kubernetes-release/release/stable.txt
https://gvisor.dev/docs/user_guide/containerd/quick_start/
[https://github.com/containernetworking/cni #how-do-i-use-cni](https://github.com/containernetworking/cni#how-do-i-use-cni)
https://docs.projectcalico.org/getting-started/kubernetes/quickstart
https://kubernetes.io/docs/concepts/overview/components/
https://matthewpalmer.net/kubernetes-app-developer/articles/how-does-kubernetes-use-etcd.html
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
https://docs.akash.network/operator/provider
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#tear-down
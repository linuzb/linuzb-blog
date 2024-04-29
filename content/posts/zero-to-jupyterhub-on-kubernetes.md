---
title: "从零开始在Kubernetes上搭建Jupyterhub"
date: 2024-04-29T15:08:47+08:00
draft: false
---

## 前言

Jupyterhub 允许用户通过网页与计算环境交互，用户可以使用浏览器实现在线或者离线的数据分析，从而摆脱本地计算环境的资源限制。有了 JupyterHub，你就可以管理多个用户的 jupyter 服务。

![jupyterlab](https://jupyter.org/assets/homepage/labpreview.webp)

[Kubernetes](https://kubernetes.io/) 是一个容器编排服务。简单来说，Kubernetes 可以管理一个集群（包含多个计算机），我们可以在 kubernetes 上部署应用，这些应用会被 Kubernetes 调度到集群中的某台机器上运行。用户只需要关心应用运行结果，而不必关心资源调度与分配的过程。

> 注: Kubernetes 又叫做 k8s, 原因是在 k 和 s 之间有 8 个字母。

那么我们是否可以将 Jupyterhub 和 Kubernetes 结合，用来支持多用户 jupyter 服务的申请，并且让Kubernetes 实现资源的管理与分配呢。

实际上，Jupyterhub 官方已经支持，这个项目叫做 [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/en/latest/index.html)，感兴趣的可以直接看 jupyterhub 的官方文档。

本文将基于 [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org/en/latest/index.html) 官方文档较少的方法，定制一个适合多用户深度学习计算平台。

### 概念介绍

这里简单粗浅的介绍一些概念，以便于大家对后续流程的理解

默认大家已经理解**操作系统**、**进程**、**存储**的概念

**虚拟机**就是模拟出来的一台完整的计算机系统。一台电脑利用虚拟机可以模拟出很多机器。但是这里我们并不使用虚拟机，介绍虚拟机是为了引出另一个概念——容器化。

**容器化**和虚拟机类似，它也可以在一台机器上模拟出很多计算系统。但是容器化技术更轻量一些，后面我们介绍的多用户环境其实都是容器化技术模拟出来的。感兴趣的可以搜一下容器化和虚拟机的区别。这里我们只需要知道，我们将使用容器化技术为每个用户创建出独立且相互隔离的计算环境。

如何使用虚拟机想必大家都知道，例如 Vmware、virtualBox 等等。那么如何创建容器呢，那就不得不提大名鼎鼎的 [Docker](https://yeasy.gitbook.io/docker_practice/introduction/what)，感兴趣的可以了解一下。不过很遗憾的是，我们没有使用 docker 作为容器化部署的工具，而是使用更轻量更符合云原生标准的 **containerd**。感兴趣的可以了解一下这两个区别。

好了，前面介绍的只需要知道，我们将使用容器化技术来实现多用户独立的计算环境。接下来我们将介绍一些 k8s 的概念。

首先，**Pod**是 k8s 的最小执行单元。这句话怎么理解呢？如果将 k8s 看作是一个操作系统的话，那么 pod 就是这个操作系统的进程。大家知道进程是操作系统进行资源分配和调度的基本单位。同理，pod 也是 k8s 中资源分配和调度的基本单位。还是优点绕？说白了，pod就是我们前面提到的容器，只不过在 k8s 中叫 pod。一个 pod 就是一个独立的计算环境，可以理解成一台虚拟机。

为了更好的理解我们所说的应用，我们还需要介绍一个概念——**镜像**。镜像这个概念大家并不陌生，大家用过各种版本的操作系统，例如 Windows7、Windows10 以及 Ubuntu20.04 等。我们常说的装机其实都要先准备一个系统镜像。同理，容器也有容器镜像（由于 docker 太流行了，所以很多情况下我们也叫 docker 镜像，大家只需要知道这是一个概念就行）。容器镜像中也预装好了各种软件，通常情况下，我们可以直接使用被人封装好的镜像，如果有特殊需求，我们也可以自己制作定制化的镜像，这个我们会在后面介绍，这里只需要知道镜像的概念即可。

我们知道，创建虚拟机需要在虚拟机软件上配置好各种资源，例如 cpu、内存、存储、网络等, 然后在虚拟机软件上创建一个虚拟机。那么在 k8s 上如何创建一个 pod 呢？ 这里需要补充一点，k8s 的所有服务都是声明式的，什么是声明式？就是把所有需要的资源（镜像、计算资源、存储资源、网络）全写到一个配置文件，然后把这个配置文件提交到集群，即可创建出你想要的 pod。

有人会想，既然把 k8s 类比成操作系统，pod 类比成进程（软件）。那么，有没有 pod 的包管理器呢？例如 ubuntu 下我可以使用`apt install vim`来安装一个软件。那么 k8s 是否也可以这样安装应用，针对特定的应用我只需要调整启动参数即可。k8s 确实有应用市场，也有类似的包管理器，叫 [helm](https://helm.sh/) ，在接下来的介绍中，我们将介绍如何使用 helm 一件部署 juputerhub 到 k8s 集群。

## 管理文档

### 搭建 kubernetes 集群

参考[kubespray 一件部署 k8s ]({{< ref "kubespray-deploy-k8s" >}} "deply")

### 安装 kubectl

kubectl 命令是操作 Kubernetes 集群的最直接和最高效的途径

安装 kubectl 参考文档 [Install kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

开启 kubectl 自动补全参考文档 [Enable shell autocompletion](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#optional-kubectl-configurations-and-plugins)

#### kubectl 配置

如何使用 kubectl 连接到远程集群呢？其实 kubectl 有一个配置文件，配置文件中描述了集群的地址和端口，以及连接证书。我们只需要获取到集群的 kubeconfig 文件即可连接到集群。

默认情况下，kubectl 读取 `~/.kube/config` 文件。

如果我们想指定其他目录下的config.yaml 文件，我们可以手动执行，例如
```shell
kubectl --kubeconfig /path/to/kubeconfig get pods

# 添加 alias
alias k='kubectl --kubeconfig /path/to/kubeconfig'
alias kubectl='kubectl --kubeconfig /path/to/kubeconfig'
alias helm='helm --kubeconfig /path/to/kubeconfig'
```

kubeconfig 样例
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ""
    server: https://172.16.0.100:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: kubernetes-admin
  name: kubernetes-admin@cluster.local
current-context: kubernetes-admin@cluster.local
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: ”“
    client-key-data: ""
```

如果想管理多个集群，可以参考配置 [Kubectl 上下文和配置](https://kubernetes.io/zh-cn/docs/reference/kubectl/quick-reference/#kubectl-context-and-configuration)

#### 命令技巧

首先需要补充一点，k8s 下的几乎所有资源都是有 namespace（简称 ns） 的，ns 用来隔离集群中的资源（一般不通的应用使用独立的 ns）。不指定的情况下默认都在 default ns 下。所以当我们想要部署或者查询资源时，都要指定 ns。

1. 查询 k8s 核心组件的 pod 状态(在 kube-system ns 下)

```shell
kubectl -n kube-system get pods
```

2. 查看 k8s 某 pod 的 日志
```shell
kubectl -n kube-system logs pod-name
```

3. 如果某个 pod 部署没有成功，可以查看 pod 的详细信息以及 event
```shell
kubectl -n kube-system describe pod-name
```

更多命令技巧查看 [kubectl 快速参考](https://kubernetes.io/zh-cn/docs/reference/kubectl/quick-reference/)

### 安装 helm

安装helm有几种形式
- 二进制安装
- 脚本安装
- 包管理器安装
- 源码安装

具体安装方式可以参考 [helm install](https://helm.sh/docs/intro/install/)

#### 命令文档

参考 [helm command](https://helm.sh/docs/helm/helm/)

## 用户文档

### 用户手册

查看 [Jupyterhub Userguide ]({{< ref "jupyterhub-userguide" >}})

### docker 镜像制作

TODO：

### github action 自动构建镜像
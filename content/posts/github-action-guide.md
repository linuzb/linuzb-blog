---
title: "Github Action 简明教程"
date: 2024-05-06T10:01:18+08:00
draft: false

tags:
- CI/CD
- DevOps
- Github Action

categories: 
- Cloud Native

authors:
- "linuzb"
---

## 前言

本文的目的是介绍一种自动化的构建、测试、部署流程。

我们在开发软件时，常常需要构建环境、编译、测试以及部署，这些工作都是简单重复的工作，一旦我们的软件及环境复杂，这些工作就会变成一个重复且枯燥的过程。例如，当我们使用了 Docker 之后，我们构建环境就变成了一个 docker build 的过程。每当我们对环境有所修改，我们都需要重新 docker build 一次。每当我们修改一次代码，都需要忍受长时间的编译和测试。 这些工作都有标准的命令，我们是否可以写一个脚本，每当我们对代码进行修改，就可以自动执行以上工作呢？ 当然可以，不过 Github Action 就是这么一个平台，通过 Github Action 我们可以自定义修改代码之后的工作流程（构建镜像、编译、测试和部署等）。

感兴趣的可以搜索下 DevOps 和 CI/CD(持续集成和持续部署)。这是一种软件开发理念，这里我们不做过多介绍，接下来的内容是 CI/CD 的一种实现。

这里看一下 Github Action官方定义

> GitHub Actions 是一种持续集成和持续交付 (CI/CD) 平台，可用于自动执行生成、测试和部署管道。 您可以创建工作流程来构建和测试存储库的每个拉取请求，或将合并的拉取请求部署到生产环境。

## Github Action 的组件

我们可以配置 Github Action workflow(工作流)，当仓库中发生特定的事件时可以触发该工作流。如下图所示，下图即为一个工作流，一个工作流包含多个作业，这些作业可以并行可以串行。每个作业中运行了我们定义的脚本（例如docker build、编译、测试等）。

![workflow](https://docs.github.com/assets/cb-25535/mw-1440/images/help/actions/overview-actions-simple.webp)

GitHub Actions 主要有以下几个概念

- **Workflows**：工作流，可以添加到存储库中的自动化过程。工作流由一个或多个作业组成，可以由事件调度或触发。
- **Event**：事件，触发工作流的特定动作。例如，向存储库提交 pr 或 pull 请求。
- **Jobs**：作业，在同一跑步器上执行的一组步骤。默认情况下，具有多个作业的工作流将并行运行这些作业。
- **Steps**：步骤，可以在作业中运行命令的单个任务。步骤可以是操作，也可以是 shell 命令。作业中的每个步骤都在同一个运行程序上执行，从而允许该作业中的操作彼此共享数据。
- **Actions**：操作是独立的命令，它们被组合成创建作业的步骤。操作是工作流中最小的可移植构建块。你可以创建自己的动作，或者使用 GitHub 社区创建的动作。
- **Runners**：运行器，安装了 GitHub Actions 运行器应用程序的服务器。。Github 托管的运行器基于 Ubuntu Linux、Microsoft Windows 和 macOS，工作流中的每个作业都在一个新的虚拟环境中运行。

## 快速入门

本文是一个入门教程，仅起到抛砖引玉的作用，详细概念介绍和使用方法以可见参考文档或者自行搜索。

这里介绍一些例子快速上手。

### 自动构建 docker 镜像并推送到 dockerhub

#### 准备

- 注册 [Dockerhub](https://hub.docker.com/)（或者其他镜像中心）
- 将 [Dockerhub](https://hub.docker.com/) 的用户名密码添加到 github action secret中。这一步的操作是将用户名密码放到一个 github action 可以读取的配置中，这个配置以 secret 形式保存，仅 github action可见。
- 在 github 仓库中编写 Dockerfile
- 添加 Actions 配置文件，这个文件定义了如何使用以上的 Dockerfile 构建镜像

#### Dockerfile

```dockerfile
# Copyright (c) Jupyter Development Team.
# Distributed under the terms of the Modified BSD License.
ARG REGISTRY=quay.io
ARG OWNER=jupyter
ARG BASE_CONTAINER=$REGISTRY/$OWNER/scipy-notebook
FROM $BASE_CONTAINER

LABEL maintainer="Jupyter Project <jupyter@googlegroups.com>"

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

RUN apt-get update --yes \
    && apt-get install --yes --no-install-recommends \
    wget \
    curl \
    vim \
    git \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Installation de Code Server et server-proxy/vscode-proxy pour intégrer Code dans JupyterLab
# Install VSCode extensions
ENV CODE_VERSION=4.22.1
RUN curl -fOL https://github.com/coder/code-server/releases/download/v$CODE_VERSION/code-server_${CODE_VERSION}_amd64.deb && \
    dpkg -i code-server_${CODE_VERSION}_amd64.deb && \
    rm -f code-server_${CODE_VERSION}_amd64.deb && \
    EXT_LIST="ms-python.python ms-python.debugpy zhuangtongfa.material-theme" && \
    for EXT in $EXT_LIST; do code-server --install-extension $EXT; done

USER ${NB_UID}

# https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/docker-specialized.html#dockerfiles
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility
```

如何编写 Dockerfile 可以参考 [docker 教程]({{< ref "docker-doc" >}})

#### github action 配置

github action 的配置都统一放在 `.github/workflows/` 目录下，该目录下可以放多个 workflow 配置。

例如以下 build docker 的配置 `.github/workflows/build-docker.yaml`

```yaml
name: build_docker

on:
  # 提交代码时触发
  push:
    # 触发条件，以下为与的关系
    # 1. 向 main 分支或者 feat/ 开头的分支提交代码
    # 2. 仅当 src/docker/ 目录下的文件有变化
    branches:
    - main
    - feat/**
    paths:
    - 'src/docker/**'
  # 手动触发
  workflow_dispatch:

env:
  # 定义环境变量，在下面可以这样用 ${{ env.TARGET_NAMESPACE }}
  TARGET_NAMESPACE: linuzb

jobs:
  build_docker:
    name: Build docker
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      # 登录到 dockerhub
      # 需要提前将 DOCKERHUB_USERNAME 和 DOCKERHUB_TOKEN 放到 github action secret 中
      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      # 获取 git commit hash 值或者 tag 值，并存储到环境变量 APP_VERSION 中
      - name: Generate App Version
        run: echo APP_VERSION=`git describe --tags --always` >> $GITHUB_ENV

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v4
        with:
          # 切换上下文到 Dockerfile 所在目录
          context: src/docker
          # 指定 Dockerfile 目录，前面 context 已经制定了dockerfile所在目录，这里可以忽略
          # file: src/docker/Dockerfile
          push: true
          platforms: linux/amd64,linux/arm64
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/${{ env.TARGET_NAMESPACE }}:${{ github.ref_name }}
            ${{ secrets.DOCKERHUB_USERNAME }}/${{ env.TARGET_NAMESPACE }}:latest
```

## 推荐

- [SSH 连接到 GitHub Actions 虚拟服务器(VM/VPS)](https://p3terx.com/archives/ssh-to-the-github-actions-virtual-server-environment.html)

## 参考

- 示例项目 [linuzb/devcontainer](https://github.com/linuzb/devcontainer)
- [GitHub Actions 快速入门](https://docs.github.com/zh/actions/quickstart)
- [了解 GitHub Actions](https://docs.github.com/zh/actions/learn-github-actions/understanding-github-actions?learn=getting_started&learnProduct=actions)
- [阮一峰|GitHub Actions 入门教程](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)
- [Publish a Docker image on DockerHub with GitHub Actions](https://blog.pradumnasaraf.dev/dockerhub-githubactions)
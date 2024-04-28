---
title: "Jupyterhub Userguide"
date: 2024-04-28T16:33:37+08:00
draft: false

tags:
- Kubernetes
- Jupyterhub
- Jupyterlab
- Jupyter
- Docker
- VsCode
- Pycharm

categories: 
- Cloud Native

authors:
- "linuzb"

---

## 前置准备

本文是 jupyterhub on kubernetes 的使用文档，本文描述的环境基于 [jupyterhub on kubernetes]({{< ref "jupyterhub-on-kubernetes" >}})

在上述文档中，我们部署的 kubernetes master 节点 nodeIP 为 172.16.0.100, 因此以下教程中使用的访问 ip 都是 172.16.0.100

## 环境准备

### 内网访问

可以访问内网的浏览器

## 登录

使用浏览器访问 http://172.16.0.100:30080/
![hub-login](/images/hub-login.png)

### 认证

点击 Singn in with Github，然后在 github 页面授权

说明：
- 这里仅使用 github oauth 做认证鉴权，暂无其它依赖，原理课参考文档 [Authorizing OAuth apps - GitHub Docs](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps)
- github 账号的用户名需要为全字母及数字，暂不支持非字母数字之外的其他符号

## 应用选择

登录完成之后就会进入一个应用服务页面

选择一个合适的应用环境，然后点击底部的 start 按钮，即可进入环境
![server-options](/images/server-options-0.png)

### 自定义应用

TODO： 待补充

# 应用环境介绍

## 功能介绍

### 界面简介

应用创建完成后，就会跳转到jupyterlab页面
![jupyterlab](/images/jupyterlab-home.png)


介绍一下主要功能

1. 启动一个luncher，即页面右侧部分，在 luncher 中可以打开各种工具
2. 上传文件按钮，点击后弹出一个对话框，选择需要上传到文件即可。如果想上传一个目录，可以先在本地将目录打包成一个压缩包，然后上传。上传前可以先按照3. 切换到对应的工作目录。
3. 工作目录，双击可打开文件以及文件夹。上传文件以及打开vscode都是以工作目录为基础，例如，当前工作目录为`~/Projects/data`, 则上传目标路径即为`~/Projects/data`
4. 新建一个jupyter notebook
5. 以工作目录为基础打开vscode
6. 打开一个terminal，可执行linux命令

### vscode

打开方式：在luncher 中点击vscode 图标即可打开

如何返回jupyter 界面？
由于vscode界面没做跳转链接，将浏览器url中的 `/vscode` 及之后的内容全部删除即可返回jupyter页面

#### 切换目录

方法1： 返回jupyter 主界面，从左边工作栏切换目标项目目录，然后重新打开vscode
![vscode-cd](/images/vscode-change-workdir.png)

方法2： 直接在vscode菜单栏切换
![vscode-open-folder](/images/vscode-open-folder.png)

#### 插件安装

插件默认安装在用户 home 目录下，由于该目录是持久化的，直接在插件市场安装即可，后续环境的重建不会影响插件。

#### 用户配置

持久化方案和插件一样，直接修改配置即可

### Terminal

#### 命令

已添加常用命令
- git
- zip, unzip
- curl

#### oh-my-zsh

TODO： 待补充如何修改自定义shell即配置

## 存储

由于该软件底层使用的是容器服务，一旦应用被删除（底层容器被删除），则容器中的文件都将被删除。

本方案为每个用户的 `/home/{username}` 目录挂载了持久化存储，保存在 home 目录下的文件是持久化的，除非人工手动删除，否则容器的销毁不会影响该目录下的文件。因此，持久化的文件一定要保存在 home目录下。

首次登录验证用户目录是否持久化：首先进入 home 目录（在luncher中新开一个 terminal），输入命令 `cd ~/ && touch test.txt`, 该命令是进入当前用户的home目录，然后销毁环境，重新进入环境查看 `~/test.txt` 是否存在。如果存在，说明已经挂载持久化盘，如果不存在请联系管理员

## 网络

### 外网

未做外网限制，请注意流量消耗

TODO： 流量监控

### kubernetes 内网

可以访问kubernetes 内网，即可以向 kubernetes 提交作业，待补充。

## 销毁环境

方法1：首先进入 jupyterlab 页面，点击 File->Hub Control Panel
![hub-control-link](/images/hub-control-panel-link.png)

然后进入新的界面
![stop-server](/images/stop-server.png)

双击 Stop My Server 按钮即可销毁环境

方法2 直接在浏览器输入 stop server 页面`/hub/home`, 即 http://172.16.0.100:30080/hub/home

### 自动销毁环境

为了提高资源利用率，本系统使用了超时自动销毁服务的功能。

当用户超过7d没有通过浏览器访问应用时，系统将自动销毁应用环境。用户再次登陆后需要重新选择并创建环境。

注意：
1. 超时时间时按照用户访问浏览器 url 的流量计算的，并非应用本身的活动
2. 应用的销毁并不会影响用户home目录，但是如果应用销毁时刚好有写用户home目录的操作，则可能会有影响

# 监控

监控页面地址 http://172.16.0.100:30759/?orgId=1

参考 [kubernetes monitoring]({{< ref "kubernetes-monitoring" >}})

# 自定义应用环境

待补充
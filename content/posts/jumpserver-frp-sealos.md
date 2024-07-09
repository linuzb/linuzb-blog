---
title: "Jumpserver Frp Sealos"
date: 2024-07-09T16:57:55+08:00
draft: false

tags:
- Docker

categories: 
- DevOps

authors:
- "linuzb"
---

## 前言

我们常常会遇到访问内网的场景（内网穿透）。[JumpServer](https://docs.jumpserver.org/zh/v2/)是广受欢迎的开源堡垒机，是符合 4A 规范的专业运维安全审计系统。我们可以使用 jumpserver 作为跳板机访问内网服务。

但是仅仅有 jumpserver 还不够，我们需要实现内网穿透，即通过公网访问内部的 jumpserver。因此这里用到了 [fatedier/frp](https://github.com/fatedier/frp)，一个反向代理工具，可帮助您将 NAT 或防火墙后的本地服务器接入互联网。

由于 jumpserver 部署在内网，且通过 http 暴露服务，如果我们直接使用 frp 代理 http 服务，很显然达不到安全的需求，因此我们还需要在公网使用一个网关，将 http 应用封装成 https 服务。一种实现方式是购买云主机和域名，部署 nginx 并通过 Let's Encrypt [Let's Encrypt](https://letsencrypt.org/)签发证书。但是这样涉及到金钱成本以及备案成本。我们暂不考虑。

这里介绍的暴露 frp 服务的方式是使用 [Sealos Cloud](https://bja.sealos.run)。Sealos 是一个云上的 kubernetes 平台，我们可以直接在上面部署 frp pod，然后利用 Sealos 提供的公网网关来代理我们的服务。


## 部署

### JumpServer

首先我们在内网的一台机器上部署 jumpserver。为了方便部署，我们使用 docker 部署

这里参考 [jumpserver](https://github.com/jumpserver/Dockerfile/blob/master/README.md) docker 部署文档。

```shell
# 测试环境可以使用，生产环境推荐外置数据
git clone --depth=1 https://github.com/jumpserver/Dockerfile.git
cd Dockerfile
cp config_example.conf .env
docker compose -f docker-compose-network.yml -f docker-compose-redis.yml -f docker-compose-mariadb.yml -f docker-compose-init-db.yml up
docker compose -f docker-compose-network.yml -f docker-compose-redis.yml -f docker-compose-mariadb.yml -f docker-compose.yml up -d

docker rm jms_init_db
```

为了避免 jumpserver 80端口和宿主机 80端口冲突，这里我们修改 `.env` 文件的 `HTTP_PORT` 为 `18080`

```
# Web
HTTP_PORT=18080
```

部署完成后，在内网可以直接使用 `http://nodeip:18080` 访问 jumpserver 服务

#### jumpserver 配置

1. 给 admin 账号配置 MFA 认证
2. 创建用户，用户配置 MFA 认证
3. 创建资产
4. 创建账号
5. 将资产授权给用户

### Sealos Cloud 部署 frp

#### 创建应用

##### 基础配置

应用名称：frps
镜像源：snowdreamtech/frps

##### 网络配置

| 容器暴露端口 | 开启公网访问 | 备注                                         |
| ------ | ------ | ------------------------------------------ |
| 7000   | 关闭     | 由于 ingress 无法开启 tcp 转发，接下来我们将创建一个 nodePort |
| 7080   | 开启     | 使用 https 协议，用于代理 jumpserver http 服务        |

##### 高级配置

配置文件名： /etc/frp/frps.toml
配置文件：

```yaml
bindPort = 7000
auth.method = "token"
auth.token = ""
```
#### 创建 NodePort

在以上frp的配置中，服务端使用 7000 端口监听，使用的协议是 TCP。

Sealos Cloud 的应用创建界面中，网络配置开启公网访问实际上是一个 nginx ingress 的网关，Sealos Cloud 的应用创建接口无法直接代理 TCP 协议。因此，我们将创建一个 NodePort 类型的Service。

我们打开 Sealos Cloud 的在线终端，使用 vim 创建 svc.yaml 文件

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    cloud.sealos.io/app-deploy-manager: frps
  name: frps-nodeport
spec:
  ports:
  - name: jumpserver
    port: 7000
    protocol: TCP
    targetPort: 7000
  selector:
    app: frps
  sessionAffinity: None
  type: NodePort
```

这个 service 只暴露 7000 端口。

```shell
kubectl get svc frps-nodeport
```

我们记录下 7000 端口对应的 NodePort 35934，这个 35934 是客户端连接 server 的端口号。

#### 客户端 frp

我们同样使用 docker 部署客户端 frp

docker-compose.yaml

```yaml
version: '3'
services:
  frps:
    image: snowdreamtech/frpc
    container_name: frps
    restart: always
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml
    deploy:
      mode: replicated
      replicas: 1
```

frpc.toml
```toml
serverAddr = "xxx.bja.sealos.run" # 公网网络配置中的 host 地址
serverPort = 35934 # 7000 对应的 nodePort
auth.token = ""

[[proxies]]
name = "jumpserver"
type = "tcp"
localIP = "172.16.0.124"
localPort = 18080
remotePort = 7080
```

部署frp

```shell
docker compose up -d
```
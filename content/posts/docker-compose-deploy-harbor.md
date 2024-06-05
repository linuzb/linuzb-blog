---
title: "Docker Compose Deploy Harbor"
date: 2024-06-05T08:55:26+08:00
draft: false

tags:
- Harbor

categories: 
- Cloud Native

authors:
- "linuzb"
---


## 下载 harbor 安装器

首先到官方 release 页面查找对应的版本号，使用以下命令下载

```shell
export HARBOR_VERSION=v2.9.4
wget https://github.com/goharbor/harbor/releases/download/v${HARBOR_VERSION}/harbor-offline-installer-v${HARBOR_VERSION}.tgz
tar xvf harbor-offline-installer-v${HARBOR_VERSION}.tgz
cd harbor
mv harbor.yml.tmp harbor.yml  
```

## 编辑 harbor.yml

```shell
vim harbor.yml
```

```yaml
# 将 hostname 设置成自己的域名或者 ip
hostname: 172.16.0.115

http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  # 修改端口
  port: 6060

# 如果没有 https 可以将 https 相关的配置注释
# https related config
# https:
#   # https port for harbor, default is 443
#   port: 443
#   # The path of cert and key files for nginx
#   certificate: /your/certificate/path
#   private_key: /your/private/key/path

# The default data volume
data_volume: /data
```

详细内容参考文档 [Configure the Harbor YML File](https://goharbor.io/docs/1.10/install-config/configure-yml-file/)

## 生成配置并部署 docker compose

执行以下命令

```shell
sudo ./install.sh
```

部署完成后即可通过 url 访问，本例中是 `http://172.16.0.155:6060`

默认密码查看文档 [Configure the Harbor YML File](https://goharbor.io/docs/1.10/install-config/configure-yml-file/)

## 客户端

### 配置私有镜像仓库

修改 `/etc/docker/daemon.json` 文件，添加 `insecure-registries`

```json
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    },
    "registry-mirrors": [
      "https://docker.m.daocloud.io",
      "https://dockerproxy.com",
      "https://docker.mirrors.ustc.edu.cn",
      "https://docker.nju.edu.cn"
    ],
    "insecure-registries": [
      "172.16.0.115:6060"
    ]
}
```

重启容器

```sehll
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 登录 harbor

```shell
docker login 172.16.0.155:6060 -u admin

# 输入密码
Harbor12345
```

### 推送镜像

```shell
# docker tag image_id(本地需要push的镜像) ip:port/项目名/保存的镜像名(例如：xxx:version1)
docker tag goharbor/harbor-exporter:v2.9.4 172.16.0.155:6060/library/goharbor/harbor-exporter:v2.9.4
```

### 拉取镜像

```shell
# docker pull ip:port/项目名/保存的镜像名(例如：xxx:version1)
docker pull 172.16.0.155:6060/library/goharbor/harbor-exporter:v2.9.4
```

> 公开镜像可以无需登录 harbor 即可拉取

## 代理镜像仓库

我们可以使用代理缓存功能让一些访问受限环境能够访问互联网上的镜像，并且如果没有某个镜像，此时客户端第一次发起pull image 请求会从指定的代理仓库下载并缓存到harbor的仓库里，下次别的客户端再需要pull 这个镜像就无需从公网再去下载该镜像了，从而避免占用过多带宽或被docker hub 速率限制。

具体操作参考文档 [使用harbor代理缓存docker hub](https://www.lishuai.fun/2020/11/05/harbor-proxy/)

## 参考

- [通过docker-compose 快速部署 harbor - 掘金 (juejin.cn)](https://juejin.cn/post/7223027325037789241)
- [Harbor docs | Harbor Installation and Configuration (goharbor.io)](https://goharbor.io/docs/1.10/install-config/)
---
title: "Jupyterhub on Kubernetes"
date: 2024-04-28T15:57:55+08:00
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

- 首先需要部署 kubernetes 集群，参考[k8s deploy]({{< ref "kubespray-deploy-k8s" >}} "deply")
- [Helm | Installing Helm](https://helm.sh/docs/intro/install/) 安装 helm

### 存储：Longhorn provider

部署 Longhorn 参考 [Longhorn deploy]({{< ref "kubespray-deploy-k8s" >}})

### 接入 github oauth

本文使用 github oauth 作为认证方式，首先我们需要提前注册一个 GitHub OAuth 应用程序，请参阅 GitHub 有关注册应用程序的官方文档 [registering an app](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app).

然后我们得到三个信息
1. client-id
2. client-secret
3. oauthCallbackUrl

为了避免在配置中明文保存密码，我们将以上信息通过 secret 的形式存储在 Kubernetes 中，可以使用挂载 secret 的形式使用 secret 变量。

### 创建secret

接下来我们给 client-id 和 client-secret 创建secret

```shell
echo -n "client-id" | base64
echo -n "client-secret" | base64
echo -n "oauthCallbackUrl" | base64
```

将以上结果填充到用于创建 secret 的 yaml 中

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: githubsecret
  namespace: jhub
type: Opaque
data:
  clientId: "base64(clientId)"
  clientSecret: "base64(clientSecret)"
  oauthCallbackUrl: "base64(oauthCallbackUrl)"
```

注意：该 yaml 配置独立存放

## 使用 helm 安装 jupyterhub

完整的 helm values 文件参考 [values.yaml](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/blob/main/jupyterhub/values.yaml)

本文中介绍的功能配置全部放在 `./config.yaml` 文件中，该文件会覆盖 helm chart 中默认的 `values.yaml`

```
# Let helm the command line tool know about a Helm chart repository
# that we decide to name jupyterhub.
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm repo update
# 查看版本
helm search repo jupyterhub -l
# 下载到本地
helm pull jupyterhub/jupyterhub
# 修改 config.yaml 从本地安装
helm upgrade --cleanup-on-fail \
  --install jupyterhub ./jupyterhub \
  --namespace jhub \
  --create-namespace \
  --version=3.3.6 \
  --values config.yaml
```

### config.yaml

TODO: 补充 config.yaml

### jupyterhub 运行其他服务

#### vscode

[How to configure Jupyterhub to run code-server? - JupyterHub - Jupyter Community Forum](https://discourse.jupyter.org/t/how-to-configure-jupyterhub-to-run-code-server/11578)

##### jetbrans

[tiaden/jupyter-projector-proxy (github.com)](https://github.com/tiaden/jupyter-projector-proxy)

## 本地调试

本地运行 single-user-notebook

```shell
docker run \
    -p 8888:8888 \
    -v /home/linuzb/Projects/huggingface/chatglm2-6b:/home/jovyan/data/huggingface/chatglm2-6b \
    --gpus all \
    --rm \
    registry.cn-hangzhou.aliyuncs.com/linuzb/pytorch-notebook:cuda11-pytorch-2.2.1-4cb74da
```

## 参考

可以看下 binderhub
- [Integrating BinderHub with JupyterHub: Empowering users to manage their own environments | 2i2c](https://2i2c.org/blog/2024/jupyterhub-binderhub-gesis/)
- [jupyterhub/binderhub: Run your code in the cloud, with technology so advanced, it feels like magic! (github.com)](https://github.com/jupyterhub/binderhub)
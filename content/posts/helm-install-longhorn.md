---
title: "Kubernetes 部署 longhorn 存储"
date: 2024-04-28T15:46:48+08:00
draft: false

tags:
- Kubernetes
- Longhorn

categories: 
- Cloud Native

authors:
- "linuzb"

---

## 背景

[longhorn](https://github.com/longhorn/longhorn?tab=readme-ov-file) 是基于 Kubernetes 构建的云原生分布式存储


## 依赖

[Installation Requirements](https://longhorn.io/docs/1.6.1/deploy/install/#installation-requirements)

### ubuntu 安装依赖

运行以下脚本，检查是否满足依赖

```shell
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/v1.6.1/scripts/environment_check.sh | bash
```

如果缺少依赖，需要安装依赖，例如

```shell
apt-get install open-iscsi
apt-get install nfs-common
```

[kubectl 安装](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)及 kubeconfig 配置

## 部署

### helm

[Longhorn | Documentation](https://longhorn.io/docs/1.6.1/deploy/install/install-with-helm/)

```shell
helm pull longhorn/longhorn --version 1.6.1
helm upgrade --install longhorn ./longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.6.1 \
  --values config.yaml
```

config.yaml

```yaml
defaultSettings:
  # 配置默认数据存放地址
  defaultDataPath: /data/longhorn

service:
  ui:
    # -- Service type for Longhorn UI. (Options: "ClusterIP", "NodePort", "LoadBalancer", "Rancher-Proxy")
    type: NodePort
    # -- NodePort port number for Longhorn UI. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: 30180
  manager:
    # -- Service type for Longhorn Manager.
    type: NodePort
    # -- NodePort port number for Longhorn Manager. When unspecified, Longhorn selects a free port between 30000 and 32767.
    nodePort: 30181

longhornManager:
  # -- Toleration for Longhorn Manager on nodes allowed to run Longhorn Manager.
  tolerations:
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"

longhornDriver:
  # -- Toleration for Longhorn Driver on nodes allowed to run Longhorn components.
  tolerations: 
  - key: "node-role.kubernetes.io/control-plane"
    operator: "Exists"
    effect: "NoSchedule"
```

## 访问管理页面

Longhorn dashboard http://{EXTERNAL-IP}:30180/#/dashboard

### 故障恢复

#### 从 replica 中恢复
- [[FEATURE] Restoring volume from single replica · Issue #469 · longhorn/longhorn (github.com)](https://github.com/longhorn/longhorn/issues/469)
- [Longhorn | Documentation](https://longhorn.io/docs/1.6.1/advanced-resources/data-recovery/export-from-replica/)


## 参考
- [17. Kubernetes - 持久化存储（Longhorn） - Dy1an - 博客园 (cnblogs.com)](https://www.cnblogs.com/Dy1an/p/17245825.html)

---
title: "Kubernetes Nfs Provider"
date: 2024-04-29T10:47:07+08:00
draft: false
---


## 前言

本文介绍如何使用 nfs 网络存储为 kubernetes 提供动态的PV创建。

本文使用的环境

| 操作系统 | 主机IP | 数据存储目录 |
| ---- | ---- | ---- |
| Ubuntu | 172.16.0.100/24 | /data/nfs |

## 部署

### step1. 在宿主机部署 NFS 服务

```shell
sudo apt update
# 服务端
sudo apt install nfs-kernel-server
```

### step2. 创建共享目录

创建目录`/data/nfs`
```shell
mkdir -p /data/nfs
```

### step3. 宿主机配置 NFS export

```bash
echo -e "/data/nfs\t172.16.0.100/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
```

配置完成后重启 nfs server
```shell
sudo systemctl restart nfs-kernel-server
```
### step4. 测试 nfs 服务可用

```bash
/sbin/showmount -e 172.16.0.100
```


## 部署 NFS Provider

使用helm 部署

```bash
helm upgrade --install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=172.16.0.100 \
  --set nfs.path=/data/nfs \
  --set storageClass.onDelete=true \
  --set image.repository=registry.cn-hangzhou.aliyuncs.com/linuzb/nfs-subdir-external-provisioner
```

- `nfs.server` 使用部署 nfs 的宿主机ip，本文是 `172.16.0.100`
- `nfs.path` 使用部署 nfs 的宿主机共享目录，本文是 `/data/nfs`

## 使用

### 获取 storageClass 名字

```shell
kubectl get sc
```

### 创建 pvc

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: nfs-client
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```
其中 
- `storageClassName` 使用的就是 nfs 的 storageclass 名字 `nfs-client`

### pod 挂载 pvc

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage

```

 对 Pod 而言，PersistentVolumeClaim 就是一个存储卷。

## 参考：

- [How To Set Up an NFS Mount on Ubuntu 20.04 | DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04#step-2-creating-the-share-directories-on-the-host)
- [kubernetes-sigs/nfs-subdir-external-provisioner: Dynamic sub-dir volume provisioner on a remote NFS server. (github.com)](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
- [How to Setup Dynamic NFS Provisioning in a Kubernetes Cluster | by Hakan Bayraktar | Medium](https://hbayraktar.medium.com/how-to-setup-dynamic-nfs-provisioning-in-a-kubernetes-cluster-cbf433b7de29)


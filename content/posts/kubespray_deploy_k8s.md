---
title: "Kubespray_deploy_k8s"
date: 2024-04-28T10:06:36+08:00
draft: false

tags:
- kubernetes
- kubespray

categories: 
- Cloud Native
authors:
- "linuzb"
---

# 部署

参考文档
- [使用Kubespray 部署Kubernetes集群 | Sunday博客 | Sunday Blog (sundayhk.com)](https://www.sundayhk.com/post/kubespray/)
- [使用kubespray安装kubernetes | kubernetes-notes (huweihuang.com)](https://k8s.huweihuang.com/project/setup/installer/install-k8s-by-kubespray)

可参考其它方式 [easzlab/kubeasz: 使用Ansible脚本安装K8S集群](https://github.com/easzlab/kubeasz)

## kubespray

### 自定义配置

```bash
# 拷贝集群清单
cp -rfp inventory/sample inventory/mycluster
```

####  inventory

inventory.ini
```ini
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
kube-master-100 ansible_ssh_host=172.16.0.100 ansible_ssh_user=lzb ip=172.16.0.100   mask=/24
kube-node-117 ansible_ssh_host=172.16.0.117 ansible_ssh_user=lzb ip=172.16.0.117   mask=/24

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
kube-master-100

[etcd]
kube-master-100

[kube_node]
kube-node-117

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```

#### 集群网络

```
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```

使用高性能的 cilium
```yaml
# Choose network plugin (cilium, calico, kube-ovn, weave or flannel. Use cni for generic cni plugin)
# Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
kube_network_plugin: cilium

```

#### 运行时


使用默认的 containerd
inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```yaml
## Container runtime
## docker for docker, crio for cri-o and containerd for containerd.
## Default: containerd
container_manager: containerd
```

容器运行目录(暂不修改)
inventory/mycluster/group_vars/all/containerd.yml
```yaml
# containerd_storage_dir: "/var/lib/containerd"
# containerd_state_dir: "/run/containerd"
```

配置容器registry

inventory/mycluster/group_vars/all/containerd.yml
```yaml
containerd_registries_mirrors:
  - prefix: docker.io
    mirrors:
      - host: https://f7ni6hqo.mirror.aliyuncs.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
      - host: http://hub-mirror.c.163.com
        capabilities: ["pull", "resolve"]
        skip_verify: false
```

#### 集群配置

自动更新集群证书， 默认一年有效
inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```yaml
## Automatically renew K8S control plane certificates on first Monday of each month
auto_renew_certificates: true
```

打开日志报错

inventory/mycluster/group_vars/all/all.yml
```yaml
## Used to control no_log attribute
unsafe_show_logs: true
```

#### 镜像国内源

[kubespray/docs/mirror.md at master · kubernetes-sigs/kubespray (github.com)](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/mirror.md)

```bash
# Use the download mirror
cp inventory/mycluster/group_vars/all/offline.yml inventory/mycluster/group_vars/all/mirror.yml
sed -i -E '/# .*\{\{ files_repo/s/^# //g' inventory/mycluster/group_vars/all/mirror.yml
tee -a inventory/mycluster/group_vars/all/mirror.yml <<EOF
gcr_image_repo: "gcr.m.daocloud.io"
kube_image_repo: "k8s.m.daocloud.io"
docker_image_repo: "docker.m.daocloud.io"
quay_image_repo: "quay.m.daocloud.io"
github_image_repo: "ghcr.m.daocloud.io"
files_repo: "https://files.m.daocloud.io"
EOF
```

### docker 部署

```bash
docker run --rm -it --mount type=bind,source="$(pwd)"/inventory/mycluster,dst=/inventory \
  quay.io/kubespray/kubespray:v2.24.1 bash
```

#### 查看集群配置

```bash
ansible-inventory -i /inventory/inventory.ini --list
```

#### 运行部署
```bash
# 该命令可以成功， 但是要输入两次密码
ansible-playbook -i /inventory/inventory.ini cluster.yml --user lzb --ask-pass --become --ask-become-pass

```

#### 扩容节点

```bash
ansible-playbook -i /inventory/inventory.ini scale.yml \
  --user=lzb --ask-pass --become --ask-become-pass -b \
   --limit=kube-node-114
```

您可以使用--limit=NODE_NAME限制 Kubespray 以避免干扰集群中的其他节点。 在没有使用--limitplaybook会运行facts.yml刷新所有节点的fact缓存。

#### 缩容节点

如果有节点不再需要了，我们可以将其移除集群，通常步骤是:

- 1.`kubectl cordon NODE` 驱逐节点，确保节点上的服务飘到其它节点上去，参考[安全维护或下线节点](https://imroc.cc/kubernetes/best-practices/ops/securely-maintain-or-offline-node.html)。
- 2.停止节点上的一些 k8s 组件 (kubelet, kube-proxy) 等。
- 3.kubectl delete NODE 将节点移出集群。
- 4.如果节点是虚拟机，并且不需要了，可以直接销毁掉。

前3个步骤，也可以用 kubespray 提供的`remove-node.yml`这个 playbook 来一步到位实现:

```sh
ansible-playbook \
  -i inventory/mycluster/inventory.ini \
  --private-key=id_rsa --user=ubuntu -b \
  -e "node=node2,node3" \
  remove-node.yml

ansible-playbook \
  -i inventory/mycluster/inventory.ini \
  --user=lzb --ask-pass --become --ask-become-pass -b \
  -e "node=kube-node-114" \
  remove-node.yml
```

`-e` 里写要移出的节点名列表，如果您要删除的节点不在线，您应该将`reset_nodes=false`和添加`allow_ungraceful_removal=true`到您的额外变量中
### 后续

#### 获取kubeconfig

部署完成后，可以在master节点上的 /root/.kube/config 路径获取到 kubeconfig

获取到kubeconfig后，将 https://127.0.0.1:6443 修改成kube-apiserver负载均衡器的地址:端口,或者其中一台master。

```bash
alias k='kubectl --kubeconfig /home/linuzb/Projects/kube-cert/kubeconfig'
```
#### nvidia container

- [Installing the NVIDIA Container Toolkit — NVIDIA Container Toolkit 1.14.5 documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html#configuring-docker)
- [NVIDIA/nvidia-container-toolkit: Build and run containers leveraging NVIDIA GPUs (github.com)](https://github.com/NVIDIA/nvidia-container-toolkit)
[[linux 开发环境#GPU#安装 nvidia-device-plugin]]

# debug

## dns
[Debugging DNS Resolution | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dnsutils
  namespace: default
spec:
  containers:
  - name: dnsutils
	image: lunettes/lunettes:v0.1.5
    # image: registry.cn-hangzhou.aliyuncs.com/linuzb/jessie-dnsutils:1.3
    command:
      - sleep
      - "infinity"
    imagePullPolicy: IfNotPresent
  restartPolicy: Always
```

## 网络排查

```bash
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl -vvsSk -H "Authorization: Bearer $KUBE_TOKEN" https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT/api/v1/namespaces/jhub/pods


curl -sSk -H "Authorization: Bearer $KUBE_TOKEN" https://172.16.0.100:6443/api/v1/namespaces/jhub/pods
```

## 网络排查

### nodelocaldns

kubernetes 节点重启后，nodelocaldns crash

原因：[loop (coredns.io)](https://coredns.io/plugins/loop/#troubleshooting)

解决方案
参考 [NodeLocalDNS Loop detected for zone "." · Issue #9948 · kubernetes-sigs/kubespray (github.com)](https://github.com/kubernetes-sigs/kubespray/issues/9948)

删除coredns [重置和重新安装kubernetes中的coreDNS_如何删除现在有的coredns-CSDN博客](https://blog.csdn.net/u013007181/article/details/129731938)

> 1. Change settings in k8 config file for kubespray. I changed 2 things: `resolvconf_mode: none` and `remove_default_searchdomains: false`

然后使用 kubespray 重新部署 kubernetes
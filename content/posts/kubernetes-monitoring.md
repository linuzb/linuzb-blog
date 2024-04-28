---
title: "kubernetes 监控部署"
date: 2024-04-28T15:09:33+08:00
draft: false

tags:
- Kubernetes
- Prometheus
- Monitoring
- GPU Monitoring
- DCGM

categories: 
- Cloud Native

authors:
- "linuzb"

---

## 准备

1. 首先需要部署 kubernetes 集群，参考[k8s deploy]({{< ref "kubespray-deploy-k8s" >}} "deply")


## helm 安装 prometheus 软件栈

[Setting up Prometheus — NVIDIA GPU Telemetry 1.0.0 documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-telemetry/latest/kube-prometheus.html#about-setting-up-prometheus)

使用以下命令安装 prometheus

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
   --namespace prometheus \
   --create-namespace \
   --set prometheus.service.type=NodePort \
   --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
   --set prometheusOperator.admissionWebhooks.patch.image.registry=registry.cn-hangzhou.aliyuncs.com \
   --set prometheusOperator.admissionWebhooks.patch.image.repository=linuzb/kube-webhook-certgen \
   --set kube-state-metrics.image.registry=registry.cn-hangzhou.aliyuncs.com \
   --set kube-state-metrics.image.repository=linuzb/kube-state-metrics
```

## 安装 GPU 监控 DCGM

```bash
helm upgrade --install \
   dcgm-exporter \
   gpu-helm-charts/dcgm-exporter \
   --values config.yaml
```

master 节点也部署
config.yaml
```yaml
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```

导入 gpu 监控 dashboard `https://grafana.com/grafana/dashboards/12239`

## 使用 grafana

修改 svc 为 node prot

```bash
k -n prometheus edit svc kube-prometheus-stack-grafana 
```

增加内容

```yaml
spec:
  - name: http-web
    nodePort: 30759
  type: NodePort
```

## dashboard

默认密码 prom-operator

```yaml
# Deploy default dashboards.
#
defaultDashboardsEnabled: true

adminPassword: prom-operator
```




---
title: 监控单个 K8s 集群
---

# 简介

假如你在一个 K8s 集群中部署了应用，本章介绍如何使用 MetaFlow 进行监控。MetaFlow 能够自动采集所有 Pod 的应用和网络观测数据（AutoMetrics、AutoTracing），并基于调用 apiserver 获取的信息自动为所有观测数据注入容器资源和自定义 Label 标签（AutoTagging）。

# 准备工作

## 部署拓扑

```mermaid
flowchart TD

subgraph K8s-Cluster
  APIServer["k8s apiserver"]

  subgraph MetaFlow Backend
    MetaFlowServer["metaflow-server (statefulset)"]
    ClickHouse["clickhouse (statefulset)"]
    MySQL["mysql (deployment)"]
    MetaFlowApp["metaflow-app (deployment)"]
    Grafana["grafana (deployment)"]
  end

  subgraph MetaFlow Frontend
    MetaFlowAgent["metaflow-agent (daemonset)"]
  end

  MetaFlowAgent -->|"control & data"| MetaFlowServer
  MetaFlowAgent -->|"get resource & label"| APIServer

  MetaFlowServer --> ClickHouse
  MetaFlowServer --> MySQL
  MetaFlowApp -->|sql| MetaFlowServer
  Grafana -->|"metrics/logging (sql)"| MetaFlowServer
  Grafana -->|"tracing (api)"| MetaFlowApp
end
```

## Storage Class

我们建议使用 Persistent Volumes 来保存 mysql 和 clickhouse 的数据，以避免不必要的维护成本。
你可以提供默认 Storage Class 或添加 `--set global.storageClass=<your storageClass>` 参数来选择 Storage Class 以创建 PVC。

可选择 [OpenEBS](https://openebs.io/) 用于创建 PVC：
```console
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```

## metaflow-agent 权限需求

metaflow-agent 需要的容器节点权限如下：
- `hostNetwork`
- `hostPID`
- `privileged`
- Write `/sys/kernel/debug`

metaflow-agent 需要的容器节点配置如下：
- `selinux` = `Permissive` OR `disabled`
  
metaflow-agent 需要以下 Kubernetes 资源的 get/list/watch 权限：
- `nodes`
- `namespaces`
- `configmaps`
- `services`
- `pods`
- `replicationcontrollers`
- `daemonsets`
- `deployments`
- `replicasets`
- `statefulsets`
- `ingresses`
- `routes`

# 部署 MetaFlow

使用 Helm 安装 MetaFlow：
```console
helm repo add metaflow https://metaflowys.github.io/metaflow
helm repo update metaflow
helm install metaflow -n metaflow metaflow/metaflow --create-namespace
```

注意：
- 虽然你可以使用 helm `--set` 参数来定义部分配置，但我们建议将自定义的配置保存一个独立的 yaml 文件中。
  例如 `values-custom.yaml` ：
  ```yaml
  global:
    storageClass: "<your storageClass>"
    replicas: 1  ## replicas for metaflow-server and clickhouse
  ```
  后续更新可以使用 `-f values-custom.yaml` 参数使用自定义配置：
  ```console
  helm upgrade metaflow -n metaflow -f values-custom.yaml metaflow/metaflow
  ```
  
# 下载 metaflow-ctl

metaflow-ctl 是管理 MetaFlow 的一个命令行工具，建议下载至 metaflow-server 所在的 K8s Node 上：
```console
curl -o /usr/bin/metaflow-ctl https://metaflow.oss-cn-beijing.aliyuncs.com/bin/ctl/latest/linux/amd64/metaflow-ctl
chmod a+x /usr/bin/metaflow-ctl
```

# 下一步

- [自动分布式追踪 - 体验 MetaFlow 基于 eBPF 的 AutoTracing 能力](../auto-tracing/overview/)
- [微服务全景图 - 体验 MetaFlow 基于 BPF 的 AutoMetrics 能力](../auto-metrics/overview/)
- [消除数据孤岛 - 了解 MetaFlow 的 AutoTagging 和 SmartEncoding 能力](../auto-tagging/elimilate-data-silos/)
- [无缝分布式追踪 - 集成 OpenTelemetry 等追踪数据](../agent-integration/tracing/overview/)
- [告别高基烦恼 - 集成 Promethes 等指标数据](../agent-integration/metrics/overview/)
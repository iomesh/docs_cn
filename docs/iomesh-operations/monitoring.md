---
id: monitoring
title: 监控
sidebar_label: 监控
---

可以使用 Prometheus 和 Grafana 监控 IOMesh 集群。

## 与 Prometheus 集成

如果 Prometheus 由 [Prometheus Operator][1] 安装在与 IOMesh 相同的 Kubernetes 集群中，只需修改 `iomesh-values.yaml` 为：

```yaml
serviceMonitor:
  create: true
```

然后升级现有的 IOMesh 集群：

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
```

一个 exporter 会被创建，并由 Prometheus 自动收集指标数据。

也可以通过导入 [iomesh-prometheus-kubernetes-sd-example.yaml][4] 手动配置 Prometheus。

## 与 Grafana 集成

下载 [iomesh-dashboard.json][3] 并将其导入任何现有的 Grafana。

[1]: https://github.com/prometheus-operator/prometheus-operator
[2]: https://grafana.com/grafana/download
[3]: https://raw.githubusercontent.com/iomesh/docs/master/docs/assets/iomesh-operation/ioemsh-dashobard.json
[4]: https://raw.githubusercontent.com/iomesh/docs/master/docs/assets/iomesh-operation/iomesh-prometheus-kubernetes-sd-example.yaml

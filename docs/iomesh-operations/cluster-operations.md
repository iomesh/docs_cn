---
id: cluster-operations
title: IOMesh 集群运维
sidebar_label: IOMesh 集群运维
---

IOMesh Cluster 可以在不中断在线服务的情况下进行横向扩展和升级。

## 扩容 IOMesh 集群

### Meta Server

#### 扩容

在 `iomesh-values.yaml` 中编辑 `meta/replicaCount`。 生产环境建议配置 3~5 个 Meta Server。

例子:
```yaml
meta:
  replicaCount: 3
```

使用 kubectl 生效该配置

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
```

### Chunk Server

#### 扩容

在 `iomesh-values.yaml` 中编辑 `chunk/replicaCount`.

```yaml
chunk:
  replicaCount: 5 # <- increase this number to scale Chunk Server
```

使用 kubectl 生效该配置

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
```

## Upgrade IOMesh storage cluster

Follow the following steps to upgrade IOMesh once a new version is released.

> **_NOTE_: If you only have 1 replica of meta server or chunk server, the upgrade process will never start.**

1. Export the default config `iomesh-values.yaml` from Chart

    > **_NOTE_: If you already exported the config, you can skip this step.**

    ```bash
    helm show values iomesh/iomesh > iomesh-values.yaml
    ```

2. Edit `iomesh-values.yaml`

    ```yaml
    # The version of the IOMeshCluster. You get a new release from: http://iomesh.com/docs/release/releases
    version: v5.0.0-rc5
    ```

3. Upgrade the IOMesh Cluster

    > **_NOTE_: `iomesh` is the release name, you may modify it.**

    ```bash
    helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
    ```

4. Wait untill the new chunk server pods are ready.

    ```bash
    watch kubectl get pod --namespace iomesh-system
    ```

## Uninstall IOMesh storage cluster

> **_Attention_: 卸载 IOMesh 存储集群后，所有数据都将丢失，包括使用 IOMesh StorageClass 创建的 PVC

运行以下命令以卸载 IOMesh 集群

```bash
helm uninstall --namespace iomesh-system iomesh
```

[1]: http://www.iomesh.com/docs/installation/setup-iomesh-storage#mount-device

---
id: cluster-operations
title: IOMesh 集群运维
sidebar_label: IOMesh 集群运维
---

IOMesh Cluster 可以在不中断在线服务的情况下进行横向扩展和升级。

## 扩容 IOMesh 集群

### Chunk Server

当集群的存储空间不足或存储空间使用率较高时（超过总空间的 80%），需要扩容 chunk 集群

#### 扩容操作

在 `iomesh.yaml` 中编辑 `chunk.replicas`.

```yaml
chunk:
  replicas: 5 # <- increase this number to scale Chunk Server
```

使用 kubectl 生效该配置

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
```

检查扩容结果
```shell
kubectl get pod -n iomesh-system | grep chunk
```
```output
iomesh-chunk-0                                         2/2     Running   0          5h5m
iomesh-chunk-1                                         2/2     Running   0          5h5m
iomesh-chunk-2                                         2/2     Running   0          5h5m
iomesh-chunk-3                                         2/2     Running   0          5h5m
iomesh-chunk-4                                         2/2     Running   0          5h5m
```

### Meta Server

当 Meta 集群负载过高时，需要扩容 meta 集群

#### 扩容操作

在 `iomesh.yaml` 中编辑 `meta.replicas`。 生产环境建议配置 3~5 个 Meta Server。

例子:
```yaml
meta:
  replicas: 5
```

使用 kubectl 生效该配置

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --values iomesh-values.yaml
```

检查扩容结果
```shell
kubectl get pod -n iomesh-system | grep meta
```
```output
iomesh-meta-0                                         2/2     Running   0          5h5m
iomesh-meta-1                                         2/2     Running   0          5h5m
iomesh-meta-2                                         2/2     Running   0          5h5m
iomesh-meta-3                                         2/2     Running   0          5h5m
iomesh-meta-4                                         2/2     Running   0          5h5m
```

## IOMesh 版本升级

你可以通过如下步骤从 IOMesh v0.11.1 版本升级到 IOMesh v1.0.0 版本

> _NOTE_: 如果集群中只有 1 个 meta Pod 或只有一个 chunk Pod，则无法升级 IOMesh 版本. 

> _NOTE_: 由于 K8s CRD 升级机制的限制，从 v0.11.1 版本升级到 v1.0.0 版本的 IOMesh 集群不支持运行在 K8s v1.25 及以上版本的 K8s 集群中.

### 前置动作

#### 1. 删除默认 StorageClass
K8s 的 StorageClass 在创建后不支持修改，由于 IOMesh v1.0.0 版本的 StorageClass 字段相对于 IOMesh v0.11.1 版本发生了变动，因此升级前需要先删除默认的 StorageClass，这个操作不会影响到已经使用了该 StorageClass 的 PVC

```shell
kubectl delete sc iomesh-csi-driver
```

#### 2. 临时关闭 Webhook

已安装的 IOMesh Webhook 有可能会导致升级失败，通过如下命令临时关闭 Webhook。在升级成功后，Webhook 会被自动开启。

```shell
kubectl delete Validatingwebhookconfigurations iomesh-validating-webhook-configuration
```

### 升级动作

<!--DOCUSAURUS_CODE_TABS-->
<!--在线升级-->

#### 1. 安装新 crd
由于 Helm 在升级时不会自动安装新版本加入的 crd，因此需要先安装新 crd

```shell
kubectl apply -f https://iomesh.run/config/crd/iomesh.com_blockdevicemonitors.yaml
```

#### 2. 获取新版本新增的 values 字段

```shell
wget https://iomesh.run/config/merge-values/v1.0.0.yaml -o merge-values.yaml
```

#### 3. 升级 IOMesh 集群，保留已有 values 配置，合并新 values 配置

```bash
helm upgrade --namespace iomesh-system iomesh iomesh/iomesh --version v1.0.0 --reuse-values -f merge-values.yaml
```

#### 4. 等待所有 pod 进入 ready 状态

```bash
watch kubectl get pod --namespace iomesh-system
```

<!--离线升级-->

参考 [离线部署](../deploy/install-iomesh#离线安装-iomesh) 章节获取离线安装包

#### 1. 安装新 crd

由于 Helm 在升级时不会自动安装新版本加入的 crd，因此需要先安装新 crd
```shell
kubectl apply -f ./config/crd/iomesh.com_blockdevicemonitors.yaml
```

#### 2. 升级 IOMesh 集群，保留已有 values 配置，合并新 values 配置

```bash
./helm upgrade --namespace iomesh-system iomesh ./charts/iomesh --reuse-values -f ./config/merge-values.yaml
```

#### 3. 等待所有 pod 进入 ready 状态

```bash
watch kubectl get pod --namespace iomesh-system
```

升级完成后，可参考 [监控](../iomesh-operations/monitoring) 章节配置监控

<!--END_DOCUSAURUS_CODE_TABS-->


## Uninstall IOMesh storage cluster

> **_Attention_: 卸载 IOMesh 存储集群后，所有数据都将丢失，包括使用 IOMesh StorageClass 创建的 PVC

运行以下命令以卸载 IOMesh 集群

```bash
helm uninstall --namespace iomesh-system iomesh
```

[1]: http://www.iomesh.com/docs/installation/setup-iomesh-storage#mount-device

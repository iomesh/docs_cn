---
id: setup-iomesh
title: 设置 IOMesh
sidebar_label: 设置 IOMesh
---

节点上的块设备需要挂载到 IOMesh 集群中，IOMesh 才能利用这些设备构建和提供分布式存储服务。

默认情况下，IOMesh 不挂载任何块设备。 用户必须在安装后手动配置 IOMesh。

## 挂载块设备
### Block Device 对象

IOMesh 使用 OpenEBS 的 [node-disk-manager(NDM)](https://github.com/openebs/node-disk-manager) 来管理 Kubernetes worker 节点的磁盘。 部署 IOMesh 会在与 IOMesh 集群相同的命名空间中创建 `BlockDevice` CR：

```bash
kubectl --namespace iomesh-system -o wide get blockdevice
```

```output
NAME                                           NODENAME             PATH         FSTYPE   SIZE           CLAIMSTATE   STATUS   AGE
blockdevice-097b6628acdcd83a2fc6a5fc9c301e01   kind-control-plane   /dev/vdb1    ext4     107373116928   Unclaimed    Active   10m
blockdevice-3fa2e2cb7e49bc96f4ed09209644382e   kind-control-plane   /dev/sda              9659464192     Unclaimed    Active   10m
blockdevice-f4681681be66411f226d1b6a690270c0   kind-control-plane   /dev/sdb              1073742336     Unclaimed    Active   10m
```

使用以下命令显示块设备的详细信息:

```shell
kubectl --namespace iomesh-system -o yaml get blockdevice <device_name>
```

```output
apiVersion: openebs.io/v1alpha1
kind: BlockDevice
metadata:
  annotations:
    internal.openebs.io/uuid-scheme: gpt
  generation: 1
  labels:
    iomesh.com/bd-devicePath: dev.sda
    iomesh.com/bd-deviceType: disk
    iomesh.com/bd-driverType: SSD
    iomesh.com/bd-serial: 24da000347e1e4a9
    iomesh.com/bd-vendor: ATA
    kubernetes.io/hostname: kind-control-plane
    ndm.io/blockdevice-type: blockdevice
    ndm.io/managed: "true"
  namespace: iomesh-system
  name: blockdevice-3fa2e2cb7e49bc96f4ed09209644382e
# ...
```

以 `iomesh.com/bd-` 开头的标签是由 IOMesh 创建的，用于描述硬件属性:

| Name | Describe |
| --- | --- |
| `iomesh.com/bd-devicePath` | the device path on node |
| `iomesh.com/bd-deviceType` | disk, loop, partition etc. |
| `iomesh.com/bd-driverType` | HDD, SSD, NVME |
| `iomesh.com/bd-serial` | disk serial |
| `iomesh.com/bd-vendor` | disk vendor |

### DeviceMap

`iomesh-values.yaml` 中的 `chunk.deviceMap` 字段用于配置将哪些块设备挂载到 IOMesh 集群中，以及他们的挂载类型

```yaml
spec:
  chunk:
    deviceMap:
      <mount-type>:
        selector:
          matchLabels:
            <label-key>: <label-value>
          matchExpressions:
          - key: <label-key>
            operator: In
            Values:
            - <label-value>
        exclude:
        - <block-device-name>
```

#### 挂载类型
在`hybrid-flash` 部署模式下，IOMesh 提供了两种挂载类型：

- `cacheWithJournal`：用于存储池的性能层。 它必须是一个可分区的块设备。 将创建两个分区：一个用于“journal”，另一个用于“cache”。 建议使用 `SATA` 或 `NVMe` SSD。
- `dataStore`：用于存储池的容量层。 建议使用 `SATA` 或 `SAS` HDD。

在 `all-flash` 部署模式下，IOMesh 只提供一种挂载类型：

- `dataStoreWithJournal`：用于存储池的容量层。 它必须是一个可分区的块设备。 将创建两个分区：一个用于“journal”，另一个用于“dataStore”。 建议使用 `SATA` 或 `NVMe` SSD。

#### Device Selector

Device Selector 的定义如下:

| Name     | Type                                                         | Explain                                                      |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| selector | [metav1.LabelSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#labelselector-v1-meta) | `BlockDevice` 的 label selector |
| exclude  | []string                                                     | 被排除的 `BlockDevice` 列表 |

Device Selector 筛选的所有块设备将以对应的挂载类型挂载到 IOMesh 中

#### Hybrid-Flash 部署模式下的 DeviceMap 配置示例
```yaml
spec:
  # ...
  chunk:
    # ...
    deviceMap:
      cacheWithJournal:
        selector:
          matchLabels:
            iomesh.com/bd-deviceType: disk
          matchExpressions:
          - key: iomesh.com/bd-driverType
            operator: In
            values:
            - SSD
        exclude:
        - blockdevice-097b6628acdcd83a2fc6a5fc9c301e01
      dataStore:
        selector:
          matchExpressions:
          - key: iomesh.com/bd-driverType
            operator: In
            values:
            - HDD
        exclude:
        - blockdevice-097b6628acdcd83a2fc6a5fc9c301e01
    # ...
```

#### All-Flash 部署模式下的 DeviceMap 配置示例
```yaml
spec:
  # ...
  chunk:
    # ...
    deviceMap:
      dataStoreWithJournal:
        selector:
          matchLabels:
            iomesh.com/bd-deviceType: disk
          matchExpressions:
          - key: iomesh.com/bd-driverType
            operator: In
            values:
            - SSD
        exclude:
        - blockdevice-097b6628acdcd83a2fc6a5fc9c301e01
    # ...
```
> **_NOTE_: IOMesh 使用的块设备不应包含任何现有的文件系统。 请确保命令 `kubectl --namespace iomesh-system -owide get blockdevice` 输出的字段 `FSTYPE` 为空.**

获得正确的 deviceMap 配置后，通过运行 `kubectl edit --namespace iomesh-system iomesh` 将其设置为 `spec.chunk.deviceMap`，然后运行 `kubectl --namespace iomesh-system -o wide get blockdevice` 来验证我们筛选的 `BlockDevice` 的状态变为 `Claimed`

```bash
kubectl --namespace iomesh-system -o wide get blockdevice
```

```output
NAME                                           NODENAME             PATH         FSTYPE   SIZE           CLAIMSTATE   STATUS   AGE
blockdevice-097b6628acdcd83a2fc6a5fc9c301e01   kind-control-plane   /dev/vdb1    ext4     107373116928   Unclaimed    Active   11m
blockdevice-3fa2e2cb7e49bc96f4ed09209644382e   kind-control-plane   /dev/sda              9659464192     Claimed      Active   11m
blockdevice-f4681681be66411f226d1b6a690270c0   kind-control-plane   /dev/sdb              1073742336     Claimed      Active   11m
```

---
id: setup-storageclass
title: 创建 StorageClass
sidebar_label: 创建 StorageClass
---

IOMesh storage class 支持配置如下参数:

| Parameter                 | Value                         | Default | Description                        |
| ------------------------- | ----------------------------- | ------- | ---------------------------------- |
| csi.storage.k8s.io/fstype | "xfs", "ext2", "ext3", "ext4" | "ext4"  | 卷文件系统类型            |
| replicaFactor             | "2", "3"                      | "2"     | 副本数                     |
| thinProvision             | "true", "false"               | "true"  | 精简置备或厚制备      |

安装 IOMesh 后，将创建一个默认的 StorageClass `iomesh-csi-driver`。 您还可以使用自定义参数创建一个新的 StorageClass：

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: iomesh-csi-driver-default
provisioner: com.iomesh.csi-driver # <-- driver.name in iomesh-values.yaml
reclaimPolicy: Retain
allowVolumeExpansion: true
parameters:
  # "ext4" / "ext3" / "ext2" / "xfs"
  csi.storage.k8s.io/fstype: "ext4"
  # "2" / "3"
  replicaFactor: "2"
  # "true" / "false"
  thinProvision: "true"
volumeBindingMode: Immediate
```
> _关于`reclaimPolicy`_
>
> `StorageClass`的`reclaimPolicy`属性可以有`Retain`和`Delete`两个值，默认为`Delete`。 通过 `StorageClass` 创建 `PV` 时，其 `persistentVolumeReclaimPolicy` 属性将继承 `StorageClass` 的 `reclaimpolicy` 属性。 您也可以手动修改 `persistentVolumeReclaimPolicy` 的值。
>
> 例子中的`reclaimPolicy`的值为`Retain`，也就是说，如果删除一个`PVC`，则该`PVC`下的`PV`不会被删除，而是进入`Released`状态。 请注意，如果删除`PV`，对应的IOMesh卷不会被删除，而是需要将`PV`的`persistentVolumeReclaimPolicy`的值改为`Delete`，然后删除`PV`。 或者在创建`PV`之前，可以将`StorageClass`的`reclaimpolicy`的值设置为`Delete`，这样所有的资源都会被级联释放。

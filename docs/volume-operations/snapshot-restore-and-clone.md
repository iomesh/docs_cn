---
id: snapshot-restore-and-clone
title: 快照/恢复/克隆
sidebar_label: 快照/恢复/克隆
---

## 快照

用户可以使用 IOMesh 为现有的 PV 创建快照。

VolumeSnapshot 对象定义了创建 PVC 快照的请求。

例如：
```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: example-snapshot
spec:
  volumeSnapshotClassName: iomesh-csi-driver-default
  source:
    persistentVolumeClaimName: mongodb-data-pvc # PVC name that want to take snapshot
```


生效 yaml 配置
```text
kubectl apply -f example-snapshot.yaml
```

创建 VolumeSnapshot 对象后，IOMesh 会创建一个对应的 VolumeSnapshotContent 对象。

```bash
kubectl get Volumesnapshots example-snapshot
```

```output
NAME               SOURCEPVC            RESTORESIZE    SNAPSHOTCONTENT                                    CREATIONTIME
example-snapshot   mongodb-data-pvc     6Gi            snapcontent-fb64d696-725b-4f1b-9847-c95e25b68b13   10h
```

## 快照回滚

用户可以通过创建一个 PVC 来恢复卷快照，其中 `dataSource` 字段引用快照。

例如：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-restore
spec:
  storageClassName: iomesh-csi-driver-default
  dataSource:
    name: example-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 6Gi
```

生效 yaml 配置

```bash
kubectl apply -f example-restore.yaml
```

## 克隆

用户可以通过创建 PVC，同时在 `spec.dataSource` 字段配置同一命名空间中现有 PVC 来克隆持久卷 (PV)。

例如：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: iomesh-csi-driver-default
  dataSource:
    name: existing-pvc # an existing PVC in the same namespace
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeMode: Block
```

生效 yaml 配置

```bash
kubectl apply -f example-clone.yaml
```

克隆操作有一些限制：

1. 克隆的 PVC 必须与原始 PVC 存在于相同的命名空间中。
2. 两个 PVC 必须具有相同的 StorageClass 和 VolumeMode 设置。

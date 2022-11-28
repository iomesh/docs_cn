---
id: setup-snapshotclass
title: 创建 SnapshotClass
sidebar_label: 创建 SnapshotClass
---

[Kubernetes VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/) 类似于 StorageClass。它有助于定义多个存储策略。每个 VolumeSnap 都与一个 VolumeSnapshotClass 相关联

使用以下配置创建 Volume Snapshot Class:

```yaml
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: iomesh-csi-driver-default
driver: com.iomesh.csi-driver  # <-- driver.name in iomesh-values.yaml
deletionPolicy: Retain
```

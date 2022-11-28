---
id: create-volume
title: 创建 volume
sidebar_label: 创建 volume
---

可以使用以下 YAML 文件创建卷。 用户应确保对应的 `StorageClass` 已经存在

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iomesh-example-pvc
spec:
  storageClassName: iomesh-example-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

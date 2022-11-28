---
id: expand-volume
title: 扩容 volume
sidebar_label: 扩容 volume
---

IOMesh 卷在创建后可以扩容，无论它们是否正在使用。

在以下示例中，假设有一个名为 example-pvc 的 PVC，其容量为 10Gi：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  storageClassName: iomesh-csi-driver-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi # original capacity
```

生效 yaml 配置

```bash
kubectl get pvc example-pvc
```

```output
NAME          STATUS   VOLUME                                     CAPACITY    ACCESS MODES   STORAGECLASS                AGE
example-pvc   Bound    pvc-b2fc8425-9dbc-4204-8240-41cb4a7fa8ca   10Gi        RWO            iomesh-csi-driver-default   11m
```

要将此 PVC 的容量扩展到 20Gi，只需修改 PVC 声明：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  storageClassName: iomesh-csi-driver-default
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi # now expand capacity from 10 Gi to 20Gi
```

生效 yaml 配置

```bash
kubectl apply -f example-pvc.yaml
```

检查扩容结果

```bash
kubectl get pvc example-pvc
```

```output
NAME          STATUS   VOLUME                                     CAPACITY    ACCESS MODES   STORAGECLASS                AGE
example-pvc   Bound    pvc-b2fc8425-9dbc-4204-8240-41cb4a7fa8ca   20Gi        RWO            iomesh-csi-driver-default   11m
```

```bash
kubectl get pv pvc-b2fc8425-9dbc-4204-8240-41cb4a7fa8ca
```

```output
NAME                                       CAPACITY   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS
pvc-b2fc8425-9dbc-4204-8240-41cb4a7fa8ca   20Gi       Retain           Bound    default/example-pvc   iomesh-csi-driver-default
```

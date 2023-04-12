---
id: hostpath-localpv
title: IOMesh Hostpath LocalPV
sidebar_label: IOMesh Hostpath LocalPV
---
### 概述
IOMesh Hostpath LocalPV 支持基于节点上的一个目录创建 Kubernetes PV 提供给 Pod 使用，并且支持 PV 级别的容量限额

### 创建 IOMesh Hostpath LocalPV

#### 配置 StorageClass

在 IOMesh LocalPV Manager 安装完成后，会默认创建一个`volumeType` 为 `hostpath` 类型的 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh-localpv-manager-hostpath
parameters:
  volumeType: hostpath
  basePath: /var/iomesh/local
  enableQuota: "false"
provisioner: com.iomesh.iomesh-localpv-manager
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

主要的参数释义如下：

| 参数名称               | 参数释义                                                     |
| ---------------------- | ------------------------------------------------------------ |
| parameters.volumeType  | localpv 类型，支持 `hostpath` 或 `device`， 该参数为必选参数                    |
| parameters.basePath    | localpv 将被创建在节点上 parameters.basePath 声明的目录中，如果 parameters.basePath 不存在会被自动创建。**该参数为必选参数且parameters.basePath 的格式必须是绝对路径** |
| parameters.enableQuota | 是否开启目录限额，该参数为可选参数，默认关闭                                       |
| volumeBindingMode      | pvc 绑定模式，仅支持 `WaitForFirstConsumer`              |

用户可以根据需要自身需求创建自定义的 StorageClass，比如改变 basePath 路径或开启容量限额

#### 创建 PVC

1. 准备 PVC 配置，使用默认的 StorageClass 配置

```yaml
# iomesh-localpv-hostpath-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: iomesh-localpv-hostpath-pvc
spec:
  storageClassName: iomesh-localpv-manager-hostpath
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

2. 创建 PVC

```shell
kubectl apply -f iomesh-localpv-hostpath-pvc.yaml
```

3. 查看 PVC 状态

```shell
kubectl get pvc iomesh-localpv-hostpath-pvc
```

可以看到 PVC 处于 Pending 状态，这是由于 StorageClass 的 `volumeBindingMode` 配置为 `WaitForFirstConsumer` ，只有当 Pod 绑定该 PVC 后，对应的 PV 才会在 Pod 调度到的 Node 上创建，PVC 才会进入 Bound 状态

```shell
NAME                          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS               AGE
iomesh-localpv-hostpath-pvc   Pending                                      localpv-manager-hostpath   1m12s
```

#### 创建 Pod 绑定 PVC

1. 准备 Pod 配置，将 PVC 挂载到 Pod 内的 `/mnt/iomesh/localpv` 目录

```yaml
# iomesh-localpv-hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: iomesh-localpv-hostpath-pod
spec:
  volumes:
  - name: iomesh-localpv-hostpath
    persistentVolumeClaim:
      claimName: iomesh-localpv-hostpath-pvc
  containers:
  - name: busybox
    image: busybox
    command:
       - sh
       - -c
       - 'while true; do sleep 30; done;'
    volumeMounts:
    - mountPath: /mnt/iomesh/localpv
      name: iomesh-localpv-hostpath
```

2. 创建 Pod

```shell
kubectl apply -f iomesh-localpv-hostpath-pod.yaml
```

3. 确认 Pod 进入 Running 状态

```shell
kubectl get pod iomesh-localpv-hostpath-pod
```

4. 再次查看 PVC 状态

```shell
kubectl get pvc iomesh-localpv-hostpath-pvc
```

可以看到对应的 PV 已自动创建，并且 PVC 进入 Bound 状态

```shell
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES     
iomesh-localpv-hostpath-pvc   Bound    pvc-ab61547e-1d81-4086-b4e4-632a08c6537b   2G         RWO           
```

5. 查看 PV 的配置，可以得到更详细的信息

```shell
kubeclt get pv pvc-ab61547e-1d81-4086-b4e4-632a08c6537b -o yaml
```

输出如下

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
...
spec:
...
  csi:
    driver: com.iomesh.iomesh-localpv-manager
    volumeAttributes:
      basePath: /var/iomesh/local
      csi.storage.k8s.io/pv/name: pvc-ab61547e-1d81-4086-b4e4-632a08c6537b
      csi.storage.k8s.io/pvc/name: iomesh-localpv-hostpath-pvc
      csi.storage.k8s.io/pvc/namespace: default
      enableQuota: "false"
      nodeID: iomesh-k8s-0
      quotaProjectID: "0"
      storage.kubernetes.io/csiProvisionerIdentity: 1675327643130-8081-com.iomesh.localpv-manager
      volumeType: hostpath
    volumeHandle: pvc-ab61547e-1d81-4086-b4e4-632a08c6537b
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.iomesh-localpv-manager.csi/node
          operator: In
          values:
          - iomesh-k8s-0
...
```

其中存在如下关键字段：

| 字段名称                           | 字段释义                                                     |
| ---------------------------------- | ------------------------------------------------------------ |
| spec.csi.volumeAttributes.basePath | localpv 创建的 basePath，该 PV 对应的目录被创建在 `iomesh-k8s-0` 这个节点的 `/var/iomesh/local/pvc-ab61547e-1d81-4086-b4e4-632a08c6537b` 路径 |
| spec.nodeAffinity                  | PV 的节点亲和性。该 PV 被创建后就与节点绑定，不会漂移到其他节点 |



## 容量限额

### 概述

在上述例子中，我们创建了一个 2G 的 iomesh localpv，它对应节点上的一个目录。在默认情况下 iomesh localpv 并不限制 Pod 对该目录的写入数据量，允许 Pod 写入超过 2G 的数据。如果用户有严格的容量隔离需求，可以开启 IOMesh LocalPV Manager 的容量限额功能。

IOMesh LocalPV Manager 的容量限额功能基于 xfs 文件系统的 xfs_quota 特性实现，当限额功能开启后， localpv 在被创建时会同时配置 xfs quota，以确保 PVC 中声明的容量大小是真实生效的

### 开启容量限额

容量限额功能要求 StorageClass 中 parameters.basePath 指向的路径是一个 xfs 格式挂载点，并通过挂载选项开启了 xfs 的 prjquota 特性。在 K8s worker 上创建该挂载点的步骤如下:

1. 假设 basePath 为 /var/iomesh/localpv-quota, 创建对应目录

```shell
mkdir -p /var/iomesh/localpv-quota
```

2. 将待挂载的磁盘（假设为 /dev/sdx）格式化为 xfs 文件系统

```shell
sudo mkfs.xfs /dev/sdx
```

3. 挂载磁盘到 basePath（假设 basePath 为 /var/iomesh/localpv-quota），通过挂载选项开启 xfs 的  prjquota 特性

```shell
mount -o prjquota /dev/sdx /var/iomesh/localpv-quota
```

> _NOTE_: 如果希望使用一个已存在的 xfs 挂载点作为 basePath，则需要先 umount 该挂载点，然后重新用 `prjquota` 挂载选项再次挂载。（xfs的 `prjquota` 挂
载选项不支持在已有挂载点上直接 remount）

> _NOTE_: 为了防止节点重启后挂载信息丢失，需要将挂载信息持久化到 /etc/fstab 配置文件中

3. 使用该挂载点作为  basePath 创建 StorageClass，并将 `parameters.enableQuota` 字段设为 "true"

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh-localpv-manager-hostpath
parameters:
  volumeType: hostpath
  basePath: /var/iomesh/localpv-quota
  enableQuota: "true"
provisioner: com.iomesh.iomesh-localpv-manager
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

4. 接着就可以使用该 StorageClass 创建 PVC，步骤与上一章节描述相同

> _NOTE_: 当前版本的 IOMesh LocalPV Manager 尚未支持调度器扩展（见[调度器扩展](scheduler)章节）, Pod 有可能被调度到容量限额不足的节点上，此时需要手动修改 Pod 的节点亲和性使其重新调度到容量限额满足需求的节点上
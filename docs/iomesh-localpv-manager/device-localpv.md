---
id: device-localpv
title: IOMesh Device LocalPV
sidebar_label: IOMesh Device LocalPV
---

### 概述
IOMesh Hostpath LocalPV 支持基于节点上的一个块设备创建 Kubernetes PV 提供给 Pod 使用

### 创建 IOMesh Device LocalPV

#### 配置 StorageClass

在 IOMesh LocalPV Manager 安装完成后，会创建一个默认的 device 类型 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh-localpv-manager-device
parameters:
  volumeType: device
  csi.storage.k8s.io/fstype: "ext4"
  deviceSelector: {}
provisioner: com.iomesh.iomesh-localpv-manager
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

主要的参数释义如下：

| 参数名称                  | 参数释义                                               |
| ------------------------- | ------------------------------------------------------ |
| parameters.volumeType     | localpv 类型，支持 `hostpath` 或 `device`，该参数为必选参数             |
| parameters.deviceSelector | device 选择器，通过 label 来筛选 blockdevice，该参数为可选参数，如果未指定则默认筛选所有 label           |
| parameters.  csi.storage.k8s.io/fstype         | pvc 的 `volumeMode` 为 `Filesystem` 时格式化的文件系统类型，该参数为可选参数，如果未指定默认为 ext4 |
| volumeBindingMode         | localpv 绑定模式，仅支持 `WaitForFirstConsumer`        |

#### 设备选择器

用户可以在 `parameters.deviceSelector` 字段中使用 [K8s 的标准配置语法](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) 来筛选 blockdevice，目前仅支持 `matchLabels` 的配置方式。

关于什么是 blockdevice 对象以及如何查看 blockdevice 已有的 label 参考 [setup-iomesh](https://docs.iomesh.com/deploy/setup-iomesh) 中的 block-device-object 章节

例如，下述的 StorageClass 只会筛选 SSD 设备用于创建 device 类型的 localpv

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh-localpv-manager-device-ssd
parameters:
  volumeType: device
  deviceSelector: |
    matchLabels:
      iomesh.com/bd-driveType: SSD
provisioner: com.iomesh.iomesh-localpv-manager
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

> _NOTE_: K8s 中的 StorageClass 中 `parameters` 字段仅支持一级的 key/value 嵌套，因此在配置时需要注意在 `parameters.deviceSelector` 字段后加上 `|` 来标识后续内容是多行字符串

#### 创建 PVC

> _NOTE_: 在之后的例子中，为了保证 PVC 能够成功创建，请确保每个 K8s worker 至少有一块大于 10G 的空闲裸块设备（不存在任何文件系统）

1. 准备 PVC 配置，使用默认的 StorageClass

```yaml
# iomesh-localpv-device-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: iomesh-localpv-device-pvc
spec:
  storageClassName: iomesh-localpv-manager-device
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeMode: Filesystem
```

PVC 的 `volumeMode` 为 `Filesystem`，这意味着当 Pod 绑定该 PVC 后，对应的块设备会被格式化为在 StorageClass 的 `parameters.fsType` 字段指定的文件系统，再挂载到 Pod 内部。除了 `Filesystem` 外，IOMesh Device LocalPV 还支持挂载裸设备，只需要将 PVC 的 `volumeMode` 为 `Block`，示例如下

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: iomesh-localpv-device-block-pvc
spec:
  storageClassName: iomesh-localpv-manager-device
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  volumeMode: Block
```

2. 创建 PVC

```shell
kubectl apply -f iomesh-localpv-device-pvc.yaml
```

3. 查看 PVC 状态

```shell
kubectl get pvc iomesh-localpv-device-pvc
```

可以看到 PVC 处于 Pending 状态，这是由于 StorageClass 的 `volumeBindingMode` 配置为 `WaitForFirstConsumer` ，只有当 Pod 绑定该 PVC 后，对应的 PV 才会在 Pod 调度到的 Node 上创建，PVC 才会进入 Bound 状态

```shell
NAME                          STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS               AGE
iomesh-localpv-device-pvc     Pending                                      localpv-manager-device     1m12s
```

#### 创建 Pod 绑定 PVC

1. 准备 Pod 配置，绑定刚刚创建的 PVC 并将其挂载到 Pod 内的 `/mnt/iomesh/localpv` 目录

```yaml
# iomesh-localpv-device-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: iomesh-localpv-device-pod
spec:
  volumes:
  - name: iomesh-localpv-device
    persistentVolumeClaim:
      claimName: iomesh-localpv-device-pvc
  containers:
  - name: busybox
    image: busybox
    command:
       - sh
       - -c
       - 'while true; do sleep 30; done;'
    volumeMounts:
    - mountPath: /mnt/iomesh/localpv
      name: iomesh-localpv-device
```

2. 创建 Pod

```shell
kubectl apply -f iomesh-localpv-device-pod.yaml
```

3. 确认 Pod 进入 Running 状态

```shell
kubectl get pod iomesh-localpv-device-pod
```
> _NOTE_: 当前版本的 IOMesh LocalPV Manager 尚未支持调度器扩展（见[调度器扩展](scheduler)章节）, Pod 有可能被调度到没有适合的块设备的节点上，此时需要手动修改 Pod 的节点亲和性使其重新调度到块设备满足需求的节点上

4. 再次查看 PVC 状态

```shell
kubectl get pvc iomesh-localpv-device-pvc
```

可以看到对应的 PV 已自动创建，并且 PVC 进入 Bound 状态

```shell
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES     
iomesh-localpv-device-pvc     Bound    pvc-72f7a6ab-a9c4-4303-b9ba-683d7d9367d4   10G         RWO           
```

5. 查看 BlockDevice 状态

```shell
kubectl  get blockdevice --namespace iomesh-system -o wide
```

可以看到对应的块设备已进入 bound 状态。IOMesh LocalPV Manager 会选择大于且最接近与 PVC 大小的块设备进行绑定，比如在这个例子是，PVC 声明的大小是 10GB，假设存在三个空闲块设备 BlockDevice-A(9GB)，BlockDevice-B(15GB)，BlockDevice-C(20GB)，BlockDevice-A 由于不满足容量需求被淘汰，BlockDevice-B 比 BlockDevice-C 的大小更接近 10GB，最终 BlockDevice-B 被绑定

6. 查看 PV 的配置，可以得到更详细的信息

```shell
kubeclt get pv pvc-72f7a6ab-a9c4-4303-b9ba-683d7d9367d4 -o yaml
```

```yaml
# output
apiVersion: v1
kind: PersistentVolume
metadata:
...
spec:
...
  csi:
    driver: com.iomesh.iomesh-localpv-manager
    volumeAttributes:
      csi.storage.k8s.io/pv/name: pvc-72f7a6ab-a9c4-4303-b9ba-683d7d9367d4
      csi.storage.k8s.io/pvc/name: iomesh-localpv-device-pvc
      csi.storage.k8s.io/pvc/namespace: default
      nodeID: ziyin-k8s-2
      quotaProjectID: "0"
      storage.kubernetes.io/csiProvisionerIdentity: 1675327673777-8081-com.iomesh.localpv-manager
      volumeType: device
    volumeHandle: pvc-72f7a6ab-a9c4-4303-b9ba-683d7d9367d4
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.iomesh-localpv-manager.csi/node
          operator: In
          values:
          - ziyin-k8s-2
...
```

其中存在如下关键字段：

| 字段名称                               | 字段释义                                                     |
| -------------------------------------- | ------------------------------------------------------------ |
| spec.csi.volumeAttributes.volumeHandle | BlockDeviceClaim 的唯一标识                                  |
| spec.nodeAffinity                      | PV 的节点亲和性。该 PV 被创建后就与节点绑定，不会漂移到其他节点 |

 

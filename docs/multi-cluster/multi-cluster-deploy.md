---
id: multi-cluster-deploy
title: IOMesh 多集群部署
sidebar_label: IOMesh 多集群部署
---

IOMesh 支持在一个 K8s 集群中部署多个 IOMesh 集群，每个 IOMesh 集群相当于一个独立的存储池。

一个 K8s 集群中的所有 IOMesh 集群会共用大部分管控组件（主要包括 operator，csi-driver，node-disk-manager），这样做的好处是可以大幅减少集群中 Pod 的数量，降低资源消耗。这些管控组件会在 IOMesh 首次部署时安装，后续将 K8s 集群中部署的第一套 IOMesh 集群称为管控 IOMesh 集群

> _Note:_
> IOMesh 暂时不支持将已存在的单 IOMesh 集群拓展成多 IOMesh 集群

## 支持该功能的 IOMesh 版本

v1.0.0+

## 部署方式

### 环境准备

假设存在 6 个 k8s woker 节点，主机名为 k8s-woker-{0~5}。

1. 在 k8s-woker-{0~2} 中部署第一套 IOMesh 集群 "iomesh"，namspace 为 iomesh-system
2. 在 k8s-woker-{3~5} 中部署第二套 IOMesh 集群 "iomesh-cluster-1"，namespace 为 iomesh-cluster-1

### 部署步骤

#### 部署第一套 IOMesh 集群 "iomesh"

1. 完成 [安装准备](https://docs.iomesh.com/deploy/prerequisites) 章节中相关 open-iscsi 等前置配置

2. 生成 helm values 配置文件

   ```shell
   helm show values iomesh/iomesh > iomesh-values.yaml
   ```

3. 配置 `iomesh-values.yaml`

   将 `iomesh.chunk.dataCIDR` 字段配置为存储网段的 CIDR
   ```yaml
   # Source: iomesh-values.yaml
   iomesh:
     chunk:
       dataCIDR: <your-data-cidr-here>
   ```

   配置 meta/chunk/redirector 三个组件的 node 亲和性让其调度到 k8s-woker-{0~2} ，对应的字段分别为：
   `iomesh.meta.podPolicy/iomesh.chunk.podPolicy/iomesh.redirector.podPolicy`

   在这三个字段中均增加如下配置

   ```yaml
   # Source: iomesh-values.yaml
   ...
   iomesh:
   ...
     chunk:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname  # any kubernete node label key
                   operator: In
                   values:  # kubernete node label key's value
                   - k8s-woker-0
                   - k8s-woker-1
                   - k8s-woker-2
    ...
     meta:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname  # any kubernete node label key
                   operator: In
                   values:  # kubernete node label key's value
                   - k8s-woker-0
                   - k8s-woker-1
                   - k8s-woker-2
    ...
     redirector:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname  # any kubernete node label key
                   operator: In
                   values:  # kubernete node label key's value
                   - k8s-woker-0
                   - k8s-woker-1
                   - k8s-woker-2
   ```

   配置 zookeeper 的 node 亲和性让其调度到 k8s-woker-{0~2} 并增加 pod 反亲和性，在 `iomesh.zookeeper.podPolicy` 中如下配置

   ```yaml
   # Source: iomesh-values.yaml
   ...
   iomesh:
   ...
     zookeeper:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname  # any kubernete node label key
                   operator: In
                   values:  # kubernete node label key's value
                   - k8s-woker-0
                   - k8s-woker-1
                   - k8s-woker-2
           podAntiAffinity:
             preferredDuringSchedulingIgnoredDuringExecution:
             - podAffinityTerm:
                 labelSelector:
                   matchExpressions:
                   - key: app
                     operator: In
                     values:
                     - iomesh-zookeeper
                 topologyKey: kubernetes.io/hostname
               weight: 20
   ```
   
   执行部署
   ```shell
   helm install iomesh iomesh/iomesh --create-namespace  --namespace iomesh-system  --values iomesh-values.yaml
   ```
   ```output
   NAME                                                  READY   STATUS           RESTARTS   AGE
   iomesh-blockdevice-monitor-766655959f-7bwvv           1/1     Running          0          5h37m
   iomesh-blockdevice-monitor-prober-pgbcp               1/1     Running          0          5h36m
   iomesh-blockdevice-monitor-prober-v848k               1/1     Running          0          5h37m
   iomesh-blockdevice-monitor-prober-w6vrw               1/1     Running          0          5h36m
   iomesh-chunk-0                                        3/3     Running          0          5h34m
   iomesh-chunk-1                                        3/3     Running          0          5h33m
   iomesh-chunk-2                                        3/3     Running          0          5h33m
   iomesh-csi-driver-controller-plugin-ccf6ccfd9-kf296   6/6     Running          3          30h
   iomesh-csi-driver-node-plugin-c8mrj                   3/3     Running          1          30h
   iomesh-csi-driver-node-plugin-jncfr                   3/3     Running          1          30h
   iomesh-csi-driver-node-plugin-qrvc6                   3/3     Running          1          30h
   iomesh-csi-driver-node-plugin-5rvc6                   2/3     CrashLoopBackOff 1          30h
   iomesh-csi-driver-node-plugin-qr4c6                   2/3     CrashLoopBackOff 1          30h
   iomesh-csi-driver-node-plugin-q3vc6                   2/3     CrashLoopBackOff 1          30h
   iomesh-hostpath-provisioner-vdmrz                     1/1     Running          0          31h
   iomesh-hostpath-provisioner-wtbfh                     1/1     Running          0          31h
   iomesh-hostpath-provisioner-xlkvb                     1/1     Running          0          31h
   iomesh-iscsi-redirector-9c455                         2/2     Running          1          5h36m
   iomesh-iscsi-redirector-k7jbx                         2/2     Running          3          5h35m
   iomesh-iscsi-redirector-l9w7l                         2/2     Running          1          5h34m
   iomesh-localpv-manager-82564                          4/4     Running          2          29h
   iomesh-localpv-manager-pkcbd                          4/4     Running          0          12h
   iomesh-localpv-manager-px4cl                          4/4     Running          0          29h
   iomesh-meta-0                                         2/2     Running          0          5h34m
   iomesh-meta-1                                         2/2     Running          0          5h35m
   iomesh-meta-2                                         2/2     Running          0          5h34m
   iomesh-openebs-ndm-cluster-exporter-68c757948-qmtt4   1/1     Running          0          31h
   iomesh-openebs-ndm-gnkzt                              1/1     Running          0          31h
   iomesh-openebs-ndm-lrww8                              1/1     Running          0          31h
   iomesh-openebs-ndm-node-exporter-rp4qr                1/1     Running          0          31h
   iomesh-openebs-ndm-node-exporter-sgzpp                1/1     Running          0          31h
   iomesh-openebs-ndm-node-exporter-txbn5                1/1     Running          0          31h
   iomesh-openebs-ndm-operator-584fdbcb94-2tjsp          1/1     Running          0          31h
   iomesh-openebs-ndm-zdngg                              1/1     Running          0          31h
   iomesh-zookeeper-0                                    1/1     Running          0          31h
   iomesh-zookeeper-operator-64957fcc4f-zpdjp            1/1     Running          0          30h
   operator-f5644b7f9-2vvw7                              1/1     Running          0          5h37m
   ```

> _Note:_
> k8s-woker-{3~5} 上的 iomesh-csi-driver-node-plugin pod 会一直处于 CrashLoopBackOff ，直到第二套集群部署好


#### 部署第二套 IOMesh 集群 "iomesh-cluster-1"

1. 创建 iomesh-cluster-1 集群 namespace

   ```shell
   kubectl create ns iomesh-cluster-1
   ```

2. 创建 iomesh-cluster-1 的 zk 集群，配置 node 亲和性和 pod 反亲和性让其调度到 k8s-woker-{3~5}，`kubectl apply -f iomesh-cluster-1-zookeeper.yaml`

   ```yaml
   # Source: iomesh-cluster-1-zookeeper.yaml
   apiVersion: zookeeper.pravega.io/v1beta1
   kind: ZookeeperCluster
   metadata:
     namespace: iomesh-cluster-1
     name: iomesh-cluster-1-zookeeper
   spec:
     replicas: 3
     image:
       repository: iomesh/zookeeper
       tag: 3.5.9
       pullPolicy: IfNotPresent
     pod:
       affinity:
         nodeAffinity:
           requiredDuringSchedulingIgnoredDuringExecution:
             nodeSelectorTerms:
             - matchExpressions:
	       - key: kubernetes.io/hostname  # any kubernete node label key
	         operator: In
	         values:  # kubernete node label key's value
                   - k8s-woker-3
                   - k8s-woker-4
                   - k8s-woker-5
         podAntiAffinity:
           preferredDuringSchedulingIgnoredDuringExecution:
           - podAffinityTerm:
               labelSelector:
                 matchExpressions:
                 - key: app
                   operator: In
                   values:
                   - iomesh-cluster-1-zookeeper
               topologyKey: kubernetes.io/hostname
             weight: 20
       securityContext:
         runAsUser: 0
     persistence:
       reclaimPolicy: Delete
       spec:
         storageClassName: hostpath
         resources:
           requests:
             storage: 20Gi
   ```
   
2. 部署 iomesh 集群 `iomesh-cluster-1`，配置 node 亲和性让其调度到 k8s-woker-{3~5}，另外需要注意 `iomesh-cluster-1.yaml` 中的注释部分，做如下配置

   a (必须). 并将 chunk 和 redirector 中的 `dataCIDR` 字段配置为集群实际的存储网段
   
   b (必须). 将 `spec.chunk.devicemanager.blockDeviceNamespace` 配置为 iomesh-system (部署第一套 IOMesh 时同时部署了其他管控组件到 iomesh-system namesapce 中，所有的 blockDevice 均在 iomesh-system 中统一管理)

   c (可选). `iomesh-cluster-1.yaml` 配置文件默认会安装 IOMesh 社区版，如需安装企业版，则需要将配置文件中的 meta/chunk/redirector 中的 `image.repository.tag` 配置为：`v5.3.0-rc13-enterprise`
   
   配置完成后执行部署： `kubectl apply -f iomesh-cluster-1.yaml`
   
   
   ```yaml
   # Source: iomesh-cluster-1.yaml
   apiVersion: iomesh.com/v1alpha1
   kind: IOMeshCluster
   metadata:
     namespace: iomesh-cluster-1
     name: iomesh-cluster-1
   spec:
     diskDeploymentMode: hybridFlash
     storageClass: hostpath
     reclaimPolicy:
       volume: Delete
       blockdevice: Delete
     meta:
       replicas: 3
       image:
         repository: iomesh/zbs-metad
         tag: v5.3.0-rc13
         pullPolicy: IfNotPresent
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
	         - key: kubernetes.io/hostname  # any kubernete node label key
	           operator: In
	           values:  # kubernete node label key's value
                   - k8s-woker-3
                   - k8s-woker-4
                   - k8s-woker-5
     chunk:
       dataCIDR: <your-data-cidr-here>  # filled data cidr here
       replicas: 3
       image:
         repository: iomesh/zbs-chunkd
         tag: v5.3.0-rc13
         pullPolicy: IfNotPresent
       devicemanager:
         image:
           repository: iomesh/operator-devicemanager
           tag: v1.0.0
           pullPolicy: IfNotPresent
         blockDeviceNamespace: iomesh-system  # filled first iomesh cluster namespace here
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
	         - key: kubernetes.io/hostname  # any kubernete node label key
	           operator: In
	           values:  # kubernete node label key's value
                   - k8s-woker-3
                   - k8s-woker-4
                   - k8s-woker-5    
     redirector:
       dataCIDR: <your-data-cidr-here>  # filled data cidr here
       image:
         repository: iomesh/zbs-iscsi-redirectord
         tag: v5.3.0-rc13
         pullPolicy: IfNotPresent
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
	         - key: kubernetes.io/hostname  # any kubernete node label key
	           operator: In
	           values:  # kubernete node label key's value
                   - k8s-woker-3
                   - k8s-woker-4
                   - k8s-woker-5
     probe:
       image:
         repository: iomesh/operator-probe
         tag: v1.0.0
         pullPolicy: IfNotPresent
     toolbox:
       image:
         repository: iomesh/operator-toolbox
         tag: v1.0.0
         pullPolicy: IfNotPresent
   ```

#### 挂载磁盘

使用 kubectl edit 编辑每一个 iomesh 对象，参考  https://docs.iomesh.com/deploy/setup-iomesh#block-device-object 配置磁盘

```shell
kubectl edit iomesh -n iomesh-system
kubectl edit iomesh-cluster-1 -n iomesh-cluster-1
```

#### 创建多集群连接配置

一套 IOMesh CSI 可以连接多套 IOMesh 集群。需要创建不同的 ConfigMap 来标识不同的 IOMesh 集群

```yaml
# Source: iomesh-csi-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: iomesh-csi-configmap
  namespace: iomesh-system
data:
  clusterId: k8s-cluster
  iscsiPortal: 127.0.0.1:3260
  metaAddr: iomesh-meta-client.iomesh-system.svc.cluster.local:10100
---
# Source: iomesh-cluster-1-csi-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: iomesh-cluster-1-csi-configmap
  namespace: iomesh-cluster-1
data:
  clusterId: k8s-cluster
  iscsiPortal: 127.0.0.1:3260
  metaAddr: iomesh-cluster-1-meta-client.iomesh-cluster-1.svc.cluster.local:10100
```

```shell
kubectl apply -f iomesh-csi-configmap.yaml
kubectl apply -f iomesh-cluster-1-csi-configmap.yaml
```

ConfigMap 字段释义

| 字段名 | 字段释义 |
| ------------------ | ------------------------------------------------------------ |
| data.clusterId | K8s 集群 ID，用户可自定义，一个 K8s 集群只能有一个，由于两个 iomesh 集群部署在同一 K8s 集群，这个值必须相同 |
| data.iscsiPortal   | iscsi 接入点，固定为 127.0.0.1:3260                          |
| data.metaAddr      | iomesh meta service 地址，格式为 <iomesh-cluster-name>-meta-client.<iomesh-cluster-namespace>.svc.cluster.local:10100 |



#### 为每个 IOMesh 集群创建分别对应的 StorageClass

> _Note:_
> 在多集群部署场景下，请不要使用部署 IOMesh 时自动创建的默认 StorageClass，必须为每个集群单独创建 StorageClass

```yaml
---
# Source: iomesh-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh
parameters:
  csi.storage.k8s.io/fstype: ext4
  replicaFactor: "2"
  thinProvision: "true"
  reclaimPolicy: Delete
  clusterConnection: "iomesh-system/iomesh-csi-configmap"  # 此处为上一步创建的主集群 configmap
  iomeshCluster: "iomesh-system/iomesh"
volumeBindingMode: Immediate
provisioner: com.iomesh.csi-driver
allowVolumeExpansion: true
---
# Source: iomesh-cluster-1-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: iomesh-cluster-1
parameters:
  csi.storage.k8s.io/fstype: ext4
  replicaFactor: "2"
  thinProvision: "true"
  reclaimPolicy: Delete
  clusterConnection: "iomesh-cluster-1/iomesh-cluster-1-csi-configmap"  # 此处为上一步创建的边缘集群 configmap
  iomeshCluster: "iomesh-cluster-1/iomesh-cluster-1"
volumeBindingMode: Immediate
provisioner: com.iomesh.csi-driver
allowVolumeExpansion: true
```

StorageClass 的字段释义如下:

| 字段名 | 字段释义 |
| ---------------------------- | ------------------------------------------------------------ |
| parameters.clusterConnection | 在 "创建 CSI 多集群连接配置" 步骤配置的 configmap 的 namespace/name |
| parameters.iomeshCluster     | iomesh 对象的 namespace/name                                 |

其他 StorageClass 字段释义参考 [setup-storageclass](https://docs.iomesh.com/deploy/setup-storageclass)

#### 验证

使用两个 IOMesh 集群的 storageClass 分别创建 PVC

```yaml
# Source: iomesh-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iomesh-pvc
spec:
  storageClassName: iomesh
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# Source: iomesh-cluster1-pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: iomesh-cluster1-pvc
spec:
  storageClassName: iomesh-cluster-1
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```shell
kubectl apply -f iomesh-pvc.yaml
kubectl apply -f iomesh-cluster1-pvc.yaml
```

IOMesh 默认会开启拓扑感知功能确保 Pod 调度的正确性。假设使用了第一套 IOMesh 集群创建了一个 PVC，则使用了这个 PVC 的 Pod 一定会被调度到第一套 IOMesh 集群所在的  K8s Worker 上，这做的好处是 Pod 的 IO 可以实现本地化。

---
id: external-csi
title: 使用 CSI 从集群外部接入 IOMesh
sidebar_label: 使用 CSI 从集群外部接入 IOMesh
---

IOMesh 支持直接对外提供 CSI 接入服务。用户可以通过在计算 K8s 集群部署 IOMesh CSI Driver 的方式来使用 IOMesh 提供的存储服务。

> _NOTE_: 为了顺利使用该功能，计算 K8s 集群需要能够访问 IOMesh 部署时配置的 DATA_CIDR 网段

## 配置稳定的 iSCSI 接入点
参考 [使用 IOMesh 作为 iSCSI Target](external-iscsi) 中的 "配置稳定的 iSCSI 接入点" 章节完成接入点配置。

配置完成后，检查 iomesh-access service 的状态，确保该 service 被正确分配了 external-ip

```shell
kubectl get service iomesh-access -n iomesh-system
```
```output
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                          AGE
iomesh-access   LoadBalancer   10.96.22.212   192.168.2.1     3260:32012/TCP,10206:31920/TCP,10201:31402/TCP,10207:31802/TCP   45h
```

## 在计算 K8s 集群部署 IOMesh CSI Driver

### 安装准备
计算 K8s 集群的 worker 节点需要正确安装并配置 open-iscsi，参考 [安装 open-iscsi](../deploy/prerequisites#设置-open-iscsi) 章节

### 安装 IOMesh CSI Driver

1. 加载 Helm chart 仓库。

   ```
   helm repo add iomesh http://iomesh.com/charts
   ```

2. 生成 IOMesh CSI Driver 配置文件的模板 **csi-driver.yaml**。

   ```
   helm show values iomesh/csi-driver > csi-driver.yaml
   ```
3. 根据 Kubernetes 运行的环境，修改配置文件 **csi-driver.yaml** 中对应的参数值。

    ```
    nameOverride: "iomesh-csi-driver"
    # … … …
    # Container Orchestration system (eg. "kubernetes"/"openshift" )
    co: "kubernetes"
    coVersion: "1.18"
    # … … …
    storageClass:
      nameOverride: "iomesh-csi-driver"
      parameters:
        csi.storage.k8s.io/fstype: "ext4"
        replicaFactor: "2"
        thinProvision: "true"
    # … … …
    volumeSnapshotClass:
      nameOverride: "iomesh-csi-driver"
    # … … …
    driver:
      # … … …
      # kubernetes-cluster-id
      clusterID: "my-cluster"
      # Meta server address
      metaAddr: "iomesh-cluster-vip:10206"
      # Chunk server iscsi portal address
      iscsiPortal: "iomesh-cluster-vip:3260"
      # EXTERNAL / HCI
      deploymentMode: "EXTERNAL"
      # … … …
      node:
        driver:
          # … … …
          # kubelet root directory
          kubeletRootDir: "/var/lib/kubelet"
      # … … …
    ```

    | 参数字段 | 参数取值 | 含义  |
    | ---------|---------------------|------|
    | nameOverride | iomesh-csi-driver | csi driver 名称  |
    | co       | kubernetes | 发行版的名称。  |
    | coVersion| 1.18 | 发行版的版本。  |
    | storageClass.nameOverride | iomesh-csi-driver | 默认的 StorageClass 名称，支持在安装时自定义  |
    | storageClass.parameters | parameters | 默认 StorageClass 的参数，支持在安装时自定义  |
    | volumeSnapshotClass.nameOverride | iomesh-csi-driver | 默认的 volumeSnapshotClass 名称，支持在安装时自定义  |
    | driver.clusterID |my-cluster | kubernetes 集群的唯一 ID，当多个 kubernetes 集群利用同一个 IOMesh 集群提供存储服务时，此 ID 的取值为 kubernetes 集群的唯一标识。  |
    | driver.metaAddr | iomesh-cluster-vip:10206 | 其中 **iomesh-cluster-vip** 表示 iomesh-access service 的 external ip。  |
    | driver.iscsiPortal | iomesh-cluster-vip:3260 | 其中 **iomesh-cluster-vip** 表示 iomesh-access service 的 external ip。 |
    | driver.deploymentMode | EXTERNAL | 此处设置为 **EXTERNAL**，表示从外部接入 iomesh|
    | driver.controller.driver.podDeletePolicy | no-delete-pod | 挂载了由 iomesh csi driver 创建的 PVC 的 Pod 所在的 K8s Worker 节点发故障时，是否自动删除该 Pod 让其在其他健康 Worker 节点重建。|
    | driver.node.driver.kubeletRootDir | /var/lib/kubelet | kubelet 运行根目录，kubelet 在此目录内管理挂载到 pod 的 volume，/var/lib/kubelet 为默认的运行根目录。 |

4. 部署 IOMesh CSI Driver。

    ```
    helm install csi-driver iomesh/csi-driver \
        --namespace iomesh-system \
        --create-namespace \
        --values csi-driver.yaml \
        --wait
    ```

5. IOMesh CSI Driver 部署结束后，执行命令 `watch kubectl get --namespace iomesh-system pods`，确认显示如下结果，表示 IOMesh CSI Driver 部署成功。

    ```
    NAME                                            READY   STATUS    RESTARTS   AGE
    iomesh-csi-driver-controller-plugin-5dbfb48d5c-2sk97   6/6     Running   0          42s
    iomesh-csi-driver-controller-plugin-5dbfb48d5c-cfhwt   6/6     Running   0          42s
    iomesh-csi-driver-controller-plugin-5dbfb48d5c-drl7s   6/6     Running   0          42s
    iomesh-csi-driver-node-plugin-25585                    3/3     Running   0          39s
    iomesh-csi-driver-node-plugin-fscsp                    3/3     Running   0          30s
    iomesh-csi-driver-node-plugin-g4c4v                    3/3     Running   0          39s
    ```

当 IOMesh CSI Driver 部署完成后，就可以参考 [卷操作](create-volume) 章节创建 pvc/volumesnapshot 等存储资源接入 IOMesh


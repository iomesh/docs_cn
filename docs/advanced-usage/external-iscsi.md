---
id: external-iscsi
title: 使用 IOMesh 作为 iSCSI Target
sidebar_label: 使用 IOMesh 作为 iSCSI Target
---

IOMesh 支持直接对外提供 iSCSI 接入服务。用户可以通过创建 PVC 的方式创建一个iSCSI LUN（以下简称 External iSCSI LUN，在 K8s 集群外可以使用任意一个 iSCSI 客户端（如`open-iscsi`）连接 iSCSI LUN。

> _NOTE_: 为了顺利使用该功能，iSCSI 客户端需要能够访问 IOMesh 部署时配置的 DATA_CIDR 网段

## 配置稳定的 iSCSI 接入点

为了保证 iSCSI 服务接入点的高可用和保持稳定的 IP 地址，IOMesh 采用了 LoadBalancer 类型的 service: iomesh-access， 作为 iSCSI 服务接入点。该 service 需要被分配一个对外暴露的 external-ip，具体的分配形式在不同的 IOMesh 集群部署环境中有所不同：
* 当 IOMesh 集群部署在 GCP、AWS、Azure 等原生支持 K8s 中 LoadBalancer 类型服务的云提供中时（详细的云提供商列表，请参阅 [internal-load-balancer] (https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer))，用户可以参考下述的 `In-Tree LoadBalancer` 部分。
* 对于其他部署场景（例如裸金属），用户可以参考下述的 `Out-of-Tree LoadBalancer` 部分。

<!--DOCUSAURUS_CODE_TABS-->

<!--In-Tree LoadBalancer-->

K8s 会自动调用 IaaS 平台对应的接口，为 iomesh-access service 分配一个 external-ip 作为 LoadBalancer 的入口。

检查 iomesh-access service 的状态，确保该 service 被正确分配了 external ip

    ```bash
    kubectl get service iomesh-access -n iomesh-system
    ```
    ```output
    NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                          AGE
    iomesh-access   LoadBalancer   10.96.22.212   192.168.2.1     3260:32012/TCP,10206:31920/TCP,10201:31402/TCP,10207:31802/TCP   45h
    ```

最后，执行 `kubectl edit iomesh -n iomesh-system` 命令，将 `spec.redirector.iscsiVirtualIP` 设置为 `iomesh-access` 服务的 external-ip。保存后，`iomesh-iscsi-redirector` Pod 会自动重启使设置生效。

> **_NOTE_:** `spec.redirector.iscsiVirtualIP` 需要和 `iomesh-access` 服务的 external-ip 一致。 如果 external-ip 发生变化，需要同时更新 `spec.redirector.iscsiVi rtualIP`。

<!--Out-of-Tree LoadBalancer-->

由于 IOMesh 运行在裸金属环境或其他不原生支持 K8s 中 LoadBalancer 类型服务的环境中，因此 K8s 不提供 LoadBalancer 的默认实现，LoadBalancer 类型的 Service 将始终处于 pending 状态。因此用户需要在 K8s 集群中安装 `MetallLB` 作为负载均衡器的默认实现，下面介绍安装配置方法。

1. 确保集群满足 `MetalLB` 的安装要求 [Installation Preparation](https://metallb.universe.tf/installation/#preparation)。

2. 安装 `MetalLB`

    ```bash
    helm repo add metallb https://metallb.github.io/metallb
    helm install metallb metallb/metallb --version 0.12.1 -n iomesh-system
    ```

3. 创建 `MetalLB` ConfigMap 文件 `metallb-config.yaml`，将 `MetalLB` 的工作模式设置为 layer2（基于arp），并为 `MetalLB` 分配一个IP地址池

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      namespace: iomesh-system
      name: metallb
    data:
      config: |
        address-pools:
        - name: iomesh-access-address-pools
          protocol: layer2
          addresses:
          - <fill-in-your-ip-address-pool-here> # eg: "192.168.1.100-192.168.1.110" or "192.168.2.0/24"
    ```
将 `<fill-in-your-ip-address-pool-here>` 中的注释部分替换为物理网络中未使用的 IP 地址池，可以使用 ip-range 格式，如 "192.168.1.100-192.168.1.110”，或使用子网掩码格式，例如 “192.168.2.0/24”。

4. 生效 ConfigMap 配置

    ```bash
    kubectl apply -f metallb-config.yaml
    ```

5. 查看服务状态，确保 IP 地址池中的 external-ip 已经分配给 `MetalLB`

    ```bash
    watch kubectl get service iomesh-access -n iomesh-system
    ```
    ```output
    NAME            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                          AGE
    iomesh-access   LoadBalancer   10.96.22.212   192.168.2.1     3260:32012/TCP,10206:31920/TCP,10201:31402/TCP,10207:31802/TCP   45h
    ```

最后，执行`kubectl edit iomesh -n iomesh-system`命令，将`spec.redirector.iscsiVirtualIP`设置为`iomesh-access`服务的external-ip。 保存后，`iomesh-iscsi-redirector` Pod 会自动重启使设置生效。

> **_NOTE_:** `spec.redirector.iscsiVirtualIP` 需要和 `iomesh-access` 服务的 external-ip 一致。 如果 external-ip 发生变化，需要同时更新 `spec.redirector.iscsiVirtualIP`。

<!--END_DOCUSAURUS_CODE_TABS-->

## 使用 PVC 创建 External iSCSI LUN

要使用 PVC 创建外部 iSCSI LUN，您需要在 PVC 中添加两个 annotation 字段。下面给出了字段的描述和完整示例。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: external-iscsi
  annotations:
    # 将 PVC 标记为集群外部使用的 LUN
    iomesh.com/external-use: "true"
    # 设置 LUN 的 initiator iqn acl。 如果不配置该字段，则禁止所有 initiator 的 login 操作。
    # 当 PVC 的 accessModes 为 RWO 时，该字段只能取一个值； 当 PVC 的 accessModes
    # 为 RWX 时，这个字段可以有多个值，用逗号隔开。 如果要放行所有的 iqns，配置为“*/*”
    iomesh.com/iscsi-lun-iqn-allow-list: "iqn.1994-05.com.example:a6c97f775dcb"
spec:
  storageClassName: iomesh-csi-driver
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

> **_NOTE_:** `iomesh.com/iscsi-lun-iqn-allow-list` 字段可以在 PVC 创建期间或 PVC 创建之后设置，支持动态更新。 一般可以通过 `cat /etc/iscsi/initiatorname.iscsi` 命令获取 iSCSI 客户端的 iqn。

PVC 创建并进入 Bound 状态后，可以查看对应 PV 的`spec.volumeAttributes.iscsiEntrypoint`字段。

```bash
kubectl get pv pvc-d84b4657-7ab5-4212-9270-ce40e6a1356a -o jsonpath='{.spec.csi.volumeAttributes.iscsiEndpoint}'
# output
iscsi://cluster-loadbalancer-ip:3260/iqn.2016-02.com.smartx:system:54e7022b-2dcc-4b43-800c-e52b6fad07d3/1
```

在这个例子中：
* iSCSI Portal 的 IP 地址和端口号为 cluster-loadbalancer-ip 和3260，其中 cluster-loadbalancer-ip 替换为您之前配置的 iomesh-access Service 的 external-ip。
* Target 名称是 iqn.2016-02.com.smartx:system:54e7022b-2dcc-4b43-800c-e52b6fad07d3。
* LUN ID 为 1。

有了以上信息，您就可以使用任何 iSCSI 客户端进行连接了。 假设 cluster-loadbalancer-ip 为192.168.25.101，以`open-iscsi`为例：

```bash
iscsiadm -m discovery -t sendtargets  -p 192.168.25.101:3260 --discover
iscsiadm -m node -T iqn.2016-02.com.smartx:system:54e7022b-2dcc-4b43-800c-e52b6fad07d3 -p 192.168.25.101:3260  --login
```

## 删除 PVC
出于安全考考，IOMesh 不允许删除 `iomesh.com/iscsi-lun-iqn-allow-list` 字段值不为空的 PVC。 如果要删除PVC，需要确保`iomesh.com/iscsi-lun-iqn-allow-list`字段的值为`""`，或者直接删除`iomesh.com/iscsi-lun-iqn-allow-list` 字段，然后删除 PVC。 删除 PVC 后是否保留 External iSCSI LUN 取决于 PVC 使用的 StorageClass 对应的 reclaimPolicy 字段。

## 在 IOMesh 多集群场景下使用 External LUN 功能
关于 IOMesh 多集群功能参考 [IOMesh 多集群管理](multi-clsuter/multi-cluster-deploy.md)

在 IOMesh 多集群场景中，不同的 IOMesh 集群会被部署在不同的 namespace 下。假设存在 IOMesh 集群 `iomesh-cluster-1` 并被部署在 `namespace: iomesh-cluster-1` 中，如果需要使用 `iomesh-cluster-1` 中的 External LUN，则只需要确保两点即可：
1. 根据上网创建一个 LoadBalancer ，namespace 和 `iomesh-cluster-1` 的 namespace 相同
2. 执行 `kubectl edit iomesh-cluster-1 -n iomesh-cluster-1` 命令，将 `spec.redirector.iscsiVirtualIP` 设置为 LoadBalancer 的 external-ip。保存后，`iomesh--cluster-1-iscsi-redirector` Pod 会自动重启使设置生效

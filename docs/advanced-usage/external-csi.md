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

参考 [IOMesh CSI Driver用户指南](http://docs.smartx.com/zbs-5.2.0/docs/k8s_csi_driver_installation_guide_01.html) 部署 IOMesh CSI Driver，在部署时 helm values 参数做如下配置：
| 参数名称                  | 参数释义                                               | 配置值           |
| ------------------------- | ------------------------------------------------------ | ------------------ |
| driver.clusterID          | k8s 集群的唯一标识，可自定义，但必须保证每一个从外部接入 k8s集群的标识唯一  | 可自定义，例如 k8s-cluster-0 |
| driver.metaAddr           | IOMesh 元数据服务的 ip 和端口           | iomesh-access 的 external-ip:10206，在本例中为 192.168.2.1:10206 |
| driver.iscsiPortal         | IOMesh iscsi 接入服务的 ip 和端口 |  iomesh-access 的 external-ip:3260，在本例中为 192.168.2.1:3260 |
| driver.deploymentMode         | IOMesh CSI Driver 部署模式        | EXTERNAL |

当 IOMesh CSI Driver 部署完成后，就可以参考 [卷操作](create-volume) 章节创建 pvc/volumesnapshot 等存储资源接入 IOMesh


---
id: overview
title: 概述
sidebar_label: 概述
---

## 什么是 IOMesh LocalPV Manager

IOMesh LocalPV Manager 是一个用于管理 K8s Worker 节点本地存储的 csi-driver，用户可以基于节点上的一个目录或一块磁盘创建 Kubernetes Persistent Volumes 提供给 Pod 使用

一些有状态应用如 分布式对象存储(Minio)/分布式数据库(TiDB) 等，能够在应用层保证数据的高可用，如果在多副本的 IOMesh PV 上运行它们会在数据路径中增加一层复制（目前 IOMesh 不支持单副本），这会导致一定程度的性能下降和空间浪费，因此这种场景下更适合使用 IOMesh LocalPV

相比于 Kubernetes hostPath 和 Kubernetes 原生的 local PV，IOMesh LocalPV 有如下优势：

1. Kubernetes 的原始功能只支持静态配置（statically provisioned），IOMesh LocalPV 支持动态配置（dynamic provisioned），这意味着用户可以使用StorageClass、PVC 和 PV 灵活的访问节点本地存储，且不需要管理员预先创建静态 PV

2. Kubernetes 的原始功能只支持使用节点上的目录，IOMesh LocalPV 支持使用目录或块设备
3. 当使用节点上的目录作为后端存储时，IOMesh LocalPV 支持 PV 级别的容量限额

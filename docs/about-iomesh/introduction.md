---
id: introduction
title: 介绍
sidebar_label: 介绍
---

## 什么是 IOMesh ？

IOMesh 是专门为 Kubernetes 工作负载设计的分布式存储系统，为 MySQL、Cassandra、MongoDB 等容器化的有状态应用程序提供可靠的持久化存储能力。

- 在 Kubernetes 集群中每分钟创建和销毁数以千计的 Pod。 IOMesh 就是为云原生时代这种高度动态、大规模的工作负载而构建的。它从设计之初就考虑到了这一点，以提供云原生应用程序所需的性能、可靠性和可扩展性。
- IOMesh 在 Kubernetes 上运行并充分利用 Kubernetes 的能力。因此运维团队可以利用标准的 Kubernetes API 统一管理应用和 IOMesh，与现有的 DevOps 流程完美结合。
- IOMesh 使用户可以从小规模开始，通过添加磁盘或节点来随意扩展存储。

## 关键特性

### 高性能
   数据库是衡量存储性能的关键应用之一。 IOMesh 在数据库性能测试中表现出色，读写延迟低且稳定，QPS/TPS 高，意味着提供稳定的数据服务。
### 无内核依赖
   IOMesh 完全运行在用户空间，可以通过有效的软件故障隔离提供可靠的服务。当 IOMesh 出现故障时，运行在同一节点上的其他应用程序可以继续运行，而不会导致整个系统崩溃。 此外，IOMesh 的部署和维护非常简单，因为您不需要安装任何内核模块，更不需要担心内核版本兼容性问题。
### 存储性能分层
   IOMesh 部署模式灵活，包括 NVMe SSD、SATA SSD 和 HDD 在内的混合磁盘。 这可以帮助用户充分利用他们的存储资源，最大限度地降低存储硬件成本，同时最大限度地提高存储性能。

## 架构

![IOMesh arch](https://user-images.githubusercontent.com/78140947/122766241-e2352c00-d2d3-11eb-9630-bb5b428c3178.png)

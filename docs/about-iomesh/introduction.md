---
id: introduction
title: 了解 IOMesh
sidebar_label: 了解 IOMesh
---

# 了解 IOMesh

## 什么是 IOMesh ？

IOMesh 是 SmartX 专为 Kubernetes 工作负载而设计的分布式存储系统，可为容器化的有状态应用程序提供可靠和持久化的存储能力，例如 MySQL、Cassandra、MongoDB 等应用。

IOMesh 运行在 Kubernetes 之上，可充分运用 Kubernetes 的能力。通过利用 Kubernetes API 统一管理其应用和 IOMesh，将 IOMesh 与现有的 DevOps 流程完美结合。

云原生时代，在 Kubernetes 集群中每分钟创建和销毁的 Pod 数以千计。IOMesh 从设计之初就充分考虑到了 Kubernetes 工作负载的灵活度高、规模大的特点，并为其成功提供云原生应用程序所需的高性能、高可靠性和可扩展性。您可以从构建小规模的 IOMesh 集群开始，即集群中只有很少数量的节点，后续可以通过添加磁盘或节点来随意扩展存储。

## IOMesh 的关键特性

### 高性能

数据库是衡量存储性能的关键应用之一。IOMesh 在数据库性能测试中表现出色，读/写延迟较低，QPS/TPS 较高，这表明 IOMesh 可以提供稳定的数据服务，性能较高。

### 无内核依赖

IOMesh 的部署和维护非常简单，无任何内核依赖。

- 部署时无需安装任何内核模块，因此无需担心内核版本的兼容性问题。

- IOMesh 完全运行在用户空间，通过软件故障隔离可提供可靠有效的服务。即使 IOMesh 出现故障，在同一节点上的其他应用程序仍然可以继续运行，整个系统不至于受到影响而崩溃，保证了系统的稳定性。
  
### 部署灵活

IOMesh 部署模式灵活，支持全闪和混闪的部署模式。

在混闪配置下部署 IOMesh 可使用分层存储模式，将速度更快的磁盘作为缓存盘，较低速度的磁盘作为存储盘，充分利用 NVMe SSD、SATA SSD 和 HDD 磁盘的各自优势，最大限度地降低存储硬件成本和提高存储性能。

## 架构

![IOMesh arch](https://user-images.githubusercontent.com/78140947/122766241-e2352c00-d2d3-11eb-9630-bb5b428c3178.png)

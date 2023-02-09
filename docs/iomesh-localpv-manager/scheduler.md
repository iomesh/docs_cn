---
id: scheduler
title: 调度器拓展
sidebar_label: 调度器拓展
---
IOMesh LocalPV Manager 会在后续的版本中会基于 [k8s-scheduling-framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/) 提供独立的调度器拓展，**当前版本尚未支持**

## 背景
为什么 IOMesh LocalPV Manager 需要独立的调度器拓展? 主要原因如下
1. 如果只使用 K8s 默认的调度策略，则使用了 IOMesh LocalPV 的 Pod 调度可能会失败，典型的原因例如：<br><br>
	a. 由于 K8s 默认的调度策略并不感知节点本地存储状态，因此 Pod 可能会调度到本地存储不满足需求的 Node 上导致 Pod 启动失败<br><br>
	b. 由于 K8s Pod 调度过程是同步的（同一时间点只为一个 Pod 分配 Node），Pod 与节点绑定过程是异步（同一时间点可并发在一个 Node 启动多个调度在该 Node 上的 Pod），因此可能出现 Pod 在启动时发现本地存储资源不足的情况，这种场景需要做竞争处理

2. 确保集群中各 Node 的 Local PV 容量使用尽可能均衡

## Roadmap
调度器拓展的计划支持版本为 IOMesh LocalPV Manager v1.0.0，当前版本为 IOMesh LocalPV Manager v0.9.0

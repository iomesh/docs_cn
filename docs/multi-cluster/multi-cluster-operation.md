---
id: multi-cluster-operation
title: IOMesh 多集群运维
sidebar_label: IOMesh 多集群运维
---

## 升级
在多集群场景下，需要先升级管控 namespace 下的 IOMesh 集群，再升级非管控 namespace 下的 IOMesh 集群

以 [IOMesh 多集群部署](multi-cluster-deploy.md) 中的部署场景为例，升级步骤如下：
1. 参考 [IOMesh 运维](iomesh-operations/cluster-operations.md) 中的升级章节，先升级第一套 IOMesh 集群和 IOMesh 管理组件
2. 第一套 IOMesh 集群升级完毕后，使用 `kubectl edit iomesh iomesh-cluster-1 -n iomesh-cluster-1` 编辑第二套 IOMesh 集群，修改所有的 `*.image.tags`  字段与第一套 IOMesh 集群保持一致

若未按照上述正确的顺序进行升级，比如先升级了第二套集群，再升级第一套，则在升级第二套集群时该集群可能会暂时处于一个错误状态。当第一套被升级完成后，两个集群都会最终达到正确状态

## 扩容
IOMesh 实例扩容方式与 [IOMesh 运维](iomesh-operations/cluster-operations.md) 中扩容章节保持一致

## 卸载
在多集群场景下，需要先卸载非管控 namespace 下的 IOMesh 集群，再卸载管控 namespace 下的 IOMesh 集群 IOMesh 集群

以 [IOMesh 多集群部署](multi-cluster-deploy.md) 中的场景为例，卸载步骤如下：
1. 卸载第二套 IOMesh 集群, 同时删除集群中的 iomesh 和 zookeeper

```shell
kubectl delete -f iomesh-cluster-1-zookeeper.yaml && kubectl delete -f iomesh-cluster-1.yaml
```

2. 参考 [IOMesh 运维](iomesh-operations/cluster-operations.md) 中的卸载章节，卸载第一套 IOMesh 集群和 IOMesh 管理组件

若未按照上述正确的顺序进行卸载，比如先卸载了第一套集群，再卸载第二套，可能会造成部分资源在该 namespace 下残留需要手动处理，如果不清理可能影响下一次在这个 namespace 下部署

## License 管理
多个 IOMesh 集群间的 license 相互独立，每个 IOMesh 集群拥有一个独立的集群序列号，当需要激活 license 时请参考 https://www.iomesh.com/license，每个 IOMesh 集群需要被单独激活

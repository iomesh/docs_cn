---
id: deploy
title: 安装部署
sidebar_label: 安装部署
---

IOMesh LocalPV Manager 不依赖于 IOMesh，可独立安装部署和使用

## 前置条件

支持的 Kubernetes 版本版本范围：v1.13~v1.25

## 安装

### 添加 IOMesh Helm Repo

```shell
helm repo add iomesh http://iomesh.com/charts 
helm repo update
```

### 安装 IOMesh LocalPV Manager

```shell
helm install iomesh-localpv-manager --create-namespace -n iomesh-system
```
检查安装结果，所有的 Pod 处于 `running` 状态
```shell
kubectl --namespace iomesh-system get pods
```
```output
NAME                                                    READY   STATUS             RESTARTS   AGE
iomesh-localpv-manager-rjg8j                                   4/4     Running            0          10s
iomesh-localpv-manager-vphz9                                   4/4     Running            0          10s
iomesh-localpv-manager-w4j8m                                   4/4     Running            0          10s
iomesh-localpv-manager-openebs-ndm-operator-7474bbfdb8-df59w   1/1     Running            0          25s
iomesh-localpv-manager-openebs-ndm-p29fq                       1/1     Running            0          13s
iomesh-localpv-manager-openebs-ndm-sff9p                       1/1     Running            0          25s
iomesh-localpv-manager-openebs-ndm-v56tj                       1/1     Running            0          25s
```

如果在已存在 IOMesh 的 K8s 集群中部署 IOMesh LocalPV Manager，则 IOMesh LocalPV Manager 必须部署在 IOMesh 所在的 namespace 中，此时 IOMesh LocalPV Manager 会与 IOMesh 共用 node-disk-manager 组件

## 卸载

### 卸载 IOMesh LocalPV Manager

```shell
helm uninstall iomesh-localpv-manager -n iomesh-system
```

如果在 IOMesh 与 IOMesh LocalPV Manager 共存的状态下卸载 IOMesh LocalPV Manager，则会保留 node-disk-manager 组件

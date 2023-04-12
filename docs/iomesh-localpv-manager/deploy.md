---
id: deploy
title: 安装部署
sidebar_label: 安装部署
---

### 安装 IOMesh LocalPV Manager

IOMesh LocalPV Manager 会在安装 IOMesh 时自动安装，通过如下命令查看 IOMesh LocalPV Manager pod 的运行状态

```shell
kubectl get pod -n iomesh-system | grep localpv-manager
```
```output
NAME                                                           READY   STATUS             RESTARTS   AGE
iomesh-localpv-manager-rjg8j                                   4/4     Running            0          10s
iomesh-localpv-manager-vphz9                                   4/4     Running            0          10s
iomesh-localpv-manager-w4j8m                                   4/4     Running            0          10s
```


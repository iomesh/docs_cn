---
id: deploy
title: 安装部署
sidebar_label: 安装部署
---

IOMesh LocalPV Manager 不依赖于 IOMesh，可独立安装部署和使用

## 安装部署

### 添加 IOMesh Helm Repo

```shell
helm repo add iomesh http://iomesh.com/charts 
helm repo update
```

### 安装 IOMesh LocalPV Manager

```shell
helm install localpv-manager --create-namespace -n iomesh-system
```

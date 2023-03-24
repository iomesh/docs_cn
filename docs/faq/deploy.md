---
id: deploy
title: 安装部署
sidebar_label: 安装部署
---

### Docker 镜像拉取失败

#### 错误日志
`Too Many Requests - Server message: toomanyrequests: You have reached your pull rate limit.`

#### 修复建议
在对接 worker 节点上进行 docker login 任意账户或参考 https://www.docker.com/increase-rate-limit 进行 docker 用户升级。另外直接使用离线安装方式可以避免该问题

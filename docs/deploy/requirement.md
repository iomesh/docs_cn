---
id: requirement
title: 构建 IOMesh 集群的要求
sidebar_label: 构建 IOMesh 集群的要求
---
# 构建 IOMesh 集群的要求

## 运行 IOMesh 的 Kubernetes 集群要求

运行 IOMesh 的 Kubernetes 集群须满足以下两点：
- 集群规模不少于 3 个 worker 节点；
- 集群版本不低于 v1.17，且不高于 v1.24。

## Worker 节点的硬件配置要求

<table>
   <thead>
   <tr class="header">
      <th><strong>硬件</strong></th>
      <th><strong>配置要求</strong></th>
   </tr>
   </thead>
  <tbody>
   <tr class="odd">
      <td><strong>CPU</strong></td>
      <td><p>至少 6 核 vCPU</p></td> 
   </tr>
   <tr class="even">
      <td><strong>内存</strong></td>
      <td>至少 12 GB</td>  
   </tr>
   <tr class="odd">
      <td><strong>缓存盘</strong></td>
      <td><p>由 IOMesh 集群的部署模式决定。</p>
      <ul><li>全闪模式：无须配置磁盘。</li>
      <li>混闪模式：至少包含一个 SSD，其容量须大于 60 GB。</li>
      </ul></td>     
   </tr>
      <tr class="even">
      <td><strong>数据盘</strong></td>
      <td><p>由 IOMesh 集群的部署模式决定。</p>
      <ul><li>全闪模式：至少包含一个 SSD，其容量须大于 60 GB。</li>
      <li>混闪模式：至少包含一个 HDD，其容量须大于 60 GB。</li>
      </ul></td>     
   </tr>
  </tbody>
</table>

## 存储网络要求

为避免其他应用抢占网络带宽，请为 IOMesh 集群设置单独的存储网络。
- 网络带宽：集群的存储网络带宽需不低于 10Gbps。
- 网络 IP：`dataCIDR` 定义了 IOMesh 存储网络的 IP 网段，运行 IOMesh 的每个节点所对应的 IP 地址须位于该网段内。

## 集群元数据存储空间要求

IOMesh 集群将元数据存储在 worker 节点的本地路径 `/opt/iomesh` 中。为了保证数据存放拥有足够的存储空间，每个 worker 节点上的 `/opt` 目录需至少预留 100GB 供元数据存放。


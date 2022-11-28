---
id: prerequisites
title: 安装准备
sidebar_label: 安装准备
---

## 安装要求
#### Kubernetes 集群要求
具有至少 3 个 worker 节点的 Kubernetes（从 v1.17 到 v1.24）集群。

#### CPU 和内存要求
##### CPU
每个工作节点上至少有 6 个核心。

##### 内存
每个工作节点上至少有 12GB RAM。

#### 磁盘要求
##### 缓存磁盘
* All-Flash 模式：无需配置。
* Hybrid-flash 模式：每个 worker 节点至少有一个可用的 SSD，SSD 容量大于 60GB。

#####数据盘
* All-Flash 模式：每个 worker 节点上至少有一块可用的 SSD，SSD 容量大于 60GB。
* Hybrid-flash 模式：每个 worker 节点至少有一个可用的硬盘，硬盘容量大于 60GB。

#### 网络要求
IOMesh 存储网络需要 10GbE 或以上的网卡。

#### 系统空间预留
每个 worker 节点上的 /opt 目录至少需要 100GB 的磁盘空间来存储 IOMesh 集群元数据。

## worker 节点设置
按照以下步骤设置运行 IOMesh 的每个 Kubernetes worker 节点。

### 设置 Open-iSCSI

1. 安装 open-iscsi.

  <!--DOCUSAURUS_CODE_TABS-->

  <!--RHEL/CentOS-->

  ```shell
  sudo yum install iscsi-initiator-utils -y
  ```

  <!--Ubuntu-->

  ```shell
  sudo apt-get install open-iscsi -y
  ```

  <!--END_DOCUSAURUS_CODE_TABS-->

2. 编辑 `/etc/iscsi/iscsid.conf` ， 将 `node.startup` 字段设为 `manual`

    ```shell
    sudo sed -i 's/^node.startup = automatic$/node.startup = manual/' /etc/iscsi/iscsid.conf
    ```
    > **_NOTE_: The default value of the MaxRecvDataSegmentLength in /etc/iscsi/iscsi.conf is set at 32,768, and the maximum number of PVs is limited to 80,000 in IOMesh. To create PVs more than 80,000 in IOMesh, it is recommended to set the value of MaxRecvDataSegmentLength to 163,840 or above.**
    
3. 关闭 SELinux.

    ```shell
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

4. 加载 `iscsi_tcp` kernel module.

    ```shell
    sudo modprobe iscsi_tcp
    sudo bash -c 'echo iscsi_tcp > /etc/modprobe.d/iscsi-tcp.conf'
    ```

5. 启动 `iscsid` 服务.

    ```shell
    sudo systemctl enable --now iscsid
    ```

### 设置本地元数据存储

IOMesh 将元数据存储在本地路径 `/opt/iomesh` 中。 确保 `/opt` 至少有 100Gb 的可用空间。

### 设置存储网络

为避免网络带宽争用，请为 IOMesh 集群设置单独的存储网络。 `dataCIDR` 定义了 IOMesh 数据网络的 IP 网段。 每个运行 IOMesh 的节点都应该有一个 IP 地址属于 `dataCIDR` 的接口。

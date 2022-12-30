---
id: prerequisites
title: 设置 worker 节点
sidebar_label: 设置 worker 节点
---

# 设置 worker 节点

在正式部署 IOMesh 集群前，请先参考如下步骤，对每个即将运行 IOMesh 的 Kubernetes worker 节点进行设置。

## 设置 Open-iSCSI

1. 安装 open-iscsi。

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

2. 编辑 `/etc/iscsi/iscsid.conf` 文件， 将 `node.startup` 字段设置为 `manual`。

   ```shell
   sudo sed -i 's/^node.startup = automatic$/node.startup = manual/' /etc/iscsi/iscsid.conf
   ```
   > **_NOTE_:**
   > 
   > 在 `/etc/iscsi/iscsi.conf` 文件中，`MaxRecvDataSegmentLength` 的默认值为 32768，而允许 IOMesh 创建的最大 PV 数为 80000；如果 IOMesh 创建的 PV 数可能超过 80000，建议将 `MaxRecvDataSegmentLength` 的值设置为 163840 及以上。
    
3. 关闭 SELinux。

    ```shell
    sudo setenforce 0
    sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

4. 加载 Kernel 内核模块 `iscsi_tcp`。

    ```shell
    sudo modprobe iscsi_tcp
    sudo bash -c 'echo iscsi_tcp > /etc/modprobe.d/iscsi-tcp.conf'
    ```

5. 启动 `iscsid` 服务。

    ```shell
    sudo systemctl enable --now iscsid
    ```




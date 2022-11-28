---
id: iomesh-for-mysql
title: 使用 IOMesh 部署 MySQL
sidebar_label: 使用 IOMesh 部署 MySQL
---

## 设置 k8s 集群存储

1. 创建一个名为 `iomesh-mysql-sc.yaml` 的文件，内容如下：

    ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: iomesh-mysql-sc
    provisioner: com.iomesh.csi-driver # driver.name in values.yaml when install IOMesh cluster
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    parameters:
      csi.storage.k8s.io/fstype: "ext4"
      replicaFactor: "2"
      thinProvision: "true"
    ```

2. 生效该 yaml 配置:

    ```bash
    kubectl apply -f iomesh-mysql-sc.yaml
    ```

## 部署 MySQL

1. 创建文件 `mysql-deployment.yaml`

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: iomesh-mysql-pvc
    spec:
      storageClassName: iomesh-mysql-sc
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 10Gi
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: mysql
    spec:
      ports:
      - port: 3306
      selector:
        app: mysql
      clusterIP: None
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: mysql
    spec:
      selector:
        matchLabels:
          app: mysql
      strategy:
        type: Recreate
      template:
        metadata:
          labels:
            app: mysql
        spec:
          containers:
          - image: mysql:5.6
            name: mysql
            env:
              # Use secret in real usage
            - name: MYSQL_ROOT_PASSWORD
              value: password
            ports:
            - containerPort: 3306
              name: mysql
            volumeMounts:
            - name: mysql-persistent-storage
              mountPath: /var/lib/mysql
          volumes:
          - name: mysql-persistent-storage
            persistentVolumeClaim:
              claimName: iomesh-mysql-pvc # pvc from iomesh created above
    ```

2. 生效 yaml 配置:

    ```bash
    kubectl apply -f mysql-deployment.yaml
    ```

## 操作 MySQL 数据

用户可以使用 IOMesh 存储提供的特性对 MySQL 数据所在的 PV 进行扩展/快照/回滚/克隆等操作，详见参考 [application-operations](https://docs.iomesh.com/volume-operations/snapshot-restore-and-clone)

---
id: iomesh-for-mongodb
title: 使用 IOMesh 部署 MongoDB
sidebar_label: 使用 IOMesh 部署 MongoDB
---

## 设置 k8s 集群存储

1. 创建一个名为 `iomesh-MongoDB-sc.yaml` 的文件，内容如下：

    ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: iomesh-mongodb-sc
    provisioner: com.iomesh.csi-driver # driver.name in values.yaml when install IOMesh
    reclaimPolicy: Retain
    allowVolumeExpansion: true
    parameters:
      csi.storage.k8s.io/fstype: "ext4"
      replicaFactor: "2"
      thinProvision: "true"
    ```

2. 生效该 yaml 配置:

    ```bash
    kubectl apply -f iomesh-mongodb-sc.yaml
    ```

## 部署 MongoDB

### 为 MongoDB 创建一个 headless Service

1. 创建 headless Service

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: mongo
      labels:
        name: mongo
    spec:
      ports:
        - port: 27017
          targetPort: 27017
      clusterIP: None
      selector:
        role: mongo
    ```

2. 生效该 yaml 配置

    ```bash
    kubectl apply -f mongodb-service.yaml
    ```

### 使用 IOMesh 集群提供的 pv 创建 MongoDB 集群

1. 创建 MongoDB 集群

    ```yaml
    apiVersion: apps/v1beta1
    kind: StatefulSet
    metadata:
      name: mongo
    spec:
      selector:
        matchLabels:
          role: mongo
          environment: test
      serviceName: "mongo"
      replicas: 3
      template:
        metadata:
          labels:
            role: mongo
            environment: test
        spec:
          terminationGracePeriodSeconds: 10
          containers:
          - name: mongo
            image: mongo
            command:
              - mongod
              - "--replSet"
              - rs0
              - "--smallfiles"
              - "--noprealloc"
            ports:
              - containerPort: 27017
            volumeMounts:
              - name: mongo-persistent-storage
                mountPath: /data/db
          - name: mongo-sidecar
            image: cvallance/mongo-k8s-sidecar
            env:
              - name: MONGO_SIDECAR_POD_LABELS
                value: "role=mongo,environment=test"
      volumeClaimTemplates:
      - metadata:
          name: mongodb-data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: iomesh-mongodb-sc # storageClass created above
          resources:
            requests:
              storage: 10Gi
    ```

2. 生效 yaml 配置:

    ```bash
    kubectl apply -f mongodb-statefulset.yaml
    ```

IOMesh 存储将为每个 MongoDB pod 创建持久卷， 这些卷副本数为 2 , 被格式化为 ext4 文件系统，自动精简配置。

## 操作 MongoDB 数据
用户可以使用 IOMesh 存储提供的特性对 MongoDB 数据所在的 PV 进行扩展/快照/回滚/克隆等操作，详见参考 [application-operations](https://docs.iomesh.com/volume-operations/snapshot-restore-and-clone)

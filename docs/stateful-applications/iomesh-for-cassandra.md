---
id: iomesh-for-cassandra
title: 使用 IOMesh 部署 Cassandra
sidebar_label: 使用 IOMesh 部署 Cassandra 
---

## 设置 k8s 集群存储

1. 创建一个名为 `iomesh-cassandra-sc.yaml` 的文件，内容如下：

    ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: iomesh-cassandra-sc
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
    kubectl apply -f iomesh-cassandra-sc.yaml
    ```

## 部署 Cassandra

### 为 Cassandra 创建一个 headless Service

1. 创建 headless Service

    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: cassandra
      name: cassandra
    spec:
      clusterIP: None
      ports:
      - port: 9042
      selector:
        app: cassandra
    ```

2. 生效该 yaml 配置

    ```bash
    kubectl apply -f cassandra-service.yaml
    ```

### 使用 IOMesh 集群提供的 pv 创建 Cassandra 集群

1. 创建 Cassandra 集群

    ```yaml
    apiVersion: apps/v1
    kind: StatefulSet
    metadata:
      name: cassandra
      labels:
        app: cassandra
    spec:
      serviceName: cassandra
      replicas: 3
      selector:
        matchLabels:
          app: cassandra
      template:
        metadata:
          labels:
            app: cassandra
        spec:
          terminationGracePeriodSeconds: 1800
          containers:
          - name: cassandra
            image: gcr.io/google-samples/cassandra:v13
            imagePullPolicy: Always
            ports:
            - containerPort: 7000
              name: intra-node
            - containerPort: 7001
              name: tls-intra-node
            - containerPort: 7199
              name: jmx
            - containerPort: 9042
              name: cql
            resources:
              limits:
                cpu: "500m"
                memory: 1Gi
              requests:
                cpu: "500m"
                memory: 1Gi
            securityContext:
              capabilities:
                add:
                  - IPC_LOCK
            lifecycle:
              preStop:
                exec:
                  command:
                  - /bin/sh
                  - -c
                  - nodetool drain
            env:
              - name: MAX_HEAP_SIZE
                value: 512M
              - name: HEAP_NEWSIZE
                value: 100M
              - name: CASSANDRA_SEEDS
                value: "cassandra-0.cassandra.default.svc.cluster.local"
              - name: CASSANDRA_CLUSTER_NAME
                value: "K8Demo"
              - name: CASSANDRA_DC
                value: "DC1-K8Demo"
              - name: CASSANDRA_RACK
                value: "Rack1-K8Demo"
              - name: POD_IP
                valueFrom:
                  fieldRef:
                    fieldPath: status.podIP
            readinessProbe:
              exec:
                command:
                - /bin/bash
                - -c
                - /ready-probe.sh
              initialDelaySeconds: 15
              timeoutSeconds: 5
            volumeMounts:
            - name: cassandra-data
              mountPath: /cassandra_data
      volumeClaimTemplates:
      - metadata:
          name: cassandra-data
        spec:
          accessModes: [ "ReadWriteOnce" ]
          storageClassName: iomesh-cassandra-sc # storageClass created above
          resources:
            requests:
              storage: 10Gi
    ```

2. 生效 yaml 配置:

    ```bash
    kubectl apply -f cassandra-statefulset.yaml
    ```

IOMesh 存储将为每个 Cassandra pod 创建持久卷， 这些卷副本数为 2 , 被格式化为 ext4 文件系统，自动精简配置。

## 操作 Cassandra 数据
用户可以使用 IOMesh 存储提供的特性对 Cassandra 数据所在的 PV 进行扩展/快照/回滚/克隆等操作，详见参考 [application-operations](https://docs.iomesh.com/volume-operations/snapshot-restore-and-clone)

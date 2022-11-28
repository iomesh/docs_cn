---
id: performance-testing
title: 性能测试
sidebar_label: 性能测试 
---

## FIO 基准性能测试

1. 创建 fio 测试 Pod

    ```shell
    kubectl apply -f https://docs.iomesh.com/assets/iomesh-csi-driver/example/fio.yaml
    ```

2. 等待 fio-pvc 进入 bound 状态并且 fio pod 进入 ready 状态

    ```shell
    kubectl get pvc fio-pvc
    ```

    ```output
    NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                AGE
    fio-pvc   Bound    pvc-d7916b34-50cd-49bd-86f9-5287db1265cb   30Gi       RWO            iomesh-csi-driver-default   15s
    ```

    ```shell
    kubectl wait --for=condition=Ready pod/fio
    ```

    ```output
    pod/fio condition met
    ```

3. 运行 fio 测试

    ```shell
    kubectl exec -it fio sh
    fio --name fio --filename=/mnt/fio --bs=256k --rw=write --ioengine=libaio --direct=1 --iodepth=128 --numjobs=1 --size=$(blockdev --getsize64 /mnt/fio)
    fio --name fio --filename=/mnt/fio --bs=4k --rw=randread --ioengine=libaio --direct=1 --iodepth=128 --numjobs=1 --size=$(blockdev --getsize64 /mnt/fio)
    ```

4. 清理环境

    ```shell
    kubectl delete pod fio
    kubectl delete pvc fio-pvc
    # You need to delete pv when reclaimPolicy is Retain
    kubectl delete pvc-d7916b34-50cd-49bd-86f9-5287db1265cb
    ```

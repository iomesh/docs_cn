---
id: install-iomesh
title: 安装 IOMesh 
sidebar_label: 安装 IOMesh 
---
# 安装 IOMesh 

IOMesh 支持在所有 Kubernetes 平台通过以下几种方式进行安装，您可以根据现场实际情况选择安装方式。
- **快速安装**：一键式在线安装，但只能使用文件里的默认设置，无法自定义参数。
- **自定义安装**：用户可以自定义参数，安装时必须确保 Kubernetes 集群网络与外网连通。
- **离线安装**：当 Kubernetes 集群网络与外网无法连通时，建议采用此种方式安装，安装时支持用户自定义参数。

此外，如果您想将 IOMesh 部署在 Red Hat OpenShift 容器平台，请参考在 OpenShift 容器平台安装 IOMesh。


## 快速安装

快速安装 IOMesh 时，请对照操作系统的版本，选择执行对应的脚本文件。

> **_NOTE_:**
> 
> 脚本文件中已包含 Kubernetes 的包管理工具 Helm3。

<!--DOCUSAURUS_CODE_TABS-->

<!--RHEL7/CentOS7-->

```shell
# 运行 IOMesh 的每个 Worker 节点所对应的 IP 地址必须位于 IOMESH_DATA_CIDR 网段内
export IOMESH_DATA_CIDR=10.234.1.0/24; curl -sSL https://iomesh.run/install_iomesh_el7.sh | sh -
```

<!--RHEL8/CentOS8/CoreOS-->

```shell
# 运行 IOMesh 的每个 Worker 节点所对应的 IP 地址必须位于 IOMESH_DATA_CIDR 网段内
export IOMESH_DATA_CIDR=10.234.1.0/24; curl -sSL https://iomesh.run/install_iomesh_el8.sh | sh -
```
<!--END_DOCUSAURUS_CODE_TABS-->

执行完脚本文件后等待几分钟，然后执行下述命令，如果 IOMesh 集群里每个节点的 pod 正常运行，表明 IOMesh 集群安装成功。

```shell
watch kubectl get --namespace iomesh-system pods
```

> **_NOTE_:**
> 
> 上述脚本文件在安装结束后仍将会继续保留，以便在安装出现错误时帮助排除故障。您可以通过运行脚本 `curl -sSL https://iomesh.run/uninstall_iomesh.sh | sh -`，删除 IOMesh 安装脚本文件。


## 自定义安装

### 安装 Helm3

在正式部署 IOMesh 前，需执行如下命令，在 Kubernetes 集群上安装 Kubernetes 的包管理工具 Helm3；如果当前 Kubernetes 集群已安装 Helm3，请跳过此步骤，直接开始[加载 IOMesh Helm Repo](#加载-iomesh-helm-repo)。更多细节请参考 Helm 官方文档 [Installing Helm](https://helm.sh/docs/intro/install/)

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### 加载 IOMesh Helm Repo

```shell
helm repo add iomesh http://iomesh.com/charts
```

### 安装 IOMesh

1. 将默认的 IOMesh 配置文件渲染至 `iomesh.yaml` 中。

    ```shell
    helm show values iomesh/iomesh > iomesh.yaml
    ```

2. 配置 `iomesh.yaml`.

   **(必须)** 根据您的网络环境填写 `dataCIDR` 字段

    ```yaml
    iomesh:
      chunk:
        dataCIDR: "10.234.1.0/24"
    ```

    **(可选)** 配置集群的部署模式，默认是混闪模式。如果要部署为全闪则需要设则 `diskDeploymentMode` 字段为`allFlash` 

    ```yaml
    diskDeploymentMode: "hybridFlash" # set `diskDeploymentMode` to `allFlash` in all-flash deployment mode
    ```
   
   **(可选)** 如果只期望将 IOMesh 使用部分 Kubernetes node 的磁盘，在 `chunk.podPolicy.affinity` 字段配置对应 node 的 label. 例如:
   
   ```yaml
   iomesh:
     chunk:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname # specific node label's key
                   operator: In
                   values:
                   - iomesh-worker-0 # specific node label's value
                   - iomesh-worker-1
    ```
    更多的 pod affinity 配置规则参考: [pod affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity).

3. 安装 IOMesh 集群

    ```shell
    helm install iomesh iomesh/iomesh \
        --create-namespace \
        --namespace iomesh-system \
        --values iomesh.yaml \
        --wait
    ```

    ```output
    NAME: iomesh
    LAST DEPLOYED: Wed Jun 30 16:00:32 2021
    NAMESPACE: iomesh-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    ```

4.  检查安装结果

    ```bash
    kubectl --namespace iomesh-system get pods
    ```

    ```output
    NAME                                                  READY   STATUS    RESTARTS   AGE
    csi-driver-controller-plugin-89b55d6b5-8r2fc          6/6     Running   10         2m8s
    csi-driver-controller-plugin-89b55d6b5-d4rbr          6/6     Running   10         2m8s
    csi-driver-controller-plugin-89b55d6b5-n5s48          6/6     Running   10         2m8s
    csi-driver-node-plugin-9wccv                          3/3     Running   2          2m8s
    csi-driver-node-plugin-mbpnk                          3/3     Running   2          2m8s
    csi-driver-node-plugin-x6qrk                          3/3     Running   2          2m8s
    iomesh-chunk-0                                        3/3     Running   0          52s
    iomesh-chunk-1                                        3/3     Running   0          47s
    iomesh-chunk-2                                        3/3     Running   0          43s
    iomesh-hostpath-provisioner-8fzvj                     1/1     Running   0          2m8s
    iomesh-hostpath-provisioner-gfl9k                     1/1     Running   0          2m8s
    iomesh-hostpath-provisioner-htzx9                     1/1     Running   0          2m8s
    iomesh-iscsi-redirector-96672                         2/2     Running   1          55s
    iomesh-iscsi-redirector-c2pwm                         2/2     Running   1          55s
    iomesh-iscsi-redirector-pcx8c                         2/2     Running   1          55s
    iomesh-meta-0                                         2/2     Running   0          55s
    iomesh-meta-1                                         2/2     Running   0          55s
    iomesh-meta-2                                         2/2     Running   0          55s
    iomesh-openebs-ndm-5457z                              1/1     Running   0          2m8s
    iomesh-openebs-ndm-599qb                              1/1     Running   0          2m8s
    iomesh-openebs-ndm-cluster-exporter-68c757948-gszzx   1/1     Running   0          2m8s
    iomesh-openebs-ndm-node-exporter-kzjfc                1/1     Running   0          2m8s
    iomesh-openebs-ndm-node-exporter-qc9pt                1/1     Running   0          2m8s
    iomesh-openebs-ndm-node-exporter-v7sh7                1/1     Running   0          2m8s
    iomesh-openebs-ndm-operator-56cfb5d7b6-srfzm          1/1     Running   0          2m8s
    iomesh-openebs-ndm-svp9n                              1/1     Running   0          2m8s
    iomesh-zookeeper-0                                    1/1     Running   0          2m3s
    iomesh-zookeeper-1                                    1/1     Running   0          102s
    iomesh-zookeeper-2                                    1/1     Running   0          76s
    iomesh-zookeeper-operator-7b5f4b98dc-6mztk            1/1     Running   0          2m8s
    operator-85877979-66888                               1/1     Running   0          2m8s
    operator-85877979-s94vz                               1/1     Running   0          2m8s
    operator-85877979-xqtml                               1/1     Running   0          2m8s
    ```

IOMesh 集群现在已成功部署. 接下来开始设置 IOMesh [Setup IOMesh](setup-iomesh.md).

[1]: http://iomesh.com/charts
[2]: http://www.iomesh.com/docs/installation/setup-iomesh-storage#setup-data-network

## 离线安装

### 准备离线安装包

1. 下载 [IOMesh offline installation package](https://download.smartx.com/iomesh-offline-v0.11.1.tgz).

2. 解压离线安装包
    ```shell
    tar -xf  iomesh-offline-v0.11.1.tgz && cd iomesh-offline
    ```

### 加载 IOMesh 镜像
在每个 Kubernetes node 上加载 IOMesh 镜像，然后根据您的容器运行时和容器管理工具执行相应的镜像加载脚本
<!--DOCUSAURUS_CODE_TABS-->

<!--Docker-->
```shell
docker load --input ./images/iomesh-offline-images.tar
```
<!--Containerd-->
```shell
ctr --namespace k8s.io image import ./images/iomesh-offline-images.tar
```

<!--Podman-->
```shell
podman load --input ./images/iomesh-offline-images.tar
```

<!--END_DOCUSAURUS_CODE_TABS-->

### 使用离线安装包安装 IOMesh

#### 安装 IOMesh

1. 渲染 IOMesh 默认配置文件到 `iomesh.yaml` 中

    ```shell
    ./helm show values charts/iomesh > iomesh.yaml
    ```

2. 配置 `iomesh.yaml`.

   **(必须)** 根据您的网络环境填写 `dataCIDR` 字段.

    ```yaml
    iomesh:
      chunk:
        dataCIDR: "10.234.1.0/24" 
    ```

    **(可选)** 配置集群的部署模式，默认是混闪模式。如果要部署为全闪则需要设则 `diskDeploymentMode` 字段为`allFlash`
   
   ```yaml
   iomesh:
     chunk:
       podPolicy:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname # specific node label's key
                   operator: In
                   values:
                   - iomesh-worker-0 # specific node label's value
                   - iomesh-worker-1
    ```
    更多的 pod affinity 配置规则参考: [pod affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)

3. 安装 IOMesh 集群.

    ```shell
    ./helm install iomesh ./charts/iomesh \
        --create-namespace \
        --namespace iomesh-system \
        --values iomesh.yaml \
        --wait
    ```

    ```output
    NAME: iomesh
    LAST DEPLOYED: Wed Jun 30 16:00:32 2021
    NAMESPACE: iomesh-system
    STATUS: deployed
    REVISION: 1
    TEST SUITE: None
    ```

4. 检查安装结果

    ```bash
    kubectl --namespace iomesh-system get pods
    ```

    ```output
    NAME                                                   READY   STATUS    RESTARTS   AGE
    iomesh-chunk-0                                         3/3     Running   0          102s
    iomesh-chunk-1                                         3/3     Running   0          98s
    iomesh-chunk-2                                         3/3     Running   0          94s
    iomesh-csi-driver-controller-plugin-5b66557959-jhshb   6/6     Running   10         3m20s
    iomesh-csi-driver-controller-plugin-5b66557959-p4m9x   6/6     Running   10         3m20s
    iomesh-csi-driver-controller-plugin-5b66557959-w9qbq   6/6     Running   10         3m20s
    iomesh-csi-driver-node-plugin-6pjpn                    3/3     Running   2          3m20s
    iomesh-csi-driver-node-plugin-dj2cd                    3/3     Running   2          3m20s
    iomesh-csi-driver-node-plugin-stbdw                    3/3     Running   2          3m20s
    iomesh-hostpath-provisioner-55j8t                      1/1     Running   0          3m20s
    iomesh-hostpath-provisioner-c7jlz                      1/1     Running   0          3m20s
    iomesh-hostpath-provisioner-jqrsd                      1/1     Running   0          3m20s
    iomesh-iscsi-redirector-675vr                          2/2     Running   1          119s
    iomesh-iscsi-redirector-d2j4m                          2/2     Running   1          119s
    iomesh-iscsi-redirector-sjfjk                          2/2     Running   1          119s
    iomesh-meta-0                                          2/2     Running   0          104s
    iomesh-meta-1                                          2/2     Running   0          104s
    iomesh-meta-2                                          2/2     Running   0          104s
    iomesh-openebs-ndm-569pb                               1/1     Running   0          3m20s
    iomesh-openebs-ndm-9fhln                               1/1     Running   0          3m20s
    iomesh-openebs-ndm-cluster-exporter-68c757948-vkkdz    1/1     Running   0          3m20s
    iomesh-openebs-ndm-m64j5                               1/1     Running   0          3m20s
    iomesh-openebs-ndm-node-exporter-2brc6                 1/1     Running   0          3m20s
    iomesh-openebs-ndm-node-exporter-g97q5                 1/1     Running   0          3m20s
    iomesh-openebs-ndm-node-exporter-kvn88                 1/1     Running   0          3m20s
    iomesh-openebs-ndm-operator-56cfb5d7b6-gwlg9           1/1     Running   0          3m20s
    iomesh-zookeeper-0                                     1/1     Running   0          3m14s
    iomesh-zookeeper-1                                     1/1     Running   0          2m59s
    iomesh-zookeeper-2                                     1/1     Running   0          2m20s
    iomesh-zookeeper-operator-7b5f4b98dc-fxfb6             1/1     Running   0          3m20s
    operator-85877979-5fvvn                                1/1     Running   0          3m20s
    operator-85877979-74rl6                                1/1     Running   0          3m20s
    operator-85877979-cvgcz                                1/1     Running   0          3m20s
    ```

IOMesh 集群现在已成功部署. 接下来开始设置 IOMesh [Setup IOMesh](setup-iomesh.md).

## 在 OpenShift 容器平台安装 IOMesh

您还可以通过 Red Hat OpenShift 容器平台内的 OperatorHub 安装和使用 IOMesh

### 安装前置步骤

运行下面的脚本，这将安装 IOMesh Operator 的依赖项并为 OpenShift 集群配置 IOMesh 相关设置。 请注意，脚本需要在 oc 或 kubectl 可以访问 OpenShift 集群的环境中执行

```shell
curl -sSL https://iomesh.run/iomesh-operator-pre-install-openshift.sh | sh -
```

### 安装 IOMesh Operator

1. 登录 OpenShift Container Platform，然后打开 OperatorHub 页面。 在OperatorHub页面，选择IOMesh Operator，点击Install，安装IOMesh Operator

2. 依次选择Installed Operators > IOMesh Operator > Create instance > YAML view，然后根据 IOMesh [YAML](https://iomesh.run/iomesh.yaml) 配置 IOMesh. 将 `spec.dataCIDR` 更改为您的存储网络 CIDR

### 安装 CSI Driver

运行以下脚本以安装 IOMesh CSI Driver。 请注意，脚本应在 oc 或 kubectl 可以访问 OpenShift 集群的环境中执行

```shell
curl -sSL https://iomesh.run/iomesh-operator-post-install-openshift.sh | sh -
```
    
IOMesh 集群现在已成功部署. 接下来开始设置 IOMesh [Setup IOMesh](setup-iomesh.md).

# Storage

本文档主要用于展示 k8s 集群内关于持久化存储的使用。

k8s 通过 PV PVC StorageClass 三者的抽象来管理集群的存储。

三者简要关系如下。

PV：PersistentVolume 集群对底层存储的抽象，用于解耦具体存储类型的实现细节
PVC：PersistentVolumeClaim 持久化存储声明，使用者(POD)对所需存储的抽象，用于解耦存储使用者与具体PV的关联，使用者只需提出使用需求(大小、权限)
SC：StorageClass 用于管理 PVC - PV 的关联关系的抽象，相当于一个工厂类，声明 PVC 时只需指定下 StorageClass，集群便会自动创建 PV，同时将 PVC 绑定到 PV

## 静态PV

PV 可以由管理员提前创建，类似于管理员对集群内的存储资源进行管理分配，一个个 PV 就相当于一个磁盘的分区。
创建完成后存于整个集群内部，等待 PVC 的绑定。

## 动态PV

如果所有的 PV 都需要管理员提前创建，当集群规模化以后那会存在巨大的工作量，以及管理混乱的问题，于是 StorageClass 被提出。
基于 StorageClass 可动态创建所需 PV，使用者仅需在声明 PVC 时指定 StorageClass 即可。

## StorageClass

StorageClass 为管理员提供了描述存储"类"的方法，被用于描述如何创建 PV。
每个 StorageClass 都包含 provisioner、parameters 和 reclaimPolicy 字段， 这些字段会在 StorageClass 需要动态制备 PersistentVolume 时会使用到。

**Provisioner**

每个 StorageClass 都有一个制备器（Provisioner），用来决定使用哪个卷插件制备 PV。

## PVC 绑定 PV

持久卷申领（PersistentVolumeClaim，PVC） 表达的是用户对存储的请求。 概念上与 Pod 类似。 
Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）。
同样 PVC 申领也可以请求特定的大小和访问模式 
例如，可以挂载为 ReadWriteOnce、ReadOnlyMany、ReadWriteMany 或 ReadWriteOncePod。

**绑定规则**

当 PVC 绑定 PV 时，需考虑以下参数来筛选当前集群内是否存在满足条件的 PV。
- VolumeMode：主要定义 volume 是文件系统（FileSystem）类型还是块（Block）类型，PV 与 PVC 的 VolumeMode 标签必须相匹配。
- Storageclass：PV 与 PVC 的 storageclass 类名必须相同（或同时为空）。
- AccessMode： 主要定义 volume 的访问模式，PV 与 PVC 的 AccessMode 必须相同。
- Size：主要定义 volume 的存储容量，PVC 中声明的容量必须小于等于 PV，如果存在多个满足条件的 PV，则选择最小的 PV 与 PVC 绑定。
- Selector：通过指定标签 matchLabels 来匹配特定PV，matchExpressions 指定表达式

如上述匹配策略都未能匹配到所需 PV，则会基于系统默认 StorageClass 来创建一个 PV，随后与 PVC 绑定。

***如果 PVC 申领指定存储类为 ""，则相当于为自身禁止使用动态制备的卷***

**一旦绑定关系建立，则 PersistentVolumeClaim 绑定就是排他性的， 无论该 PVC 申领是如何与 PV 卷建立的绑定关系。 
PVC 申领与 PV 卷之间的绑定是一种一对一的映射，实现上使用 ClaimRef 来记述 PV 卷与 PVC 申领间的双向绑定关系。**


## 示例

下述示例展示了 PV 与 PVC 的基本使用。

### 静态制备PV

下面示例展示了使用 Local 模式创建了本地 pv, 同时创建了 pvc，通过 matchlabels 进行 pv 的查找绑定。

```shell
# 创建 pv
kubectl apply -f local-pv.yaml
```

```shell
# 创建 pvc 并绑定
kubectl apply -f local-pvc.yaml
```

```shell
# 查看绑定关系
kubectl get pv
kubectl get pvc
```

尝试对 local-pvc.yaml 进行复制再次编辑 local-pvc-1.yaml，看能否实现一对多绑定。
```shell
# 创建 pvc 并绑定
kubectl apply -f local-pvc-1.yaml
```
最终通过 kubectl get pvc 可以看到一直处于 pending 状态。

### 静态制备PV，通过StorageClass进行绑定

```shell
kubectl apply -f local-storage-class.yaml
```

```shell
kubectl get pv
kubectl get pvc
kubectl get sc
```
可以看到 pvc 一直处于 pending 状态，是因为local-storage-class 指定了 volumeBindingMode 为 WaitForFirstConsumer，
只有当有 pod 进行调度时才会执行绑定。故创建如下 pod 进行测试。
```shell
kubectl apply -f local-storage-pod.yaml
```
```shell
# 查看调度 可看到已调度到 pv 声明时指定的节点亲和性的节点上
kubectl get pods -o wide
```
```shell
# 删除并重建, 可看到还是调度到相同节点
kubectl delete -f local-storage-pod.yaml
# 等待删除完成后重建
kubectl apply -f local-storage-pod.yaml
kubectl get pods -o wide
```

### NFS PV

#### 安装 NFS 服务器

[安装参考](https://kubesphere.io/zh/docs/v3.4/reference/storage-system-installation/nfs-server/)
[防火墙设置1](https://cloud.tencent.com/developer/article/2106196)
[防火墙设置2](https://cloud.tencent.com/developer/article/2106136)

#### 静态制备：创建 PV PVC StorageClass Pod

```shell
kubectl apply -f nfs-storage-class-staic.yaml
```

#### 动态制备

**安装 nfs-client-provisioner**

[安装参考](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
```shell
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=120.79.152.224 \
    --set nfs.path=/mnt/nfs \
    --set image.repository=eipwork/nfs-subdir-external-provisioner
```

安装完成后查看 pod 是否正常，是否有对应的 StorageClass
```shell
kuebctl get sc
NAME                   PROVISIONER                                     RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
nfs-client             cluster.local/nfs-subdir-external-provisioner   Delete          Immediate              true                   57s
```

测试能否自动创建PV，若能正常绑定则说明 nfs-provisioner 以及 nfs-client(StorageClass) 能正常工作。
```shell
kubectl apply -f nfs-pvc.yaml
kubectl get pvc 
```

测试 POD 能否自动创建pvc pv
```shell
kubectl apply -f nfs-client-pod.yaml
```

### Longhorn

Longhorn 是 Kubernetes 的一个开源分布式块存储系统。

[官网](https://github.com/longhorn/longhorn/tree/master)

安装
```shell
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```





---
layout: post
title:  "Sotrage in kubernetes"
date: 2017-08-22 15:10:00 +0000
categories: Cloud
---

### Introduction
所有的 container 系统，包括docker，kubernetes 都有一个很重要的部分就是storage。  Storage 的概念在单node 的使用没有特别明显的作用，但是如果是cluster，就是一个不可或缺的部分。 因为container 在cluster 平台中是随机运行在其中的一个node上的。所以persistent的数据如果依赖于某一个node的文件系统，期行为将不可预料。 所以docker，kubernetes 都有相应的storage概念。 

### container 系统的 storage 

相对于IAAS 平台， container 系统中的storage 相对来说要稍微有些不一样。 对于 Iass 平台，比如openstack， 每一台机器的生命周期相对来说会稳定很多。 所以相对应的storage 比如 cinder 提供的 block storage 不是经常需要变化。 但是到了container 环境， 每一个 container 的启动和结束会迅速很多。这样对于sotrage 动态加载要求就高了很多。


#### kubernetes 中的stroage 
有两个基本的概念 需要先了解的，一个是 persistent volumn 还有一个是persistent volumn claim。 官方定义如下  
```
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.


A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., can be mounted once read/write or many times read-only).
```

从上面的描述可以看出来， persistent volume 是使用个各种 volumn plugins 来创建的一个个 cluster 的资源。 在云环境中， 所有的东西都可理解为资源， 在，所以这里的PV 就可以理解为由cluster administrator提供出来的一个个sotrage 资源。 通常这个需要提前准备好，这样po 才可以使用

PVC 则是另一个抽象，PVC 定义了一个 pod 需要多大的一个资源。 PVC 是从使用的角度就看待问题，因为用户在使用的时候并不知道cluster 如何提供 storage， 或者提供什么样的stroage。 但是对于我这个pod， 我是需要这样一个资源的。 在部署的时候，如果资源需求得不到满足，会报相应的错误。


##### pvc 是如何 和 pv 关联的 

默认情况下， PVC 会根据 storage class 的定义来找到符合自己需求的定义。 如果没有定义storage class， 则会选择一个默认的 来bind。 官方定义如下 

```
hile PersistentVolumeClaims allow a user to consume abstract storage resources, it is common that users need PersistentVolumes with varying properties, such as performance, for different problems. Cluster administrators need to be able to offer a variety of PersistentVolumes that differ in more ways than just size and access modes, without exposing users to the details of how those volumes are implemented. For these needs there is the StorageClass resource.

A StorageClass provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called “profiles” in other storage systems.
```

从上面的描述可以看到 StorageClass 仅仅是提供了一个让 adminstorator 来定义它提供的storage 类型， 比如快速，稳定等。 但是用户无法明确指定storage 的binding 关系。


##### PV 的使用方式  

根据下面的描述方式， kubernetes 支持静态的和动态的使用pv.  静态的其实比较简单 ，动态的会使用很多。 按照官方的描述，admin 可以动态的创建一个PV 给PVC 使用。

···
Static

A cluster administrator creates a number of PVs. They carry the details of the real storage which is available for use by cluster users. They exist in the Kubernetes API and are available for consumption.
Dynamic

When none of the static PVs the administrator created matches a user’s PersistentVolumeClaim, the cluster may try to dynamically provision a volume specially for the PVC. This provisioning is based on StorageClasses: the PVC must request a class and the administrator must have created and configured that class in order for dynamic provisioning to occur. Claims that request the class "" effectively disable dynamic provisioning for themselves.
···


##### PV 的重用

在定义PV 的时候，可以定义PV 的重用属性。 kubernetest 支持一下集中属性 
- Retaining ： 当 PVC 被删除的时候，这个PV 会继续保留，暂时不会被重用
- Recycling ： 当 PVC 被删除的时候，这个PV 会被其他 PVC 使用
- Deleting ： 当 PVC 被删除的时候，PV 也会被删除


##### PVC 的挂载方式 

- ReadWriteOnce – the volume can be mounted as read-write by a single node
- ReadOnlyMany – the volume can be mounted read-only by many nodes
- ReadWriteMany – the volume can be mounted as read-write by many nodes







### Introduction
所有的 container 系统，包括docker，kubernetes 都有一个很重要的部分就是storage。  Storage 的概念在单node 的使用没有特别明显的作用，但是如果是cluster，就是一个不可或缺的部分。 因为container 在cluster 平台中是随机运行在其中的一个node上的。所以persistent的数据如果依赖于某一个node的文件系统，期行为将不可预料。 所以docker，kubernetes 都有相应的storage概念。 

### container 系统的 storage 

相对于IAAS 平台， container 系统中的storage 相对来说要稍微有些不一样。 对于 Iass 平台，比如openstack， 每一台机器的生命周期相对来说会稳定很多。 所以相对应的storage 比如 cinder 提供的 block storage 不是经常需要变化。 但是到了container 环境， 每一个 container 的启动和结束会迅速很多。这样对于sotrage 动态加载要求就高了很多。


### kubernetes 中的stroage 
有两个基本的概念 需要先了解的，一个是 persistent volumn 还有一个是persistent volumn claim。 官方定义如下  

```
A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.


A PersistentVolumeClaim (PVC) is a request for storage by a user. It is similar to a pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., can be mounted once read/write or many times read-only).
```

从上面的描述可以看出来， persistent volume 是使用个各种 volumn plugins 来创建的一个个 cluster 的资源。 在云环境中， 所有的东西都可理解为资源， 在，所以这里的PV 就可以理解为由cluster administrator提供出来的一个个sotrage 资源。 通常这个需要提前准备好，这样po 才可以使用

PVC 则是另一个抽象，PVC 定义了一个 pod 需要多大的一个资源。 PVC 是从使用的角度就看待问题，因为用户在使用的时候并不知道cluster 如何提供 storage， 或者提供什么样的stroage。 但是对于我这个pod， 我是需要这样一个资源的。 在部署的时候，如果资源需求得不到满足，会报相应的错误。


#### pvc 是如何 和 pv 关联的 

默认情况下， PVC 会根据 storage class 的定义来找到符合自己需求的定义。 如果没有定义storage class， 则会选择一个默认的 来bind。 官方定义如下 

```
hile PersistentVolumeClaims allow a user to consume abstract storage resources, it is common that users need PersistentVolumes with varying properties, such as performance, for different problems. Cluster administrators need to be able to offer a variety of PersistentVolumes that differ in more ways than just size and access modes, without exposing users to the details of how those volumes are implemented. For these needs there is the StorageClass resource.

A StorageClass provides a way for administrators to describe the “classes” of storage they offer. Different classes might map to quality-of-service levels, or to backup policies, or to arbitrary policies determined by the cluster administrators. Kubernetes itself is unopinionated about what classes represent. This concept is sometimes called “profiles” in other storage systems.
```

从上面的描述可以看到 StorageClass 仅仅是提供了一个让 adminstorator 来定义它提供的storage 类型， 比如快速，稳定等。 但是用户无法明确指定storage 的binding 关系。


#### PV 的使用方式  

根据下面的描述方式， kubernetes 支持静态的和动态的使用pv.  静态的其实比较简单 ，动态的会使用很多。 按照官方的描述，admin 可以动态的创建一个PV 给PVC 使用。

···
Static

A cluster administrator creates a number of PVs. They carry the details of the real storage which is available for use by cluster users. They exist in the Kubernetes API and are available for consumption.
Dynamic

When none of the static PVs the administrator created matches a user’s PersistentVolumeClaim, the cluster may try to dynamically provision a volume specially for the PVC. This provisioning is based on StorageClasses: the PVC must request a class and the administrator must have created and configured that class in order for dynamic provisioning to occur. Claims that request the class "" effectively disable dynamic provisioning for themselves.
···


#### PV 的重用

在定义PV 的时候，可以定义PV 的重用属性。 kubernetest 支持一下集中属性 
- Retaining ： 当 PVC 被删除的时候，这个PV 会继续保留，暂时不会被重用
- Recycling ： 当 PVC 被删除的时候，这个PV 会被其他 PVC 使用
- Deleting ： 当 PVC 被删除的时候，PV 也会被删除


#### PVC 的挂载方式 

- ReadWriteOnce – the volume can be mounted as read-write by a single node
- ReadOnlyMany – the volume can be mounted read-only by many nodes
- ReadWriteMany – the volume can be mounted as read-write by many nodes


#### Volumn Template

如果先创建volumn 以及 volumnclaim 那么就可以在replication set 中共享数据
如果想要每一个 pod 都可以自动的创建一个 volumn 那么就需要使用 volumn template ， 这个好像只有stateful set 里面支持


### Gluster 作为 storage

Gluster 是一个cluster 服务，下面的例子只使用了一个standlone的服务

#### 安装 Gluster 服务
centos 7 可以比较方便的安装gluster ，在每一个server 上执行如下命令
```bash
yum install centos-release-gluster
yum --enablerepo=centos-gluster*-test install glusterfs-server
yum install glusterfs-server
systemctl enable glusterd
systemctl start glusterd
systemctl status glusterd
```

#### 手动创建 brick

##### 创建磁盘
```
#格式化
mkfs.xfs -f -i size=512 /dev/mapper/mpathb

#创建挂在目录
mkdir -p /data

# fstab 挂载
echo '/dev/mapper/mpatha /data/brick1 xfs defaults 1 2' >> /etc/fstab

# mount
mount -a && moun
```



##### 创建GV

```bash
mkdir /date/brick1/gv
gluster volume create gv0  10.170.13.153:/data/brick1/gv0
# show volume
gluster volume info
```


##### 在kubernetest 中使用 gluster ##



```bash
sudo yum install -y glusterfs-fuse

#测试mount 
sudo mount  -t glusterfs 10.170.13.153:/gv0 /mnt/gluster
```


#### 使用 heketi 来管理 gluster

使用heketi 来管理gluster 非常有用， 这样可以实现kubernetes 的动态provisioning 

##### 原理 

heketi 利用 root/sudo 的权限来执行 ssh 命令。 当配置完 key 以后，heketi 会从 pvcreate -> vgcreate -> lvcreate -> gluster create 一套全部执行。 大大减少我们使用时候的复杂度

##### 安装
我是吧 heketi 和 gluster 安装在同一台机器上， 装机之前需要先在 lvm的配置中吧 filter 配置好，不然会看不到磁盘

```bash
yum install heketi heketi-client -y
```

##### 准备 key 

```bash
#create a new key pair
ssh-keygen -f /etc/heketi/heketi_key
chown heketi:heketi /etc/heketi/heketi_key*
ssh-copy-id -i /etc/heketi/heketi_key.pub root@localhost
```
创建好key 以后， heketi 就可以无密码登录 localhost 了

##### 配置和启动 heketi

 每一条的具体含义在这里  [配置](https://github.com/heketi/heketi/wiki/Running-the-server)。简单介绍一下就是
 

```
vi /etc/heketi/heketi.jso
```

```json
...

 "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",  #very imporatnt 

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/etc/heketi/heketi_key",
      "user": "root"
    },

...

```


##### 准备 topology 文件 
因为我的cluster 只有一台机器，所以配置很简单，如下。 有一个比较不太能接受的就是居然没有清除命令。 如果配置有问题，只能一个一个的自己删除

```json
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "localhost"
                            ],
                            "storage": [
                                "10.170.13.153"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/mapper/mpathc",
                        "/dev/mapper/mpathd"
                    ]
                }
            ]
        }
    ]
}

```

```bash
#load it, please replace to your json file path
heketi-cli topology load --json=topo.json

```

##### 测试 

```
#create from heketi
heketi-cli volume create --size=1024 --durability=none

#check volumn information 
gluster volume info

# check lvm information 
lvscan
pvscan
```

##### Integration with kubernete

###### Create a storage class 

Create a storage class in this platform as example in [document](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#glusterfs) . "volumetype = none" means it is an destrbute volumn and no replication. 

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.170.13.153:8080"
  gidMin: "40000"
  gidMax: "50000"
  volumetype: "none"  

```

```bash
kubectl create -f storageClass.yaml
```

###### Create a pvc to test it 
This PVC definition trys to claim a 20G block storage using "slow" stroage-class

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gluster-test
  annotations:
     volume.beta.kubernetes.io/storage-class: slow
spec:
  accessModes:
  - ReadWriteMany
  resources:
     requests:
       storage: 20Gi

```
```bash
#create
kubectl create -f storageClass.yaml

#show the bind
kubectl get pvc

```



#### reference 
- [Setup volumn in gluster from official document](https://gluster.readthedocs.io/en/latest/Administrator%20Guide/Setting%20Up%20Volumes/)
- [Complete Example of Dynamic Provisioning Using Dedicated GlusterFS](https://docs.openshift.org/latest/install_config/storage_examples/dedicated_gluster_dynamic_example.html)
- [这一篇文档里面还讨论了gluster的使用 以及 selinux的使用，感觉很不错](https://docs.openshift.com/container-platform/3.5/install_config/storage_examples/gluster_example.html)
- [glusterfs分布式文件系统详细原理](http://blog.csdn.net/yujin2010good/article/details/75268877)
- [http://jeremy-xu.oschina.io/2016/07/25/%E5%88%9D%E8%AF%86glusterfs/](http://jeremy-xu.oschina.io/2016/07/25/%E5%88%9D%E8%AF%86glusterfs/)
- [heketi 使用](https://github.com/heketi/heketi/wiki/Create-a-volume)
- [heketi 配置](https://github.com/heketi/heketi/wiki/Running-the-server)

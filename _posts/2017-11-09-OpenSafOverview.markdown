---
layout: post
title:  "Opensaf Overview 翻译"
date: 2017-11-10 15:10:00 +0000
categories: SourceCode, Kafka
---

2 Overview
2.1 OpenSAF Overview
OpenSAF is a suite of HA middleware which implements the Service Availability Forum™ (SAF) interface specification. The SAF is a consortium of industry-leading communications and computing companies working together to develop and publish high availability and management software interface specifications.
In addition to the services that implement the SAF interface, OpenSAF contains complementary services which are required in a complete high-availability system solution.

OpenSAF是一个 实现了SAF interface specification 的HA 的中间件。 SAF 是个由各大公司组成的一个集体，致力于开发和发布高可用和易于管理的软件标准
除了实现SAF的interface，OpenSAF 还包含了一些补充的services，而这些service是一个完整的HA system所必须的。

The set includes:

**SA Forum-compliant services:**
- Availability Management Framework
- Checkpoint Service
- Global Lock Service
- Event Distribution Service
- Log Service
- Message Queue Service
- Information Model Management Service
- Notification Service
- Platform Management Service
- Software Management Framework
- Cluster Management Service

**OpenSAF complementary services:**
- Message Distribution Service
- Message-Based Checkpoint Service
- OpenSAF Base

SAF-compatible services, as well as the complementary services, are described in detail in this manual.
All these services have versioning infrastructure to support In Service Upgrade.
####OpenSAF Services
**Architectural Overview**
OpenSAF system architecture defines two types of nodes in the OpenSAF cluster:
- System Controller Node
- Payload Node
**System Controller Node** - A designated node within an OpenSAF cluster that hosts the centralized functions of various OpenSAF services. The system controller node also includes the management access point for the entire cluster. Application components may or may not be configured on a system controller node depending on the target OpenSAF application's design. For high availability purposes, you can configure several system controller nodes. OpenSAF will utilize the system controller nodes using the 2N redundancy model, where one node will be ACTIVE, another node will be STANDBY, and the rest of the controller nodes (if any) will be spares.

Controller Node – 在OpenSAF cluster 中包含了一些集中管理的 Service。 此外它还包含了整个cluster的管理接口。 更具OpenSAF应用程序的设计， Controller node可以选择包含或者不包含Application的service。 作为高可用性的需求， Controller node 可以 2N redundancy 模式， 在这种模式下， 一个是ACTIVE 而另一个是STANDBY， 而剩下的都是用来做备份的。


**Payload Node** -All nodes which are not system controller nodes are termed payload nodes. These nodes contain node-scoped functions of various OpenSAF services. They are expected to host the target OpenSAF application components.

Payload Node – 所有不是controller的node都是payload node。 挺和谐node 包含提供主要的application服务。

The following section gives an overview on different components of the OpenSAF Services and provides brief insights into each service.
The following figure illustrates the main software layers that constitute the OpenSAF software:

3.1.1 OpenSAF Services
The following table lists and briefly explains the OpenSAF services that implement the Service Availability Forum (SAF) Application Interface Specification (AIS).
OpenSAF Service Name
Corresponding SAF AIS Service(s)
Description
Availability Management Framework (AMF)
Availability Management Framework (AMF)  
This service provides a standardized means to model system components and standardized mechanisms for monitoring, fault reporting, fault recovery and repair of components.  
这个service 提供的标准的模型化component的手段，标准化的监控机制，错误报告机制，错误回复机制。
Cluster Management Service (CLMSv)
Cluster Management Service(CLM)
It further provides functionality that oversees cluster nodes as they join and leave the cluster.
这个service提供的监控cluster node 的加入，删除的功能
Checkpoint Service (CPSv)
Checkpoint Service (CKPT)
This service oversees the life and integrity of entities called checkpoints. Active components write to checkpoints so that stand-by components recover the last known good state while turning active. CPSv coordinates the creation and deletion of checkpoints and maintains the checkpoint inventory within a cluster.

这个service是用来保证系统恢复的point的。 Active的component负责写入这个checkpoint， 然后stand、by component 就可以根据这个check point 来回复最新的那个已知的健康状态。 这个service 会在整个cluster 里面负责 协调 checkpoint的创建，删除以及checkpoint的仓库。 
Message Queue Service (MQSv)
Messaging Service (MSG)
This service provides a standardized means for distributed applications to send messages among themselves. MQSv oversees entities called queues and queue groups and is capable of preserving unread messages if a reader process dies.

这个service提供的标准的方法给那些分布式的应用互相发送消息。 这个service会监控 实体的 queue 和 queue group 以及在reader的process 出现问题的时候，监控queue的容量

Event Distribution Service (EDSv)
Event service (EVT)
This service provides a standardized means to publish events and to subscribe to events anywhere in a cluster.
这个service 提供了标准的方法去发布和订阅消息。

Global Lock Service (GLSv)
Lock Service (LCK)
This service provides a means to control access to a cluster resources by competing distributed clients.
这个service提供了一个用来空间资源使用的全局锁
Log Service (LOGSv)
Log Service (LOG)
This service provides a standardized means for applications to express and forward log records through well-known streams that leads to an associated named file.
提供了标准的方法给应用来 记录以及forwoard log 给常用的连接到named file的 streams。 
Information Model Management Service (IMMSv)
Information Model Management Service (IMM)
The IMM Service delivers the requested operations to the appropriate AIS Services or applications (referred to as object implementers) that implement these objects for execution.
IMM service 会吧 操作发送给对应的 application 或者 service (这个应该不是一个queue，而是同步的操作)
Notification Service (NTFSv)
Notification Service (NTF)
The Notification Service is used by a service user to report an event to a peer service user. A notification producer generates notifications. A consumer consumes notifications that were generated by producers. A subscriber for notifications gets notifications forwarded as they occur.
这个service 用来将一个 event report 给另外一个 service。 Notification的producer 会生成 notification 然后 consumer 会consume 这个 notification。 Subscriber会在notification发生的时候得到通知。

Software Management Framework (SMF)
Software Management Framework (SMF)
The Software Management Framework maintains the information about the availability, the contents, and the deployment of different software bundles in the SA Forum system and coordinates the upgrades.

这个 frameword 负责维护 可用性的 信息， 已经在 SAF 的系统里面部署不同的软件包， 此外还负责协调升级功能

Platform Management Service (PLMSv)
Platform Management (PLM)
Platform Management Service provides a logical view of the hardware and low-level software of the system. PLM Service also provides objects that are administratively configurable.

这个service 负责提供了一层 硬件以及底层软件的逻辑抽象。此外它还提供的用语 管理员关系的配置对象

Table 1: SAF Compliant OpenSAF Services
Further details about each of these services will be given in the following sections.
The following table lists and briefly describes complementary OpenSAF services.
OpenSAF Service Name
Description
Message Based Checkpointing Service (MBCSv)
This service provides message-based checkpointing between an active and one or more stand-by components Message-Based Checkpointing Service.

这个service提供了基于消息的 checkpoint (具体怎么使用不太明白)

Message Distribution Service (MDS)
The Message Distribution Service (MDS) provides high-performance, reliable message distribution services. OpenSAF services and user applications invoke the services provided by MDS through an API which is exposed in the form of a library that can be linked to a process. MDS supports intra-process, inter-process as well as inter-node communication. Further details about MDS is given in section Message Distribution Service. 
这个服务提供了高可用，高性能，可到了消息同步服务。 OpenSAF service 以及用户应用可以通过MDS 的 API  来获得具体的服务。

Table 2: OpenSAF-Complementary Services
Further details about these services will be given in the following sections.   
3.2 Distribution of OpenSAF Services
Many OpenSAF services are subdivided into subparts. All subparts together form one OpenSAF service. The subparts may run together on one node or may even be distributed among several nodes. This section will provide more details.

许多OpenSAF的service会被分成多个子模块，这些子模块在一起成为一个OpenSAF的service。 这个子模块可以在同一个node或者分布在多个node中

The following figure provides an overview on how OpenSAF services are distributed in a system.

3.2.1 OpenSAF Service Directors
A Director for a particular OpenSAF service manages and coordinates key data among the other distributed sub-parts of that service. The Director is located on a controller node and is implemented with a 2N redundancy model. Active Director checkpoints the data to StandBy Director using MBCSv. Directors on spare system controller nodes receive checkpoint data until they are given an active or standby assignment.

一个 服务的Director 负责管理和协调 所有子系统之家的 关键数据。 Director 存活在 controller node 以 2N 的redundancy model 。 Active Director 会负责将 checkpoint 数据会写到 MBCSy中。 在冗余的 Director 会接受这些checkpoint data 直到有成为 active 或者 standby 系统

A director communicates with Node Directors that are located on blades in a system. Node Directors handle node-scoped activities such as messaging with the central Director or with the local OpenSAF service agent.

Director 会和存活在 blades 中的node的Director 通信。 Node Director 负责处理node 级别的事务， 比如和central Director 通信的数据或者 Local的service agent的通信数据。

The OpenSAF service agent makes service capabilities available to clients such as customer applications, by way of shared linkable libraries that expose well defined APIs.
The following figure illustrates Directors and their interaction with other OpenSAF service subparts.

OpenSAF service 的agent 负责将各种 SAF的服务的提供给 client， 具体的方法则是提供 linkable的libraries。


The following OpenSAF service Directors exist in an OpenSAF system:
Availability Director
Checkpoint Director
Cluster Management Director
Global Lock Director
Message Queue Director
Information Model Management Director
Software Management Framework Director (SMFD)
A more detailed description of these Directors will be given in the description of the respective OpenSAF service later in this manual.
3.2.2 OpenSAF Service Servers
An OpenSAF Service specific server provides central intelligence for a particular OpenSAF service, but unlike with OpenSAF service Directors, there is no corresponding Node Director for an OpenSAF service Server. Instead the server communicates directly with OpenSAF service Agents.
OpenSAF service server 为某一个具体的 OpenSAF提供 central intelligence。 和 service Directors 不同的是，在这里没有一个相对的 Node Director 。 取而代之的是它会直接和 OpenSAF agent 通信 ( 个人理解 所谓的 server 也就是一种形式，也是一种服务，因为不需要涉及到 node 相关的底层操作，所有不需要有一个 NODE director 与之合作)

These Servers are implemented in a 2N redundancy fashion. Active Server checkpoints the data to StandBy Server using MBCSv. Servers on spare system controller nodes receive checkpoint data until they are given an active or standby assignment. The following figure illustrates the role of OpenSAF service Servers in a system.

Server 也是用2N redundancy的方式实现。 Active Service 也会将 checkpoint 信息写入 MBCSy 中。 


The following OpenSAF Servers exist in a system:
Cluster Membership Server
Distributed Trace Server
Event Distribution Server
Log Server
Notification Server
Platform Management Server

A more detailed description is given together with the description of the respective OpenSAF service later in this manual.
3.2.3 Sample Applications
The OpenSAF software suit contains a set of sample applications and sample make files that ease the development of customer applications. They illustrate the use of the various APIs and serve as a good starting-point to develop your own applications.
OpenSAF System Model

在 OpenSAF 的软件套件中 包含了一系列的sample application 来帮助简化开发流程。这些例子展示了如何使用各种各样的 API 以及如何开始开发你自己的application

The OpenSAF System description provides an easy and flexible way to describe, create, and configure a system using XML syntax. This makes it easy to view hierarchical information in a structured way.

OpenSAF 系统标书文档提供了一个简单， 灵活通过XML的方式来描述，创建以及配置系统。 这样可以比较容易观察 整个系统的 层级信息

OpenSAF system definition provides a set of XML clauses that describe a system and its default configuration. Together, these XML clauses make up the system configuration file.

OpenSAF system definition 提供的一组 XML 语句来描述系统以及系统配置。 将这些组合在一起就成为了一个系统的 configuration

For arriving at default system configuration the nodes in the system has to be specified as argument to the IMM utility tool immxml-clustersize, and yet another IMM utility immxml-configure generates the default imm.xml file by walking through all service specific configuration templates, classes and objects files.

为了创建默认了系统配置，系统中的node需要在 IMM utility tool 中创建configuraiton并且生成imm.xml

The system is visualized by breaking it into three node groups
Only System controller nodes
Only Payload nodes
All nodes ( controller nodes and payload nodes )
All the OpenSAF service directors are modeled in a 2N redundancy model, and typically runs only on controller nodes. For this reason SG (object of class SaAmfSG) with a DN name safSg=2N,safApp=OpenSAF hosts all the service units hosted by node group safAmfNodeGroup=SCs.
Similarly all the OpenSAF service node directors are modeled in No-Redundancy model and runs on all the nodes including controller nodes. For this reason SG with DN name safSg=NoRed, safApp=OpenSAF hosts all the service units hosted by node group safAmfNodeGroup=AllNodes.
3.3 SAF-Compliant OpenSAF Services
This section briefly describes those OpenSAF services which implement standard SAF services. For each service an architectural overview and a functional description is given. Furthermore, a reference to user manuals is given where you can find more detailed information.

下面简单介绍一下这些OpenSAF的service。 具体的介绍在每个model中都有相对应的介绍。
3.3.1 Availability Management Framework (AMF)
The Availability Management Framework (AMF) provides the following functionality:
Leverage the SAF "Information Model Management "
Honor the Availability Management Framework" API
The Availability Management Framework Director (AMFD) maintains a software system model database which captures SAF described logical entities and their relationships to each other. The software system model database is initially configured from data contained in the System Description file. Through time the system model will modify due to changing system realities and administrative actions.
AMF 提供了两个功能，如上所述。 AMFD 维护了一个 software system model 数据库。 这个数据库总结了 SAF 的 逻辑数据以及相互之间的关系。 这些software system model 数据的初始配置是从System Description File 中来的。 随后这些system model 可能会因为 系统的改变或者管理员的配置有所变化。

The SAF logical entities configured in the information management model include components which normalize the view of physical resources such as processes, drivers or devices. Components are grouped into Service Units according to fault dependencies that exist among them. A Service Unit is also scoped to one or more (physical) fault domains. Service Units of the same type are grouped into Service Groups (SG) which exhibit particular redundancy modeling characteristics. Service Units within a SG are assigned to Service Instances (SI) and given a High Availability state of active and standby.

在Information management model 中配置的逻辑节点包含了各种物理资源的标准化以后的逻辑模块， 例如 process， drivers 或者 devices。 这些模块会根据依赖关系被分组成为 Service units. 一个Service unit 也会被划分到一个或者多个 fault domain。 同一个类型的Service Unit 组成在一起成为一个 Service Group， 每一个group 会一同一种 冗余模式展现给用户。  Service Unit会被指派给 Service Instance 并且提供相对应的状态。

Further functionality provided by AMFD includes:
Automatic and administrative means to instantiate, terminate and restart resources
Automatic and administrative means to manage or reflect Service Group, Service Unit, Service Instance and Resource state
Administrative means to perform switch-over
Administrative means to lock and unlock nodes
Heartbeat and event subscription schemes for fault detection, isolation and identification
Health-check services to probe and prevent system trauma that lead to faults
Fault recovery mechanisms to fail-over SIs which maintain service availability in case of system trauma
Fault repair mechanisms to restore failed components
The AMF itself cannot be a single-point of failure. It provides its own internal scheme and mechanisms to protect itself from its own failure.

AMF 不能成为整个系统的 single-point failure。它提供了自己的机制来避免拒绝服务。

3.3.1.1 Architecture
The AMF  is comprised of the following main software components:
Availability Management Framework Director (AMFD)
Availability Management Framework Node Director (AMFND)  
Availability Management Framework Agent (AMFA)
Availability Management Framework Process
3.3.1.1.1 Availability Management Framework Director
The Availability Management Framework Director (AMFD) maintains the most abstract portions of the Software System Model, such as  service groups, service instances and service nodes.

AMFD 维护在 Software System Model 中抽象级别最高的部分。 比如 service group， service instance 以及 service nodes

Its main tasks include fault detection, isolation and recovery procedures as defined in the SAF AMF. Any problems and failures on a component that cannot be handled locally, are promoted to the Availability Director which controls and triggers the isolation of the affected component and, if possible, the activation of a stand-by component.
它负责维护一些task 包括 错误检测，隔离，以及在SAF AFM中定义的错误恢复。 所有无法再本地恢复的错误都会被promote 到 Availability Director 中处理，他会决定是不是隔离这些模块，或者直接让stand-by 模块来处理。
3.3.1.1.2 Availability Management Framework Node Director
The Availability Management Framework Node Director (AMFND) resides on each system node and its main task is to maintain the node-scoped part of the software system model described above.

AMFND 存活在在所有的系统node中，他主要的任务就是处理没有 软件系统模型中描述的部分

The AMFND coordinates local fault identification and repair of components and furthermore facilitates any wishes it receives from the AMFD.

AMFND 协调本地的错误并且修复components，此外还会出息来自于AMFD的请求。

The AMFND watches for components arriving or leaving the system and summarizes this information in a Service Unit (SU) presence state, and keeps the AMFD informed about the current status and changes. The AMFND is capable of disengaging, restarting and destroying any component within its scope. This may occur according to AMFD instructions or as a result of an administrative action or automatically triggered by policies.

AMFND 会监控 模块的 键入以及离开并且将这些信息总结在 Service unit presence state 中，并且将星系通知给 AMFD。 AMFND 可以隔离，重启，删除任何在他的scope下的component。 当然这种情况一般是根据AMFD的 instruction下发生， 由管理员出发或者由policies 触发
 

3.3.1.1.3 Availability Management Framework Agent
The Availability Management Framework Agent (AMFA) is a linkable library that exposes the SAF APIs to applications. Its task is to convey requests from the AMFND or the AMFD through the AMFND to the application and vice versa.
AMFA 是一个 library， 提供了 SAF 的API。 主要的工作就是传达从 AMFND 或者 AMFD 来的request，或者将消息传回给 AMFND 或者 AMFD

3.3.2 Checkpoint Service
The Checkpoint Service (CPSv) implements the SAF Checkpoint Service. It provides check-pointing of data in a manner which is equivalent to hardware shared memory between nodes.

CPSv实现了SAF中的 Checkpoint Service。它提供了一个类似在nodes中共享内存的方式。
3.3.2.1 Basic Functionality
The CPSv maintains a set of replicated repositories called checkpoints. Each checkpoint may have one or more replicas within the scope of a cluster. At most, one replica per checkpoint may exist on one node within a cluster.

CPSv 维护了一系列的 replicated repository，这些repository被叫做 checkpoint。 每一个 checkpoint 可能在cluster中拥有一个或者多个 replica。 最多的情况下是每一个node中都有相应的备份。

Each checkpoint comprises one or more sections which can be dynamically created or deleted. The CPSv does not encode the data written into checkpoint sections. If checkpoints are replicated on heterogeneous nodes, for example nodes with different endian architecture, you must make sure that the data can be appropriately interpreted on all nodes.

每一个checkpoint 拥有一个或这个多个 可以被动态创建或者删除的section。 CPSy直接将这些数据放到section中(这个应该是为了速度考虑，和KAFKA的设计很像)。 如果checkpoint 在异构的node中被 replicated， 那么application必须要保证这些数据可以在这些node中被正确解读。

The CPSv service supports the following two types of update options:
Asynchronous update option
Synchronous update option
In the case of asynchronous update option, one of the replicas is designated as the active replica. Data is always read from the active replica and there is no guarantee that all the other replicas contain identical data. A write call returns after updating the active replica.

如果是异步的update， 其中一个 replica 会作为active的replica。 数据永远都是从 active中读取，这时是无法保证所有的其他replica都完全一样。 写的话在完成了active 的replica以后就会完成。
In the case of synchronous update options, the call invoked to write to the replicas returns only when all replicas have been updated, i.e. either all replicas are updated or the call fails and no changes are made to the replicas.
如果是同步的update，则只有当所有的replica都被update以后才会返回

The CPSv supports both collocated and non-collocated checkpoints. In case of checkpoints opened with collocated and asynchronous update option, it is up to the application to set a checkpoint to the active state. In all other cases the CPSv itself handles which checkpoint is currently active.

CPS 支持

The Checkpoint Service defined by SAF does not support hot-standby. This means that the currently stand-by component is not notified of any changes made to the checkpoint. When the stand-by component gets active, it has to iterate through the respective checkpoint sections to get up-to-date. To overcome this drawback, the CPSv provides additional, non-SAF APIs which help to notify the stand-by component of changes and thus facilitate the implementation of a hot-stand-by

SAF 定义的Checkpoint Service 不支持 hot-standby。 也就是说现在的stand-by component 不会受到任何checkpoint 的跟新通知。 当stand-by component 被激活以后， 它必须要迭代的检查每一个checkpoint 的 section 已得到更新。 为了解决这个问题， CPSv 还提供了一个辅助的非SAF API的方式来帮助notify standby ， 以解决如何实现 hot-standby的问题。
3.3.2.2 Architecture
The CPSv service consists of the following subparts:
Checkpoint Director (CPD)
Checkpoint Node Director (CPND)
Checkpoint Agent (CPA)
3.3.2.2.1 Checkpoint Director
The Checkpoint Director runs on the active system controller node. Its main tasks are:
Generating a unique ID for each new checkpoint created by applications
Maintaining the list of nodes on which replicas of a particular checkpoint exist
Selecting the Checkpoint Node Director (CPND) which oversees the active replica for each checkpoint
Coordinating the creation and deletion of checkpoints
Maintaining a repository for the CPSv policy and configuration-related information
There is an active and a stand-by CPD running respectively on the two system controller nodes. CPD uses the OpenSAF Message based Checkpoint Service to keep the two synchronized and available for failover situations.

Checkpoint Director 运行在 active 的 controller node 中。主要的任务是
为每一个application的checkpoint生成全局唯一的ID
为active的replica选择一个CPND来监控它
协调checkpoint的创建和删除
维护CPSv policy的repository并为之创建相关信息
CPD会在两个controller node 中分别运行active 和 standby。 CPD 使用 OpenSAF Message based Checkpoint Service 来保证两个服务node之间的同步。


3.3.2.2.2 Checkpoint Node Director
The Checkpoint Node Director (CPND) runs as process both on payload blades and on the two system controller nodes. Its tasks are:
Accepting checkpoint requests from Checkpoint Agents and streamline requests from applications to checkpoints
Maintaining and controlling the state information pertaining to checkpoints
Coordinating read and write accesses to/from checkpoint applications across the cluster
Keeping track of CPNDs on other nodes in order to update the local data if a CPND that is managing the active checkpoint goes down
Maintaining local replicas in shared memory
Storing checkpoint control information in the shared memory so that it may be retrieved after a CPND restart
Managing accesses to sister replicas and coordinating accesses from other applications to the replicas within the scope of the CPND

CPND 会在Controller 和 payload 中都运行。 主要负责以下任务
接受Checkpoint |Agent 发送过来的checkpoint request  以及 application 发过来的streamline request
维护和控制和checkpoint 有关的信息
在cluster中控制和协调 application 对于checkpoint 的写操作和读操作
如果负责关系active checkpoint的那个 CPND down了，它负责追踪其他的 CPNDs 保持监控以保证本地数据被跟新。
管理和兄弟 replicas的访问以及协调其他application 对于本scope下replia的访问权
3.3.2.2.3 Checkpoint Agent
The Checkpoint Agent (CPA) is a linkable library available to applications that want to use checkpoint services.

Checkpoing Agent 则是一个静态链接库，用来访问 checkpoint service
3.3.3 Message Queue Service
The Message Queue Service (MQSv) implements the SAF Message service API.
3.3.3.1 Basic Functionality
Sender application(s) which use this service, send messages to queues and queue groups managed by MQSv and not to receiving application(s) directly. This means, if a process dies, the message persists in the queue and can be read by the restarted application or by another process.

用这个service的发送者将消息发送给 MQSv 的queue 以及 queue manager而不是直接发给接受者。 也就是说如果接受者死掉了，数据还是被保存在queue中，可以被重启过的application 或者其他的 application 读取。

Applications may create and destroy queues, where each queue has a globally unique name. Multiple senders may then direct messages to a queue, while a single receiver may read messages from the named queue.

应用程序可以创建和删除queue，每一个queue都会有全局唯一的名字。 多个发送者可以发给同一个queue然后由同一个接受者读取。
Applications may group several queues into a system-wide named queue group. When sending a message to a queue group, a group policy dictates which queue actually receives the message. The sender does not know how many queues are in the group or what the policy is.
应用程序有可能会将多个queue group 成一个 queue 。当发送者发送消息给这个组的时候，group策略会决定哪一个queue应该接受这个消息。 发送者不需要知道具体queue的细节信息。

3.3.3.2 Architecture
The MQSv service consists of the following three subparts:
Message Queue Director
Message Queue Node Director
Message Queue Agent
3.3.3.2.1 Message Queue Director
The Message Queue Director (MQD) runs as process on a system controller node. Its main tasks are:
Maintaining location and state data of all queues and queue groups in a system
Resolving all queue and queue group names and location information
Supporting group change tracking on behalf of registering clients
There is an active and a stand-by MQD running respectively on the two system controller nodes. MQD uses the OpenSAF Message based Checkpoint Service to keep the two synchronized and available for failover situations.

MQD是run在system controller 中的。主要的tasks是
维护queue 以及 queue group 的 位置和状态信息
解析queue以及queuegroup的名字和位置关系
提供 因为client注册造成的group 变化的跟踪能力

3.3.3.2.2 Message Queue Node Director
The Message Queue Node Director (MQND) runs as process both on payload and on controller
nodes. Its main tasks are:
Managing queue send/receive operations initiated by Message Queue Agents (MQA)
Creating, maintain and destroy queues  
Notifying MQAs when messages are delivered, received or when tracked group traits change
Destroying a queue if its creator process dies or a retention timer expires
Preserving messages until fetched or queue is destroyed

MQND 试运行在 payload 和 controller node 中主要的任务是
接受和发送由MQA过来的操作
创建，维护和删除queue
当有消息比如message 发送，接受或者group 变化时通知 Notifying MQA
当创建者不存在，或者超时以后会负责删除queue
在queue被删除之前维护queue里面的message 
3.3.3.2.3 Message Queue Agent
This is a linkable library that makes all MQSv APIs available to applications.  
一个静态的链接库
3.3.4 Event Distribution Service
The Event Distribution Service (EDSv) is compliant with the Event Service APIs defined by the SAF.
This service controls the multiplexing of event messages in a publish/subscribe environment. It exposes a rich set of APIs which allow applications to control event distribution criteria. The implementation details of the event distribution mechanism remain transparent to the application. In the OpenSAF environment, the EDSv uses the underlying Message Distribution Service (MDS) to implement the communication channels.

EDSv  实现了 Event Service 的API。这个service在发布订阅环境中控制event 消息的复用。 它提供了丰富的API来允许Application 控制消息的分发。对于Application来说，消息的分发机制仍然是透明的。在OpenSaf环境中，EDSv使用 underlying Message Distribution Service (MDS)来实现通信的channel

The EDSv functionality is closely linked with other OpenSAF services, such as System Definition, Availability Management Framework and Checkpoint Service.
EDSy功能是非常重要的一个service，很多服务都会用到它。比如系统定义，Availability Management Framework 以及Checkpoint Service.


EDSv 

3.3.4.1 Architecture
The EDSv consists of the following two parts:
Event Distribution Server (EDS)
Event Distribution Agent (EDA)
3.3.4.1.1 Event Distribution Server
The Event Distribution Server (EDS) is an OpenSAF process on the System Controller blade which handles the distribution of events based on client subscriptions and filtering mechanisms. If an event was posted and event persistence was specified, the event will be retained by the server process for the time period specified in the call. During the retention time period, the EDS may redistribute the event to new subscribers for that event. Events are distributed based on a match against the filter settings specified by the subscribed client and a priority specified in the event header.

这个service没有director，只有两个组件，一个是server一个是Agent。 Service 是一个工作在Controller blade 上的，它会根据client的订阅情况和filter机制来负责分发消息。 如何一个event需要被持久化，则在一段时间内，这个消息会被server保留着。EDA可能会将event redistribute给另一个subscirbers。 Event 的分发是基于subscriber client的filter，以及hader中定义的 priority。

There is an active and a stand-by EDS running respectively on the two system controller nodes. EDS uses the OpenSAF Message based Checkpoint Service to keep the two synchronized and available for failover situations.

Active 和 Stand-by系统会分别run在两个controller node上。 EDS 使用 Message based Checkpoint Service 来保证两个 service 同步以提供 HA 能力。

3.3.4.1.2 Event Distribution Agent
This is a library that makes the EDSv APIs available to applications. The APIs themselves are all described in the respective SAF documents.

同样是一个API

3.3.5 Global Lock Service
The Global Lock Service (GLSv) implements the SAF Lock Service API.
3.3.5.1 Basic Functionality
The GLSv provides a distributed locking service which allows applications running on multiple nodes to coordinate access to shared resources.
Locks are created and destroyed by applications as needed. Participating applications know that the locks exist and know how to use them. Access policies are outside the scope of the GLSv, which only provides the locking mechanism.
The GLSv supports exclusive and shared access modes. Exclusive access mode means that only one requester is allowed through the lock at a time. Shared access mode means that multiple requesters are allowed through a lock at a time.
The GLSv furthermore supports synchronous and asynchronous APIs to carry out locking operations. In addition, GLSv provides an internal mechanism which ensures deadlock detection and prevention.
If an application creates a lock and then exits without unlocking, orphan locks are the result. Orphan locks are managed until they are properly purged from the system.

Global Lock Service 提供了 分布式的锁系统以提供分布式的application协调共享资源的能力。 Application根据需要创建和销毁锁。  其他参与竞争的Application会相应知道这个锁的存在与否以及如何使用它。 Access policies 则不再这个服务范围内，这个服务只提供锁的机制
GLSv支持排它锁和共享锁。 Exclusive 同时只允许一个request访问，而共享锁则支持多个request同时持有
GLSy以后还支持同步和异步的API， 此外它还提供内部的机制来保证检测和防止死锁
如果一个application创建了锁，但是退出的时候却没有解锁，那么这个锁就成为了Orphan 锁， Orphan 锁会在被system 清除之前一直被维持。
3.3.5.2 Architecture
The GLSv  consists of the following subparts:
Global Locking Director
Global Locking Node Director
Global Locking Agent
There is an active and a stand-by GLD running respectively on the two system controller nodes. GLD uses the OpenSAF Message based Checkpoint Service to keep the two synchronized and available for failover situations.
架构还是这样 director ， node director 以及agent
3.3.5.2.1 Global Locking Director
The Global Locking Director performs the following tasks:
Generating unique IDs referred to by an application process
Naming one of the Global Locking Node Director (GLND) subparts as master of a particular resource
Reelecting a new master GLND for a resource if a GLND has left the system
Global Locking Director 主要负责一下功能
根据application process的信息生成唯一的ID
为其中的一个Node Director 制定 管理权限
选择新的 GLND master
3.3.5.2.2 Global Lock Node Director
The Global Lock Node Director (GLND) runs as process on all the  payload and system controller nodes. Its main tasks are:
Managing the resource open and lock operation initiated by GLAs.
For a particular resource, the GLND designated by GLD act as Master. This Master GLND is responsible for managing the lock and unlocks operations on those resources.
GLND maintains the persistence state information in a shared memory to protect against GLND crashes and restarts
而NODE级别的Director 则负责一下任务
控制GLA过来的resource或者锁的操作
作为某些资源的master，它负责关系对于这个资源的加锁和解锁操作
GLND 在共享内存中维护持久化的state information
3.3.5.2.3 Global Locking Agent
A Global Locking Library (GLA) is a linkable library which makes the respective GLSv APIs available to applications.  
一个library
3.3.6 Log Service
The Log Service (LogSv) implements the SAF Log Service API.
3.3.6.1 Architecture
The LogSv consists of the following parts:
Log Server (LGS)
Log Agent (LGA)
3.3.6.1.1 Log Server
The Log Server (LGS) is an OpenSAF process executing on the System Controller nodes which handles formatting of log records and writing them to file.
There is an active and a stand-by LGS running respectively on the two system controller nodes. LGS uses the OpenSAF Message based Checkpoint Service to keep the two synchronized and available for fail over situations.
See the LOG programmer's reference for more information.
这个只有Service，运行在 Controller NODE 上面，负责接收log record 并且写到文件中。 这个是一个active standby的系统。也是用 OpenSAF Message based Checkpoint Service来负责同步和保证高可用性

3.3.6.1.2 Log Agent
This is a library that makes the LogSv API available to applications. The API itself is described in the respective SAF document.
3.3.7 Platform Management Service
The OpenSAF Platform Management Service (PLMSv) is the core service of the OpenSAF software. It provides a logical view of the hardware and low-level software of the system. This logical view is presented in the Service Availability Forum Information Model by a set of objects that
allows for the management of hardware entities and execution environments
allows other software to keep track of status changes of the hardware and execution environments
allows the mapping of the HPI data to objects represented in the SAF Information Model
Platform Management Service is made up of the following sub-parts:
Platform Management Server (PLMS)
HPI Request Broker (HRB)
HPI Session Manager (HSM)
Platform Management Coordinator (PLMC)
Platform Management Agent (PLMA)

PMSSv 是一个非常重要的service，它提供了对于硬件以及底层软件的抽象。这个逻辑抽象被表示成为Service Availability Forum Information Model。
允许管理硬件实体和执行环境
允许其他软件追踪硬件和执行关键的变化
允许将HPI的数据maping成为SAF认识的OBJECT
这个服务的service相对来说多一些

3.3.7.1 Platform Management Server
PLMS uses the Information Model Management Service (IMM) to retrieve its configuration and reflects the runtime view of the platform. PLMS is responsible for managing Finite State Machine, Readiness tracking and Entity Group management. PLM uses the SA Forum Notification Service (NTF) to generate notifications. PLMS runs on each controller (active/ standby).
这个服务用IMM来获取相对的配置并且反应到platform的执行环境中。 PLMS 负责关系有限状态机，可读的追踪信息，实体的分组。 PLM用 Notification service 来生成 notification。 PLMS 运行在所有的 controller上。 Active 和standby
3.3.7.2 Platform Management Coordinator
PLMC provides a common interface to PLMS and coordinates admin operations and verifications on EEs irrespective of which OS is run on EEs. There will be one PLMC on each cluster node.
PLMC提供了公共的接口给 PLMS以及 admin 而不需要关心这个 具体的操作系统。 每一个 cluster node上都有一个 PLMC 
3.3.7.3 Platform Management Agent
The PLMA is a library linked with a client application, PLMA will execute in the context of the client process. Clients will access PLM service through PLM API. There can be any number of such PLMA/client processes per node. Each PLMA   talks directly to the Active PLMS on the controller through MDS.

又是一个 library，运行在client process 上。节点上可以有多个 Agent， 每一个 AGENT 直接通过 MDS 来和 PLMS 通信
3.3.7.4 HPI Request Broker
HRB facilitates all the HPI requests from PLMS. MDS is used for communication from PLMS to this module. This module implemented as a thread of PLMS receives and processes the HPI requests from PLMS. PLMS uses entity path to refer to any entity.


3.3.7.5 HPI Session Manager
HPI Session Manager interfaces directly with the HPI and is implemented as a separate thread of PLMS. HSM is designed to manage single domain only. It discovers the resources available on the domain managed by the session and blocks for receiving HPI events. Only Hot-swap and Resource events are forwarded to PLMS.
3.3.8 Software Management Framework
The Software Management Framework maintains the information about the availability, the contents, and the deployment of different software bundles in the SA Forum system. A collection of software for these software entities is delivered to the SA Forum system in the form of a software bundle. The contents of a software bundle are described in terms of the types of software entities it delivers.

软件负责管理软件 bundle。 一个bundle的意思是一个 

The SMF service is implemented as a director process (smfd) executing on the controllers using a 2N redundancy model and a node director (smfnd) process executing on all nodes without redundancy. The node director is a helper process used by the director when a command needs to be executed on a specific node
SMF 也是 一个 Director 和 node Director。  Node Director 负责处理对于某一个特殊node的操作

The smfd is the process that is the IMM-OI for the campaigns and executes the actual upgrade campaign. Only one campaign can be executed at the time. When the admin operation execute is called on a campaign object a campaign thread is started that executes the campaign


3.3.9 Cluster Membership Service
The Cluster Membership Service (CLMSv) provides applications with membership information about the nodes that have been administratively configured in the cluster configuration, these nodes are called cluster nodes or configured nodes. CLMSv is core to any clustered system. A cluster consists of the set of configured nodes, each with a unique node name.

CLMSv 给 application 提供成员关系信息，这些信息是之前已经被配置在 cluster configuration 里面了， 这些node 被叫做 cluster node 或者 configured node。 CLMSv 是cluster中的一个核心系统。 在一个 cluster 中有多个 configured node，每一个 node 有一个 为了的 node name

Cluster Membership Service is comprised of the following modules:

Cluster Membership Server (CLMS)
Cluster Membership Agent (CLMA)
Cluster Membership Node Agent (CLMNA)
3.3.9.1 Cluster Membership Server
The CLMS maintains the database of all nodes in cluster. CLMS maintains information of applications that have registered for cluster tracking and the applications are notified through callbacks about membership changes to the cluster nodes.

Server 维护了membership的数据。
3.3.9.2 Cluster Membership Agent
The CLMA is a library that any process must link with, to access CLMSv functionality. CLMA supports two sets of API’s based on B.01.01 and B.04.01 version respectively.
一个 library
3.3.9.3 CLM Node Agent
One instance of the CLMNA runs on each node. The CLMNA updates the node specific attributes to the CLMS. The CLMNA allows a node to join the cluster only if it is a configured node. It is also responsible for monitoring the presence of system controller nodes in the cluster, and initiates an election for a new active system controller when the cluster has no active or standby controller at cluster startup or when both the active and standby controller nodes have failed simultaneously.
Node agent 则是在每一个 node中都会运行的。 Agent 会将数据 update 到 CLMS 中。 Agent 只会讲配置过的node 加入到 cluster中。 同时他还负责监控当前系统的node， 同时在系统中没有active or standby 的controller 也会发起 election。
3.3.10 Information Model Management Service
The IMM service comprises of three subparts distributed among many processes
across the cluster:
• IMM Director
• IMM Node Director
• IMM Agent

Information Model management Service也是Director， Node Director 
3.3.10.1 IMM Director
The IMM Director is responsible for implementing mainly two functions:
Reliable multicast between IMMNDs.
Election of IMMND Coordinator.

The multicast service that the IMMD provides for the IMMNDs is supposed to be reliable in the sense that any message arriving to one IMMND should also arrive at all other IMMNDs and in the same order.
Director 负责两个部分，一个是是将消息发送到 IMMNDS，还有一个是负责选择 IMMND的 Coordinator

3.3.10.2 IMM Node Director
The IMMND process contains the IMM repository and is the actual provider of the IMM service at the node. All connections and sessions started at the node are handled by the IMMND at that node.

Node Director 包含了 IMM 仓库以及 IMM service服务。 所有在当前node上发起的connection 以及 serssion 会被这个node handle

At all times when the IMM service is available in the cluster, one and only one of the IMMNDs has the role of coordinator.
The IMMND that is currently the coordinator will take on the task of conducting a few cluster global operations when needed:
Initial loading;
Sync of restarted IMMNDs;
 Aborting CCBs waiting on apparently hung implementers.

在一个cluster中，只有一个 IMMND 是coordinator， 作为 coordinator 主要负责一些cluster 全局的操作比如
初始化 loading
同步以及重启 IMMND
在某些情况放弃CCB waiting

3.3.10.3 IMM Agent
The IMM Agent (IMMA) is a library that any process must link with in order to access IMMSv functionality. It supports the API defined by SAF’s IMM Service specification.
There are two interfaces OM and OI, and two libraries libSaImmOm.so and libSaImmOi.so provided. Management applications use only the OM interface. Object implementer applications use both the OI and OM interface. In particular, the OM interface is used by implementers to obtain the initial configuration.

IMM agent 是一个library。 这个library提供了连个interface，一个是OM 一个是OI. 管理application使用 OM interface 而  具体干活的application则连个interface都用。更明确的说，OM interface 主要被 具体干活的 application 用来获取 initial configuration

3.3.10.4 Persistence
The current implementation of the IMMSv in OpenSAF does not fully support persistence at the CCB granularity. Persistence is supported by IMM using Persistent Back-End (PBE). PBE has been made based on sqlite.
当前的 IMMSv 实现不支持 CCB 力度的吃持久化。 IMM 使用 PBE 来负责持久化， 而PBE 则使用了Sqlite

This allows the user to:
dump imm contents to an sqlite3 database file.
load the imm-service from an sqlite3 database file
Enable and Disable the PBE feature at runtime.

The following command line programs are available: immdump, immfind, immlist, immcfg and immadm. Each command is described in the OpenSAF IMM Service Programmer's Reference document.
IMM service has dependency on the xml2 library. This is a 3rd party open-source product. It is used by the immloader and imm-dumper to parse and generate the imm.xml file.

IMM 目前支持如下命令 immdump， immfind immlist immcfg 以及 immadm 在文档中都有描述。

3.3.11 Notification Service
NTFSv comprises two subparts distributed among many processes across the cluster:

Notification Server (NTFS)
Notification Agent (NTFA)

3.3.11.1 Notification Server
The Notification service is implemented as a server process executing on the controllers using a 2N redundancy model. Message based check pointing is used to synchronize the two server instances. The NTF Server holds a database of the subscribers in the cluster and send incoming notifications to subscribers. The NTF server also holds a cache of the latest 100 sent alarm notifications which can be read by using the reader API.

Notificaiton Server 持有 subscribers database 并且将过来的 notification 发送给 subscriber。同时它在内存中保持最少100个 alarm notification
3.3.11.2 Notification Agent
The Notification Agent (NTFA) is the library that any process must link with to access NTFSv functionality. It supports a subset of the API defined by SAF’s NTF Service specification. Notifications sent by invoking the API in the Notification Service are forwarded to the NTFS by NTFA.

一个library
3.4 OpenSAF Complementary Services
This section describes in more detail the OpenSAF complementary services which were introduced to complement the OpenSAF services that implement SAF APIs.
3.4.1 Message-Based Checkpointing Service
The Message-Based Checkpointing Service (MBCSv) was introduced to complement the SAF-compliant Checkpointing Service. MBCSv defines a simple synchronization protocol which keeps an active and one or more of its stand-by entities in synchronization. Unlike the SAF-compliant Checkpointing Service, the MBCSv does not maintain replicas.

MBCSv 是为了作为 SAF Checkpoint的补充。它主要提供了一个简单的同步协议，以保持active 和 stand by 的实体同步。 但是和 Checkpoint 服务部一样的是，这个service并不保存副本
3.4.1.1 Basic Functionality
The main tasks of the MBCSv are:
Dynamic discovery of peer entities
Providing an interface to the active entity for check-pointing the state updates to the standby peers
Whenever the clients role changes to stand-by from any other role or whenever a new active client is detected, the client synchronizes its state with that of the active client (cold synchronization)
Periodic synchronization of the client’s state with that of the currently active client to obtain an abbreviated summary account (warm synchronization)
Driving client behavior depending on the HA role assigned by the client application
主要任务有
动态发现 peer entites
给active 提供接口将消息发同步给 standby peer
无论何时，当一个client 从 无论什么状态变成stand-by， 或者一个生成了一个新的active client。 当前的这个client都会和active做一次同步。这个叫做cold synchronization
 定期的和当前active 的 client 做同步 (warm synchronization)
3.4.1.2 Architecture
The only component of the MBCSv is the Message-Based Checkpointing Agent (MBCA). It provides stateful, message-based checkpoint replication services for its clients. 








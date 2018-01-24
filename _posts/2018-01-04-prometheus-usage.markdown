---
layout: post
title:  "5分钟理解promethus"
date: 2018-01-04 15:10:00 +0000
categories: container, Prometheus
---


### prometheus
prometheus 的使用本身来说其实没什么难度, 重点在于理解他的工作方式,以及数据结构.  理解工作方式是因为被监测的application 需要提供必须的interface, 这样prometheus 才能够 pull 到所需的数据.  理解数据结构是因为数据结构直接和如何进行search 相关连. 不然我们肯定会淹没在大量的数据信息之中




### Datamodel
从 [data model](https://prometheus.io/docs/concepts/data_model/) 开始理解. 按照官方文档的顺序来理解.  

首先, 所有的数据都是按照时间戳来记录的, 也就是说所有的数据应该都是有时间信息.如果是一些特殊的查询结果, prometheus 也会生成一些临时的时间戳信息.
但是光有时间戳无法区分每一条数据对应的含义, 所以另外一个重要的信息就是Metric name.  metric 是用来标志比较通用的feature 名字, 比如 (http_request_total, 所有收到的 http的消息).  prometheus 规定了一定的格式
如果说 Metric 是 namespace的话, 那么label 就是真正的查询条件了, 因为一个数据可以给与很多的label, 将 label 组合在一起就可以给出成变化无穷的dimensional data model. (例如 所有给 /api/tracks 发送的 post https request)
[bast practices](https://prometheus.io/docs/practices/naming/)  里面描述了如何起一个好的名字.

最后,如果觉得自己写 metric 和 lable 的组合太麻烦, 可以预订一些基础的定义, 也就是 Notation, 例如

api_http_requests_total{method="POST", handler="/messages"}



#### Metric Types 
Proetheus 的客户端提供了四种 core Metric Type, 这四种librairs 只是在客户端以及协议里面有所区分 (PS:这里没有太弄明白) . 但是在server 这一段并没有做什么特殊的处理.  不过这可能将来会有变化.  
四种 core type 分别是 

* Counter:  Conter 简单的说就是一个只能正向增加的数值. 一般用作 request 处理数, 完成的task 数量, errors 的数量. 
* Gauge: 则是是和 conter 想对应可以用来表示一个可以增大以及减少的数值. 例如当前的线程数啊一类的
* Histogram: 这类的数据是 将监控的数据在一定的范围内进行计数, 或者对于某一个被观察的数据计算总和. 举个例子就是关于时间的柱状图.
* Summary: Summary 和 histogram 有点类似,但是不会以住装图显示, 而是显示一个当前的值. 例如当前的cpu占用率,磁盘剩余空间等. 这种便于以静态图的方式显示


#### Instances and jobs
最后两个重要的概念就是 instances 以及 jobs了. 本来在以前,只需要一个instance的概念可能就能解决大部分的问题了, 但是随着现代计算机服务的发展,基本上所有的服务都有scalability 和 failover的概念了,所以需要 增加一个 job的概念.  Instance 代表了某一个具体的数据, 而job 则是代表了一个cluster. 例如

- job: api-server
-- instance 1: 1.2.3.4:5670
-- instance 2: 1.2.3.4:5671
 -- instance 3: 5.6.7.8:5670
 -- instance 4: 5.6.7.8:5671
  
  当prometheus 抓取数据的时候, 它会自动根据job 以及 instance 的信息对数据贴上标签. 默认的情况是 
-- job: The configured job name that the target belongs to.
-- instance: The <host>:<port> part of the target's URL that was scraped.







 






---
layout: post
title:  "总结一下prometheus的相关配置"
date: 2018-01-05 15:10:00 +0000
categories: container, Prometheus
---


### prometheus
之前一片文章说道prometheus 的使用本身来说其实没什么难度, 重点在于理解他的工作方式,以及数据结构.  那么里理解数据结构以及概念以后, 下一步就是熟悉它的配置, 所谓的配置也就是它所提供的feature, 这个关系到我们怎么去用这个工具解决我们的实际问题, 所以非常重要.


#### 配置种类
为了方便理解诶和设计自己的配置文件, 可以先理解一下这个配置文件的思路. 所谓的思路就是这里的配置如何分类的. Prometheus 将配置分为了一下几类

* **Global configuration**: 这些配置在所有的环境下有作用, 可以被覆盖, 如果没有覆盖,就可以为其他的 configuration 提供默认值

* **scrape_config**: 翻译成中为是 "抓取配置", 也就是用来配置如何抓取 target 信息相关的参数. 没一个 scrape_config  描述了一个 "job" 由于给很重要的feature 就是它提供了动态 discovery 的机制,  也就是所Prometheus 可以根据 service discovery 动态的配置.

* ** tls_config**: tls config 是用来配置 TLS 安全链接的,  估计现在这一个是必需品了

* **各种 service discovery 的配置**: 上面提到了, service disovery 可以为Prometheus 动态的提供 target 信息. 所以这里支持了各种云的 service discvoery 的支持.

* **relabel_config** : 这个根据官方文档来说, 是一个非常powerful的工具, 他可以在抓取数据之前以及之后,修改默认的lable信息.
-- metric_relabel_configs : 这个和relabel有点类似,但是是在抓取数据之后做处理,这个参数的有一个用处就是用来选择丢弃一些需要大量处理的数据
-- alert_relabel_configs: 这个是在发送 alarm 之前对数据进行 relabel

* **alertmanager_config**: 这个是用来配置 alarm manager的, 注意的是这个 alarm manager 支持service discvoery, 也就是说可以动态增加删除 alarm 的target


#### Kubernetes 配置
因为 kubernetes 应该是现在最火的工具了, 所以主要看一下关于 kubernetes的 SD 配置. 
目前默认的kubernetes支持 4种role  endpoint, service, pod 以及 node, 这些是不需要写任何代码就可以提供的功能.








 






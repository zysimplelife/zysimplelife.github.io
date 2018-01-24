---
layout: post
title:  "Container的一些使用经验"
date: 2018-01-20 15:10:00 +0000
categories: Cloud
---


一不小心就调到 kubernetes的坑里面去了, 本来以为挺有意思的事情做到后来发见好大半年都没有真正写过代码了,折腾来折腾去都是各种docker/kubenetes的配置. 感觉再这样下去就要失业了. 
不过半年多的时间也不想就这样浪费了, 总希望可以学点新的东西. 想来想去也只能希望可以不可以把一些有用的经验写一写吧, 还有一些坑.


### 常用命令 ###
有一些常用命令不知道放在什么地方, 放在这里也算占一些篇幅吧

#### 快速清空 docker 所有环境 ####
这个是我认为最有用的命令了. docker 动不动就弄一大堆的退出的container, 看着挺讨厌的. 还有很多不用的镜像 不过这个主要是在开发环境用, 因为不要一不小心删了一些有用的东西
```bash
#删除containers
docker stop $(docker ps -a -q)
#删除images
docker rmi -f $(docker images -q)
```

不过后来有了更强大的命令, 注意如果加上 -a 则会清理所有dangling的镜像, 个人所谓dangling的概念就是那些没有任何名字的image. 一般是你创建image新的image将原来的image的tag顶替掉了. 原来的就只能变成 danging的概念了
```bash
#删除containers
docker container prune 
#删除images
docker system prune -a
```




###kubenetes 学习到的经验 ###
因为现在日常使用kubernetes, 所以我们还是会碰到一些问题, 这里希望可以记录一些问题.不要浪费了那些花费在上面的时间.

####关于readiness probe 以及 graceful shutdown ####
首先先说一句, readiness 是非常重要的部分, 一定要保证你所提供的镜像或者产品支持kubernetes的 readiness check.  为什么说 ? 因为 kubenetes 默认的 service 将 request 转发到后台的条件就是这个 pod 是否已经 ready. 当然我们假设 liveness prob 没有问题, 不然kubernetes会不停的重启这个pod.  
所以如果一个典型的 系统 , 由一个service + 多个 pod 作为后端组成, 那么如果要保证在重启,升级,扩展等操作中保证 traffic 不丢失, 那就需要做到 readiness check.  关于这一点, 很多的文章中都是解释, 所以搜一艘就很容易找到问题. 
这里我想记录的是我们还是碰到一个问题, 那就是在 delete pod 的时候, readiness check 和 delete 不是同步的, 他们是两个异步的过程. 具体的来说,就是当 scale in的时候, kubenetes 会发 Singal 给 POD, 然后等待默认30秒的 Terminate 时间. 问题是如果这段时间内, Readiness Prob 不能及时将pod 的状态设置为 unready, 系统还是会收到 traffic, 对于客户端来说,这时的行为将是不可控制的. 关于这个问题, kubernets 的 github 上有一个 [ticket](https://github.com/kubernetes/kubernetes/issues/43576) ticket 作了一些讨论.   最后我们的方案也是基于ticket中的结论. 具体是在prestop 这个lifecycle 中增加了一个 readiness 的条件, 当系统收到 singal的时候, 将这个 条件设置为 false. 然后再sleep 一定时间让 readness 工作. 











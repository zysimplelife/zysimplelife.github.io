---
layout: post
title:  "Container的一些使用经验"
date: 2019-10-08 15:10:00 +0000
categories: Cloud
---


Systemd AuditD syslog 以及和 container 的关系

### Systemd 和 syslog 的关系

自从大部分的主流linux都开始使用 systemd 来作为 init system 以后, linux 系统中管理service  查看log 的方法都有所变化, 个人理解大家选择systemd的主要原因是可以大大减少启动的时间. 因为 systemd 启动service是并行的.

> The reason for this wide-scale adoption is the versatility of systemd. It manages not only daemons and processes in a Linux system, but also various resources like devices, sockets, and mount points. When the system boots, it does not load services sequentially like System V, which saves significant time at startup. Services are loaded in parallel, and a service waits until other required resources for it are also activated.

systemd 有一套自己管理log 的方式, 所以其实systemd可一替代 syslog 作为唯一的log方式, 或者作为syslog的一种补充.  目前来看后者居多, 大部分的系统是同时拥有 syslog 和 systemd 的.  所谓问题就来了, 如何让他们两个和谐的工作在一起.  目前我的理解是一下几点

* 使用 systemd 来管理 syslog(rsyslog) service
* syslog(rsyslog) 监听 journal 的socket

#### Systemd 是如何管理 log 的

这里直接摘抄引用文章中的描述. systemd 将左右的 log streams 都集中到 journal daemon 中. 比如 /dev/log, stdout/stderr 或者 kernel.  journal daemon 根据配置将他们存在内存或者磁盘上.

>A few notes in advance: systemd centralizes all log streams in the Journal daemon. Messages coming in via /dev/log, via the native protocol, via STDOUT/STDERR of all services and via the kernel are received in the journal daemon. The journal daemon then stores them to disk or in RAM (depending on the configuration of the Storage= option in journald.conf), and optionally forwards them to the console, the kernel log buffer, or to a classic BSD syslog daemon -- and that's where you come in.

#### syslog.service 如何连接到 journal 中的

syslog.service 的配置一般是放在 /etc/systemd/system/syslog.service 里. 仔细观察这个配置文件会发现他其实依赖syslog.socket 这个配置.  这里就能发现 systemd 不仅仅负责管理 service, 还负责管理 socket 等资源.  这里 syslog 和 systemd 的关系就比较清楚其.  syslog 不再负责监听 /dev/log 而换做是 journal 负责. 但是 systemd 提供了一个 新的 socket 来分发log.

>Note that it is now the journal that listens on /dev/log, no longer the BSD syslog daemon directly. If your logging daemon wants to get access to all logging data then it should listen on /run/systemd/journal/syslog instead via the syslog.socket unit file that is shipped along with systemd. On a systemd system it is no longer OK to listen on /dev/log directly, and your daemon may not bind to the /run/systemd/journal/syslog socket on its own. If you do that then you will lose logging from STDOUT/STDERR of services (as well as other stuff).



```
[Unit]
Description=System Logging Service
Requires=syslog.socket
Documentation=man:rsyslogd(8)
Documentation=http://www.rsyslog.com/doc/

[Service]
Type=notify
ExecStart=/usr/sbin/rsyslogd -n
StandardOutput=null
Restart=on-failure

[Install]
WantedBy=multi-user.target
Alias=syslog.service
```

```
[Unit]
Description=Syslog Socket
Documentation=man:systemd.special(7)
Documentation=http://www.freedesktop.org/wiki/Software/systemd/syslog
DefaultDependencies=no
Before=sockets.target shutdown.target

# Don't allow logging until the very end
Conflicts=shutdown.target

[Socket]
ListenDatagram=/run/systemd/journal/syslog
SocketMode=0666
PassCredentials=yes
PassSecurity=yes
ReceiveBuffer=8M

# The default syslog implementation should make syslog.service a
# symlink to itself, so that this socket activates the right actual
# syslog service.
#
# Examples:
#
# /etc/systemd/system/syslog.service -> /lib/systemd/system/rsyslog.service
# /etc/systemd/system/syslog.service -> /lib/systemd/system/syslog-ng.service
#
# Best way to achieve that is by adding this to your unit file
# (i.e. to rsyslog.service or syslog-ng.service):
#
# [Install]
# Alias=syslog.service
#
# See http://www.freedesktop.org/wiki/Software/systemd/syslog for details.

```

#### syslog 和 container 的关系
理解了传统的log, 再来看一下 container 是如何处理 syslog 的. 因为在container 的概念里默认是没有systemd 以及 syslog 的. 因为container不会启动一个完整的 operration system. 但是 host机器中是有 systemd 的. 他们一般是如何联系的?  这里涉及到 docker 的日志系统  docker 的日志系统其实支持多种 driver, 默认情况下是写在一个文件系统之中.  但是可以在启动的时候选择不同的driver. 也就是说docker 是支持将log 写入除console 之外的地方.

>By default, docker logs or docker service logs shows the command’s output just as it would appear if you ran the command interactively in a terminal. UNIX and Linux commands typically open three I/O streams when they run, called STDIN, STDOUT, and STDERR. STDIN is the command’s input stream, which may include input from the keyboard or input from another command. STDOUT is usually a command’s normal output, and STDERR is typically used to output error messages. By default, docker logs shows the command’s STDOUT and STDERR. To read more about I/O and Linux, see the Linux Documentation Project article on I/O redirection.
>
支持的 log driver 有 挺多 可以在link 中查到 [Configure logging drivers \| Docker Documentation](https://docs.docker.com/config/containers/logging/configure/) 默认的 driver 可以用命令查看

```
docker info --format '{{.LoggingDriver}}'
```






#### 如何让现有的log 都写道 console 里面去?
因为 docker 日志系统默认只会写 console log, 所以作为 image 需要做一定的修改来满足docker. 如果是自己写的代码还是比较容易控制的, 但是如果是第三方的程序有时候就比较麻烦一点.  现有的解决方法就是做sympol lync. 比如  nginx 的image 就床了一个 symbolic link 到 stdout

>The official nginx image creates a symbolic link from /var/log/nginx/access.log to /dev/stdout, and creates another symbolic link from /var/log/nginx/error.log to /dev/stderr, overwriting the log files and causing logs to be sent to the relevant special device instead. See the Dockerfile.


#### Kubernetes 如何管理 log
默认情况下 k8s 一般是将 log 的事情交给 container engine 来处理的, 比如说 docker 的情况下就是会将log 配置成文件. 但是同时会有一个组件来负责对 rotate log, 以避免文件过大. [logrotate(8) - Linux man page](https://linux.die.net/man/8/logrotate)

>Everything a containerized application writes to stdout and stderr is handled and redirected somewhere by a container engine. For example, the Docker container engine redirects those two streams to a logging driver, which is configured in Kubernetes to write to a file in json format.

但是系统组建是会将log 写到系统日志里. 对于有 systemd 的系统, 会写道journal 里面. 如果没有的话就会写道 /var/log 下面了.  注意对于以container 模式运行的系统组建, 是绕过的 默认的 logging 系统, 而使用 klog 来写入 /var/log 里

> On machines with systemd, the kubelet and container runtime write to journald. If systemd is not present, they write to .log files in the /var/log directory. System components inside containers always write to the /var/log directory, bypassing the default logging mechanism. They use the klog logging library. You can find the conventions for logging severity for those components in the development docs on logging.



![logging-node-level.png](https://d33wubrfki0l68.cloudfront.net/59b1aae2adcfe4f06270b99a2789012ed64bec1f/4d0ad/images/docs/user-guide/logging/logging-node-level.png)



### Reference
1. [Writing syslog Daemons Which Cooperate Nicely With systemd](https://www.freedesktop.org/wiki/Software/systemd/syslog/)
1. [rsyslog, journal or both](https://albertomolina.wordpress.com/2017/12/30/rsyslog-journal-or-both/)
1. [Linux Logging with Systemd - The Ultimate Guide To Logging](https://www.loggly.com/ultimate-guide/linux-logging-with-systemd/)
2. [Journald system and Docker logs \| Log Analysis | Log Monitoring by Loggly](https://www.loggly.com/docs/journald-system-and-docker-logs/)
3. [Logging Architecture - Kubernetes](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
---
layout: post
title:  "How to use persistently install docker volume plugin in boot2docker"
date:   2016-05-17 00:16:00 +0000
categories: JAVA
---

### Introduction ###
From version 1.8, docker provide new volume persistence feature via volume plugin which makes it available to integration with 3rd part persistence solution including both local and cloud storage.  User can install and use 3rd volume plugin like flocker to help build their persistent solution.  Details can be found from website of those 3pps 

- [rexray](https://github.com/emccode/rexray)
- [Flocker](https://clusterhq.com/flocker/introduction/) 

boot2docker is a tiny Linux release to provide a complete client/server environment to docker user. I am using it in my design environment which help to solve a lot of environment problem. Because the the tiny release will forget all the system update after restart,  I can't persistence any new software including volume plugin.  After reading several web page from Google,  I can't find a existing solution for my use case. I think it because,in most case, designer in local environment do not need persist the result into external storage.

I need this feature because I want to have a more flex way to save the data, so I am going to trying a method to solve my problem.  

### bootsync.sh, bootlocal.sh and shutdown.sh ###

From the source code of boot2docker, there are three hook script can be used for customization. there are bootsync.sh, bootlocal.sh and shutdown.sh Those files actually do something very useful. By providing an access point to the boot up and
shutdown process they greatly reduce the temptation to modify the system boot and shutdown
scripts.

I want to add the installation script into bootsync.sh file to make a new install of volume plugin. 

### local-persist plugin ###

[Local persist](https://github.com/CWSpear/local-persist) is an good plugin which is useful for small project. 

### steps ###
- Download binary
<pre>
mkdir -p /var/lib/boot2docker/plugins ; cd /var/lib/boot2docker/plugins
sudo wget https://github.com/CWSpear/local-persist/releases/download/v1.1.0/local-persist-linux-amd64
chmod +x local-persist-linux-amd64
</pre>

- start this service in bootsync.sh
<pre>
echo "sudo /var/lib/boot2docker/plugins/local-persist-linux-amd64 > /var/log/persist.log & " >> /var/lib/boot2docker/bootsync.sh
</pre>

- restart boot2docker or manually start the service 
<pre>
sudo /var/lib/boot2docker/plugins/local-persist-linux-amd64 > /var/log/persist.log & "
</pre> 

- create a volume to test installation 
<pre>
docker volume create -d local-persist -o mountpoint=/data/images --name=images
docker volume ls
</pre> 

## summary ##

With method above, we can have boot2docker start volume plugin every time when it restart.  It is useful for some user like me.  
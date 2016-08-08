---
layout: post
title:  "How to create route to docker "
date: 2016-08-8 15:10:00 +0000
categories: docker
---

OSX:

    # show route table
    netstat -nr
    # set vm host as the gateway of the docker containers' prviate network 
    sudo route -n add 172.17.0.0/16 192.168.56.101
    or 
    sudo route add -net 172.17.0.0 -netmask 255.255.0.0 192.168.56.101

Linux:
    # show route table
    route
    # set vm host as the gateway of the docker containers' prviate network 
    route add -net 172.17.0.0 netmask 255.255.0.0 gw 192.168.56.101

Windows:
    # show route table
    route
    # set vm host as the gateway of the docker containers' prviate network 
    route add 172.17.0.0 mask 255.255.0.0 192.168.56.101
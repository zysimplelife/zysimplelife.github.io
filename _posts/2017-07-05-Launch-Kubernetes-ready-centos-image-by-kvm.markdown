---
layout: post
title:  "Launch Kubernetes ready centos image on kvm"
date: 2017-07-05 15:10:00 +0000
categories: Cloud
---

### Introduction ####
After using KubeAdm to init kubernetes cluster, installation of kubernentes become a easy job. But to prepare a kubernetes ready virtual machine in cloud or non-cloud environment become another timecost task. I was trying to make it more easier, so here comes my way of so called "automation"
Because I was working to a kvm envrionment rather than an openstack cluster.  I have to create clould init configuration following [non-clould example](http://cloudinit.readthedocs.io/en/latest/topics/examples.html) 


### Requirements ###
I want to have a kubernetes ready machine means almos everything has been prepare and I need only init this machine to join a existed kubernetes cluster. My expectation or requirements including 
- one-click : I need only run command or click a button then everything will be ready
- static ip : because I want to expose my service to others, I need a static IP host.
- DNS ready : In our lab, DNS is very important or some internatl website will be accessable
- proxy : I want to the machine ready with pre-configured proxy 
- root/password : this is an lab environmetn, I want to root/password be ready
- kubeadm/docker is ready : of cause, there are mandatory
- system configuration : 


### Image ###
Almost all the those OS provider has put the cloud native image for downloading which release together with cloud-init. But people may be annony when they found they can't log on after lauching it because it only support logon on via ssh key. That why I have to create a no-cloud cloud-init configuration to lauch the image. I chose centos 7 as my host OS which can be access from [website](https://cloud.centos.org/centos/7/images/)

After downloading it, I found the default disk size is only 10GB which seem too small to us.  I have to chooses: attach another disk or enlarge it.  Both of them are easy and I was trying to enlarge it. 


### cloud-init configuration ###
cloud init confgiuration for no-cloud need two file: user-data and meta-data.  here is my example 

user-data
'''yaml
# Hostname management
preserve_hostname: False
hostname: $1
fqdn: $1.example.local

# Remove cloud-init when finished with it
runcmd:
  - echo "proxy=http://10.170.67.6:3128" >>  /etc/yum.conf
#workaround to add DNS confiugration since "resolv_conf" is not work
  - echo "DNS1=10.175.250.86" >> /etc/sysconfig/network-scripts/ifcfg-eth0
  - sed -i -e '/^PermitRootLogin/s/^.*$/PermitRootLogin no/' /etc/ssh/sshd_config
  - sed -i -e '/^PasswordAuthentication/s/^.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
  - [ yum, -y, remove, cloud-init ]

# update dns configration
manage_resolv_conf: true
resolv_conf:
  nameservers: ['10.175.250.86','8.8.8.8']

# Configure where output will go
output:
  all: ">> /var/log/cloud-init.log"

# configure interaction with ssh server
ssh_svcname: ssh
ssh_deletekeys: True
ssh_genkeytypes: ['rsa', 'ecdsa']

# Install my public ssh key to the first user-defined user configured
# in cloud.cfg in the template (which is centos for CentOS cloud images)
ssh_authorized_keys:
  - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDDC0nHpG/g7gmN0yOSF3IhJsTSc2Cp/24gWPQ3k8o0S6jql18SYpyvXp49hTErfQUdEmmQHrbxFZgIu+vQxmwnnurMVkQEi6tIRpx6HkBiM2h6bs3G0w7sYfXVEHwLW/rle1JBU6jPLRmrbem0uGu+B9XVItwOUyDK5tQSrxy6UZUBRcP/emigrGxCM0bIJokpat6EQqN7Wva1xSbZQBhyl80+6USxjpbxtnTsWN0ype4wt3BUHZ1qWr4KnXN9HD6kEeSYGQ8iOBix0fVsW/Lwu+BmVMySm5Q/AUttgRaxAD8cc4RrJvQ+vkw01sC9iIaABZQPOHLmid4NAPOCQNRdkEDIPC7AEVRZeS/qUTvtyNO1qPk7Lxxr5HtaXYPMte/IRcG7Vp0ryzcSoUGMZZcEco8TjdN1zYCAmreLS8YuJ/AcfwuzecnqOy0NzzCWu+fLzOpfdT3LIiiMfLij1OmHedJVyqPMiEcF2uFCQpLqMM+E4mQQd1iyiqiL8ruJBMtjBbiy7WXvyubHgq9VO/QrZWLrvJ/sKVSPdw6NAVeJc2JTM6U3GxS18MR3dVr/BCOrViKjU3yU+j0wRWVeZyAgfELuj2ocF9CyFgcfwiCZ4+ISL2Tl1VQ6zuLTfpPU9kW6BJLqD+MKMjxmkDQzfva/bHWwpOtUaAfG7+oFcjEuuw== charlie.zha@ericsson.com



#set root password
chpasswd:
  list: |
    root:root000
  expire: False

# add kubernets repository
yum_repos:
    # The name of the repository
    Kubernetes:
        # Any repository configuration options
        # See: man yum.conf
        #
        # This one is required!
        baseurl: https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
        enabled: true
        failovermethod: priority
        gpgcheck: true
        gpgkey: https://packages.cloud.google.com/yum/doc/yum-key.gpg
        name: Kubernetes
		
		
# packages may be supplied as a single package name or as a list
# with the format [<package>, <version>] wherein the specifc
# package version will be installed.
packages:
 - [docker, kubelet, kubeadm, kubernetes-cni]


'''



### Reference ###

- [How checking out Kube DNS](https://rsmitty.github.io/Manually-Checking-Out-KubeDNS/) 
- [What is flannel](http://datastart.cn/tech/2017/01/18/k8s-flannel.html)
- [Install Kubernetes](http://blog.frognew.com/2017/04/kubeadm-install-kubernetes-1.6.html)
- [Kubernetes installation gist](https://gist.github.com/patrickhuber/e600629a69fec64cfb45c63a23df4b3c)





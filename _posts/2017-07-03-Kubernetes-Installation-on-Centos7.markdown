---
layout: post
title:  "Install Kubernetes on CentOS 7"
date: 2017-07-03 15:10:00 +0000
categories: Cloud
---

### Introduction ####
Those days, I have been trying to install Kubernetes in Lab environment.  Even though it has been tried more than thousand times, it still costed me a lot of time to resolve different problem. Mostly because I am not familiar with those concept of Kubernetes. So, I write this post to summary those problem I had occurs.

I used Kubeadm to install kubernetes which is not recommend to product concept because it put all kubernetes concept in container environment. In order to copy configuration from container to host system, I have to disable selinux, making the system be unsecured because every container with privilege configuration can access to host Env. But, anyway, I would not use it in product system, so it was OK to me so far. 

### Proxy ###
Kubeadm need the access to google so that it can do some pre-check and down all those service images. From the official document, kubeadm will read system environment of http_proxy, https_proxy and no_proxy. I your system is behind proxy like me, it is mandatory to set those configuration before any operation. Notice that no_proxy suppose to contains all your node IP or the API-Service will not able to get pod information.
 
```bash
export PROXY_PORT=3128
export PROXY_IP=Proxy_Server
export http_proxy=http://$PROXY_IP:$PROXY_PORT
export HTTP_PROXY=$http_proxy
export https_proxy=$http_proxy
export HTTPS_PROXY=$http_proxy
export no_proxy="/var/run/docker.sock,localhost,127.0.0.1,[NODE1],[NODE2]"
```
   
Of cause, docker and yum service needs proxy too or you will never get image or package ready. 

```shell

echo "proxy=http://Proxy_Server:3128" >> /etc/yum.conf

mkdir -p /etc/systemd/system/docker.service.d/
cat <<EOF > /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://Proxy_Server:3128/"
Environment="HTTPS_PROXY=http://Proxy_Server:3128/"
Environment="NO_PROXY=127.0.0.1,/var/run/docker.sock,.ericsson.se"
EOF

```


### Install kubeAdm Package ###
Kubeamd is an tools helping build kubernetes cluster easely, it is release together with kubernetes 1.4. Following command will install all needed package and it step is mandotory for both master and compute node.

```bash


cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

yum install -y docker kubelet kubeadm kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet

```


### Load overlay disk for Docker ###

Docker can work in several filesystem but kubeadm will throw error if overlay module are note ready. 

```bash
############load module #########
sudo tee /etc/modules-load.d/overlay.conf <<-'EOF'
overlay
EOF

echo 'STORAGE_DRIVER="overlay"' >> /etc/sysconfig/docker-storage-setup

reboot

```

### set security configuration ###

As said previous, Kubeadm requires you disable selinux to access filesystem from container to host. what more, [FORWARD is needed by DNS server](https://github.com/kubernetes/kubernetes/issues/40182). 

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
EOF

sysctl -p /etc/sysctl.d/k8s.conf

sudo iptables -P FORWARD ACCEPT
setenforce 0

```

### Init master node ###

Following command will help me to start kubernetes master node. It implicit that we are going to set pod overlay network to 10.224.0.0 which is mandotory if use flannel plugin.

kubeadm init --kubernetes-version=v1.6.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=[master node IP]


It may takes you several minutes and if everything goes will, it will print like this. 

```
kubeadm init --kubernetes-version=v1.6.1 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.61.41
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[init] Using Kubernetes version: v1.6.1
[init] Using Authorization mode: RBAC
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[certificates] Generated CA certificate and key.
[certificates] Generated API server certificate and key.
[certificates] API Server serving cert is signed for DNS names [node0 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.61.41]
[certificates] Generated API server kubelet client certificate and key.
[certificates] Generated service account token signing key and public key.
[certificates] Generated front-proxy CA certificate and key.
[certificates] Generated front-proxy client certificate and key.
[certificates] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/controller-manager.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/scheduler.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 14.583864 seconds
[apiclient] Waiting for at least one node to register
[apiclient] First node has registered after 6.008990 seconds
[token] Using token: e7986d.e440de5882342711
[apiconfig] Created RBAC rules
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  sudo cp /etc/kubernetes/admin.conf $HOME/
  sudo chown $(id -u):$(id -g) $HOME/admin.conf
  export KUBECONFIG=$HOME/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token e7986d.e440de5882342711 192.168.61.41:6443

```


### Install flannel plugin ###
Before network installed, dns service will not start. Kubernetes support several network framwork and I choose flannel this time only because I heared it before :-).


```bash 
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl apply -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### Post Check installation ###

- check all docker container is running by "docker ps"
- check all kubernetes is ready by "kubectl get po --all-namespaces"
- check is dns work properly 

```bash
kubectl run curl --image=radial/busyboxplus:curl -i --tty


[ root@curl-2421989462-vldmp:/ ]$ nslookup kubernetes.default
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local


```

### Init Compute node ###
```bash
#run this in master node
kubeadm  token list
```

```bash
#run this in compute node
kubeadm join --token e7986d.e440de5882342711 192.168.61.41:6443
```



### Reference ###

- [How checking out Kube DNS](https://rsmitty.github.io/Manually-Checking-Out-KubeDNS/) 
- [What is flannel](http://datastart.cn/tech/2017/01/18/k8s-flannel.html)
- [Install Kubernetes](http://blog.frognew.com/2017/04/kubeadm-install-kubernetes-1.6.html)
- [Kubernetes installation gist](https://gist.github.com/patrickhuber/e600629a69fec64cfb45c63a23df4b3c)





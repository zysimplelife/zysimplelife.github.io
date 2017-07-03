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
export PROXY_IP=10.170.67.6
export http_proxy=http://$PROXY_IP:$PROXY_PORT
export HTTP_PROXY=$http_proxy
export https_proxy=$http_proxy
export HTTPS_PROXY=$http_proxy
export no_proxy="/var/run/docker.sock,localhost,127.0.0.1,10.170.13.33,192.168.122.229,.ericsson.se"
```
   


```java
Factory<Command> 

public interface Command {
    void setInputStream(InputStream in);
    void setOutputStream(OutputStream out);
    void setErrorStream(OutputStream err);
    void setExitCallback(ExitCallback callback);
    void start(Environment env) throws IOException;
    void destroy();
}

public interface PublickeyAuthenticator {

    /**
     * Check the validity of a public key.
     *
     * @param username the username
     * @param key the key
     * @param session the server session
     * @return a boolean indicating if authentication succeeded or not
     */
    boolean authenticate(String username, PublicKey key, ServerSession session);

}


```


### Reference ###

- [Mina框架IoHandler与IoProcessor详解](http://shiyanjun.cn/archives/310.html) 
- [Channels](http://net-ssh.github.io/ssh/v1/chapter-3.html)
- [How to implement an ssh deamon](http://javajdk.net/tutorial/apache-mina-sshd-sshserver-example/)
- [RFC4254](https://tools.ietf.org/html/rfc4254) 




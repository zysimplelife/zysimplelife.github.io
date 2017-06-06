---
layout: post
title:  "MINA-SSHD Understanding"
date: 2017-06-5 15:10:00 +0000
categories: Source Code
---

### MINA-SSHD 

We were working for a new project to provide an ssh server demon receiving custom request. There are a good example [website](http://javajdk.net/tutorial/apache-mina-sshd-sshserver-example/) for us to start. But in order to understand better, I have to go thought the source code of MINA-SSHD and understand how it works. Following part is based on sshd 0.14 which is a very old one but is enough for us. 


### Reference 

- [Apache Commons File Upload: ](https://commons.apache.org/proper/commons-fileupload/streaming.html)



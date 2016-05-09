---
layout: post
title:  "Create your work environment with docker "
date:   2016-05-09 15:16:00 +0000
categories: Docker
---


create the docker machine

<pre>
docker-machine create -d virtualbox dev
</pre>


create a docker volume according to https://github.com/CWSpear/local-persist
<pre>
docker run -d \
    -v /run/docker/plugins/:/run/docker/plugins/ \
    -v /path/to/where/you/want/data/volume/:/path/to/where/you/want/data/volume\ cwspear/docker-local-persist-volume-plugin

docker volume create -d local-persist -o mountpoint=[your workspace path] --name=workspace
</pre>

work with that, for example 
<pre>
docker run -it -v workspace:/workspace -P jekyll/jekyll sh
</pre>
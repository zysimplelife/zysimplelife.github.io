---
layout: post
title:  "Use Docker to build local envrionment of Jekyll "
date:   2016-05-09 15:16:00 +0000
categories: Docker
---


I am trying to use git hub as the blog server because it become very popular those year. 
Github page uses Jekyll rather than orther dynaimc blog page to provide blog platfrom, which makes more hacker way to write your blog.

Jekyll https://jekyllrb.com/ is a fancy tool to record your blog in a hacker way.  Because of failed to launch the server in my local windows envrionment, I am trying to use docker to build development envrionmetn, which recored in following sections. 

create the docker machine

<pre>
docker-machine create -d virtualbox dev
</pre>

Because Jekyll have provided an docker image, ther is no need to recreate the wheel. 

<pre>
docker run -it -v /path/of/your/workspace:/workspace -p 4000:4000 jekyll/jekyll sh
</pre>

Build the blog project

<pre>
jekyll new /workspace/myblog --force
</pre>


start the serve
<pre>
cd /workspace/myblog
jekyll serve
</pre>


visit your server 
<pre>
http://[your boot2docer machine ip]:4000
</pre>

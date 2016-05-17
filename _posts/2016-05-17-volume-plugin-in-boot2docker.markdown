---
layout: post
title:  "How to use persistently install docker volume plugin in boot2docker"
date:   2016-05-17 00:16:00 +0000
categories: JAVA
---

### Introduction ###
To do trouble-shooting on performance or memory issue, we are possible to use visualVM to get information on  suspect objects. 

The problem is, sometimes, we do not know what we are expect for. In this case, OQL is a useful tools to get what you want by providing some search criteria.  

It seem different version of OQL has different syntax and I am trying to use the very common way to solve my issues.  Some document is quite useful which I want to have it attached in this page 

- [VisualVM](https://visualvm.java.net/oqlhelp.html "VisualVM Helper") 
- [Memory Analyzer](https://visualvm.java.net/oqlhelp.html "VisualVM Helper") 


### Base Usage ###

<pre>
select [target] from [classname] where [Query Criteria]
</pre>

for example 

<pre>
select s from java.lang.String s where s.count >= 100
select file.path.toString() from java.io.File file
</pre>


Here are an examples with I used to find those objects I am interesting in. 


- Find all threadpool with named 
<pre>
select x , x.threadFactory.threadName from java.util.concurrent.ThreadPoolExecutor x where classof(x.threadFactory).name.contains("Named")
</pre>

- Find all failed response in NAC 
<pre>
select x.exception.detailMessage from scala.util.Failure x
</pre>
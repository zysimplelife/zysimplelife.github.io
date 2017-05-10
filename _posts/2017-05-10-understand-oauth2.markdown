---
layout: post
title:  "My Understanding of OAuth2"
date: 2017-05-10 15:10:00 +0000
categories: OAuth2
---

In order to support role based authentication & authorization and make our system available to integration with external authentication system easily,  we have read several article about OAuth2 protocol before making our solution.  Here is the summary of my understanding so far, marking it for future study.

#### What is OAuth2  ####
From the the official description following, we can find OAuth2 is designed to provide third-part access capability between website. More specific example is like "Login in via Facebook".  Meanwhile, it can also be used very well within a website for authorization purpose. 
<pre>
	The OAuth 2.0 authorization framework enables a third-party
	application to obtain limited access to an HTTP service, either on
	behalf of a resource owner by orchestrating an approval interaction
	between the resource owner and the HTTP service, or by allowing the
	third-party application to obtain access on its own behalf.  This
	specification replaces and obsoletes the OAuth 1.0 protocol described
	in RFC 5849
</pre>


#### Protocol Flow ####

In order to understand how OAuth2 work, we mush understand the protocol flow and especially what they stand for. I will provide the concrete example from my understanding in this blog to make it easier to follow.
 
<pre>
 +--------+                               +---------------+
 |        |--(A)- Authorization Request ->|   Resource    |
 |        |                               |     Owner     |
 |        |<-(B)-- Authorization Grant ---|               |
 |        |                               +---------------+
 |        |
 |        |                               +---------------+
 |        |--(C)-- Authorization Grant -->| Authorization |
 | Client |                               |     Server    |
 |        |<-(D)----- Access Token -------|               |
 |        |                               +---------------+
 |        |
 |        |                               +---------------+
 |        |--(E)----- Access Token ------>|    Resource   |
 |        |                               |     Server    |
 |        |<-(F)--- Protected Resource ---|               |
 +--------+                               +---------------+
</pre>

Let's take a general use case, "Login via Facebook" as example. The first thing we have to do is putting different items into correspondingly place. In case an website named "MySite" want to provide feature of "Login via Facebook", it need work as an client to ask Facebook for authorization. 


- Client : MySite
- Resource Owner : User who has Facebook account
- Authorization Server : Facebook
- Resource Server : Facebook

After put all roles into right place, the process become quite easy to understand and can be described as follows

- "MySite" redirect to Facebook OAuth2 url and ask User input username&password
-  If everything goes well, authorization will response Access Token to "MySite"
-  Then "MySite" use this Token to as Facebook for User Information 
-  It is up to "MySite" how to use the User Information.


 





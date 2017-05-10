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


#### Authorization Grant ####

After understanding the general process, we should focus on the important part, Authorization Grant. The reason of saying it is the important part is because this is how client communicate with Authorization. In brief, it process includes Four types for different scenarios. 

- Authorization Code
- Implicit
- Resource Owner Password Credentials
- Client Credentials

##### Authorization Code #####

Authorization is considered as the most completed solution for granting authorization. It based on redirect flow and hide the token in client. The flow illustrated includes the following steps:

<pre>
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- & Redirection URI ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(D)-- Authorization Code ---------'      |
     |  Client |          & Redirection URI                  |
     |         |                                             |
     |         |<---(E)----- Access Token -------------------'
     +---------+       (w/ Optional Refresh Token)
	
</pre>


- Client : MySite
- User-Agent : Web Browser
- Resource Owner : User who has Facebook account
- Authorization Server : Facebook


Again, let put different roles into corresponding place as above and go thought the process base on that.

- MySite redirect URI to Facebook authentication Server and let User input username and password. 
- User use Browser to input username and password and Facebook return authorization code if everything goes well 
- Notice that the authorization code will be redirect by browser to MySite. 
- Then, MySite use this code to exchange for access token from authorization and continue access user information


**I want to high light the authorization code and why it seem more safe. The key point from my understanding is, Web Browser will never have the information of Access Token in such flow. Because exposure of Access Token make it become vulnerable.**



##### Implicit Grant #####

The implicit grant type is used for the scenario that you can't have client code run in server side.  These clients are typically implemented in a browser using a scripting language such as JavaScript. 

<pre>
     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier     +---------------+
     |         -+----(A)-- & Redirection URI --->|               |
     |  User-   |                                | Authorization |
     |  Agent  -|----(B)-- User authenticates -->|     Server    |
     |          |                                |               |
     |          |<---(C)--- Redirection URI ----<|               |
     |          |          with Access Token     +---------------+
     |          |            in Fragment
     |          |                                +---------------+
     |          |----(D)--- Redirection URI ---->|   Web-Hosted  |
     |          |          without Fragment      |     Client    |
     |          |                                |    Resource   |
     |     (F)  |<---(E)------- Script ---------<|               |
     |          |                                +---------------+
     +-|--------+
       |    |
      (A)  (G) Access Token
       |    |
       ^    v
     +---------+
     |         |
     |  Client |
     |         |
     +---------+
</pre>


- Client : MySite
- User-Agent : Web Browser
- Resource Owner : User who has Facebook account
- Authorization Server : Facebook
- Web-Hosted Client Resource : Java Script in MySite


Web-Hosted Client Resource could be some JavaScript code from MySite which can only run in Web Browser. the details work flow is similar with Authorization Code.


##### Resource Owner Password Credentials Grant #####

This method is used for the scenario that client and authorization server are in same organization. For example SSO solution for several web system in one site.  


<pre>
     +----------+
     | Resource |
     |  Owner   |
     |          |
     +----------+
          v
          |    Resource Owner
         (A) Password Credentials
          |
          v
     +---------+                                  +---------------+
     |         |>--(B)---- Resource Owner ------->|               |
     |         |         Password Credentials     | Authorization |
     | Client  |                                  |     Server    |
     |         |<--(C)---- Access Token ---------<|               |
     |         |    (w/ Optional Refresh Token)   |               |
     +---------+                                  +---------------+
</pre>


- Client : MySite
- Resource Owner : User who has Facebook account
- Authorization Server : Facebook

In this work flow, user will type username and password directly in MySite which will use it to apply for access token. **If so, MySite will have all the Facebook username password information for this User, which is quite unsafe**


##### Client Credentials Grant #####

This is the simplest but most unsafe way to get access token. It will be only used when the client is in same system. if you are working for a small system but want to an authorization feature, it is one of the good choice.

<pre>
     +---------+                                  +---------------+
     |         |                                  |               |
     |         |>--(A)- Client Authentication --->| Authorization |
     | Client  |                                  |     Server    |
     |         |<--(B)---- Access Token ---------<|               |
     |         |                                  |               |
     +---------+                                  +---------------+
</pre> 


- Client : MySite
- Authorization Server : Facebook


It is not validate for because will never provide this way to get access token. 





#### Summary ####

Above introduced all my understanding of the OAuth2 mechanism and what scenarios they should be used. Most of time, the first grant way is your best choice because it much more safe.  
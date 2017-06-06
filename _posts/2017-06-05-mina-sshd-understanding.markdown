---
layout: post
title:  "MINA-SSHD Understanding"
date: 2017-06-5 15:10:00 +0000
categories: Source Code
---

### MINA-SSHD ####

We were working for a new project to provide an ssh server demon receiving custom request. There are a good example [website](http://javajdk.net/tutorial/apache-mina-sshd-sshserver-example/) for us to start. But in order to understand better, I have to go thought the source code of MINA-SSHD and understand how it works. Following part is based on sshd 0.14 which is a very old one but is enough for us. 

### MINA-Architucture ####

Apache MINA-SSDH bases on MINA which is a network application framework which helps users develop high performance and high scalability network applications easily. It provides an abstract event-driven asynchronous API over various transports such as TCP/IP and UDP/IP via Java NIO. 

Before understanding how SSHD works we should understand how MINA-SSHD do. MINI framework provides the interface of IoHandler for user to customized there own business logic. There are a lot of [example](http://developer.51cto.com/art/201103/248125_all.htm) on this topic on how to create a new protocol based on MINA. Following picture show the work flow. 

![](https://www.planttext.com/plantuml/img/SoWkIImgAStDuU8goIp9ILNmzVHpL0ZFByfEoyalv-AA3aujAaij2ivCIOrLq2tAJCyeqLM8Tix9JCqh0GkYAR7HrRLJYFRCTqnEJYqeIIsAvKBcmANTCdDWab0cNCeZCIyb1oG7D1d97eWyI85pVbvUQd99FaGxWaUYuLnS3gbvAK2V0m00)
[EDIT](https://www.planttext.com/?text=SoWkIImgAStDuU8goIp9ILNmzVHpL0ZFByfEoyalv-AA3aujAaij2ivCIOrLq2tAJCyeqLM8Tix9JCqh0GkYAR7HrRLJYFRCTqnEJYqeIIsAvKBcmANTCdDWab0cNCeZCIyb1oG7D1d97eWyI85pVbvUQd99FaGxWaUYuLnS3gbvAK2V0m00)


Lets see a example as below 


```java
public static void main(String[] args) throws IOException {  
    // Create Acceptor   
    IoAcceptor acceptor = new NioSocketAcceptor();  
 
    // register filter   
   acceptor.getFilterChain().addLast( "logger", new LoggingFilter() );  
   acceptor.getFilterChain().addLast( "codec", new ProtocolCodecFilter( new TextLineCodecFactory( Charset.forName( "UTF-8" ))));  
 
    // register business logic    
    acceptor.setHandler( new TimeServerHandler() );  
 
    // configuration   
    acceptor.getSessionConfig().setReadBufferSize( 2048 );  
    acceptor.getSessionConfig().setIdleTime( IdleStatus.BOTH_IDLE, 10 );  
 
    // bing IP and port   
    acceptor.bind( new InetSocketAddress(PORT) );  
}  
```


IoFilter will handle low level stream, for example encode decode while handler will handle high level message. MINA has provided several Acceptor or and Filter for developer, so that we can pick up what we need and focus on business logic.  Let us have a look on IoHandler interface

```java
public interface IoHandler {  
 
     void sessionCreated(IoSession session) throws Exception;  
 
     void sessionOpened(IoSession session) throws Exception;  
 
     void sessionClosed(IoSession session) throws Exception;  
 
     void sessionIdle(IoSession session, IdleStatus status) throws Exception;  
 
     void exceptionCaught(IoSession session, Throwable cause) throws Exception;  
 
     void messageReceived(IoSession session, Object message) throws Exception;  
 
     void messageSent(IoSession session, Object message) throws Exception;  
}   

```

There are several session releated event will be trigger by mina for customnized protocol. for example messageSent is the major method to handle recieved message in server side. This Interface is very clear and easy to understand. What I want to know is how the message came to IoHandler


#### IoAccpeter and IoProcessor ####

Actually, in the MINA source code, IoAccepter work together with IoProcessor to dispatch request or message to above work flow. Take the default IoAccepter as example, the major logic locale on **AbstractPollingIoAcceptor**. A code snipplet from it show what happens when call the method **bind()** 

```java

protected final Set<SocketAddress> bindInternal(List<? extends SocketAddress> localAddresses) throws Exception {
        // Create a bind request as a Future operation. When the selector
        // have handled the registration, it will signal this future.
        AcceptorOperationFuture request = new AcceptorOperationFuture(localAddresses);

        // adds the Registration request to the queue for the Workers
        // to handle
        registerQueue.add(request);

        // creates the Acceptor instance and has the local
        // executor kick it off.
        startupAcceptor();

        // As we just started the acceptor, we have to unblock the select()
        // in order to process the bind request we just have added to the
        // registerQueue.
        try {
            lock.acquire();

            // Wait a bit to give a chance to the Acceptor thread to do the select()
            Thread.sleep(10);
            wakeup();
        } finally {
            lock.release();
        }

        // Now, we wait until this request is completed.
        request.awaitUninterruptibly();

        if (request.getException() != null) {
            throw request.getException();
        }

        // Update the local addresses.
        // setLocalAddresses() shouldn't be called from the worker thread
        // because of deadlock.
        Set<SocketAddress> newLocalAddresses = new HashSet<SocketAddress>();

        for (H handle : boundHandles.values()) {
            newLocalAddresses.add(localAddress(handle));
        }

        return newLocalAddresses;
    }

```


Notice that there will be another executor to launch Accepter to accpet message. Because the code is too long to put all of them, I pick up the most important part

```java
/**
     * This class is called by the startupAcceptor() method and is
     * placed into a NamePreservingRunnable class.
     * It's a thread accepting incoming connections from clients.
     * The loop is stopped when all the bound handlers are unbound.
     */
    private class Acceptor implements Runnable {
...
 /**
         * This method will process new sessions for the Worker class.  All
         * keys that have had their status updates as per the Selector.selectedKeys()
         * method will be processed here.  Only keys that are ready to accept
         * connections are handled here.
         * <p/>
         * Session objects are created by making new instances of SocketSessionImpl
         * and passing the session object to the SocketIoProcessor class.
         */
        @SuppressWarnings("unchecked")
        private void processHandles(Iterator<H> handles) throws Exception {
            while (handles.hasNext()) {
                H handle = handles.next();
                handles.remove();

                // Associates a new created connection to a processor,
                // and get back a session
                S session = accept(processor, handle);

                if (session == null) {
                    continue;
                }

                initSession(session, null, null);

                // add the session to the SocketIoProcessor
                session.getProcessor().add(session);
            }
        }
...
}
...
 /**
     * {@inheritDoc}
     */
    @Override
    protected NioSession accept(IoProcessor<NioSession> processor, ServerSocketChannel handle) throws Exception {

        SelectionKey key = handle.keyFor(selector);

        if ((key == null) || (!key.isValid()) || (!key.isAcceptable())) {
            return null;
        }

        // accept the connection from the client
        SocketChannel ch = handle.accept();

        if (ch == null) {
            return null;
        }

        return new NioSocketSession(this, processor, ch);
    }

...

```


From the code above we can see IoAccepter will return the a NioSocketSession which contains both IoProcessor and ServerSocketChannel. That means accepter only handle the new session coming from client but let IoPrecessor to handle the details operation. So, we should understand how IoProcessor works.







### Reference 

- [Apache Commons File Upload: ](https://commons.apache.org/proper/commons-fileupload/streaming.html)



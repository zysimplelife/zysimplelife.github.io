---
layout: post
title:  "MINA-SSHD Understanding"
date: 2017-06-5 15:10:00 +0000
categories: Source Code
---

### MINA-SSHD ####

We were working for a new project to provide an ssh server demon receiving custom request. There are a good example [website](http://javajdk.net/tutorial/apache-mina-sshd-sshserver-example/) for us to start. But in order to understand better, I have to go thought the source code of MINA-SSHD and understand how it works. Following part is based on sshd 0.14 which is a very old one but is enough for us. 

### MINA-Architucture ###

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
From the code **AbstractPollingIoProcessor** we can find it is a thread launched but another thread pool. Each Thread is in charge of pooling the Selector. Let us how it works

```java

 /**
     * The main loop. This is the place in charge to poll the Selector, and to
     * process the active sessions. It's done in
     * - handle the newly created sessions
     * -
     */
    private class Processor implements Runnable {
        public void run() {
            assert (processorRef.get() == this);

            int nSessions = 0;
            lastIdleCheckTime = System.currentTimeMillis();

            for (;;) {
                try {
                   ... Code to handle timeout ...

                    // Manage newly created session first
                    nSessions += handleNewSessions();

                    updateTrafficMask();

                    // Now, if we have had some incoming or outgoing events,
                    // deal with them
                    if (selected > 0) {
                        //LOG.debug("Processing ..."); // This log hurts one of the MDCFilter test...
                        process();
                    }

                    // Write the pending requests
                    long currentTime = System.currentTimeMillis();
                    flush(currentTime);

                    // And manage removed sessions
                    nSessions -= removeSessions();

                    // Last, not least, send Idle events to the idle sessions
                    notifyIdleSessions(currentTime);

                    // Get a chance to exit the infinite loop if there are no
                    // more sessions on this Processor
                    if (nSessions == 0) {
                        .. exit if no more session
                    }

                    // Disconnect all sessions immediately if disposal has been
                    // requested so that we exit this loop eventually.
                    if (isDisposing()) {
                        ..dispose
                    }
                } catch (ClosedSelectorException cse) {
                    ...
                } catch (Throwable t) {
                    ...
                }
            }

            ..dispose
        }
    }

```

Above code is is quite clear that each process would handle several sessions operation. it will exist if all the session is closed or disposing method is called. Processor handle 3 types of message:create new sesson, remove closed session and process incoming message. The first two part is quite easy so let look at how to process message

```java
	/**
     * Deal with session ready for the read or write operations, or both.
     */
    private void process(S session) {
        // Process Reads
        if (isReadable(session) && !session.isReadSuspended()) {
            read(session);
        }

        // Process writes
        if (isWritable(session) && !session.isWriteSuspended()) {
            // add the session to the queue, if it's not already there
            if (session.setScheduledForFlush(true)) {
                flushingSessions.add(session);
            }
        }
    }


	 private void read(S session) {
        ...

        try {
            int readBytes = 0;
            int ret;
            ...
            ret = read(session, buf);
            ...
         
            if (readBytes > 0) {
                IoFilterChain filterChain = session.getFilterChain();
                filterChain.fireMessageReceived(buf);
                ...
            }

            if (ret < 0) {
                scheduleRemove(session);
            }
        } catch (Throwable e) {
            ...
            IoFilterChain filterChain = session.getFilterChain();
            filterChain.fireExceptionCaught(e);
        }
    }

	**@Override
    protected int read(NioSession session, IoBuffer buf) throws Exception {
        ByteChannel channel = session.getChannel();
        return channel.read(buf.buf());
    }**

```

From the above code we can found processor will call the filter chain metioned at begining to handle input message which stored in the buffer. But how the handler to be called?  That is because there is an default filter named **TailFilter** which will call handler

```java
private static class TailFilter extends IoFilterAdapter {
...
	@Override
	public void messageReceived(NextFilter nextFilter, IoSession session, Object message) throws Exception {
	    AbstractIoSession s = (AbstractIoSession) session;
	    if (!(message instanceof IoBuffer)) {
	        s.increaseReadMessages(System.currentTimeMillis());
	    } else if (!((IoBuffer) message).hasRemaining()) {
	        s.increaseReadMessages(System.currentTimeMillis());
	    }
	
	    try {
	        session.getHandler().messageReceived(s, message);
	    } finally {
	        if (s.getConfig().isUseReadOperation()) {
	            s.offerReadFuture(message);
	        }
	    }
	}
...
}
```

So far I have understanded how the message works in the MINA server side, it is better to provide a dragram to descripte the whole work flow, helping us under stand how mina-sshd works later. Because I am not familar how NIO works, all the descirption hasn't went into details which I need make up in future. Anyway,  It could be a good start

![](https://www.planttext.com/plantuml/img/XP7TJiCm38Nl_HHMTmCNVO4AeMqWnAI19lO4QM9QYpJk4fUVjoVGJL1rY3idvtpsYRDCQg8EdGTGLazOxEamKB24jsoQQBe201vPzc9VI5VMKgyIiIolSLKdZSRgJhpdq6paf5Or1mT_oYDyyc8VnL9AzoOuJmachldW2irt-UExAikplg-xt9Sb0DJoZiLMf4SEg6qaumfSRBbfTUq7WddOtPZgc6CZT-oLuarhSeCAdpdIGvPDGqzaYL_9NNJZ-HAcvjzu9hifXNFiI8pxE8Fy6tRzePHdXq0-qs-HDJ-GWiEy1O1bhl9_Vm80)


[EDIT](https://www.planttext.com/?text=XP7TJiCm38Nl_HHMTmCNVO4AeMqWnAI19lO4QM9QYpJk4fUVjoVGJL1rY3idvtpsYRDCQg8EdGTGLazOxEamKB24jsoQQBe201vPzc9VI5VMKgyIiIolSLKdZSRgJhpdq6paf5Or1mT_oYDyyc8VnL9AzoOuJmachldW2irt-UExAikplg-xt9Sb0DJoZiLMf4SEg6qaumfSRBbfTUq7WddOtPZgc6CZT-oLuarhSeCAdpdIGvPDGqzaYL_9NNJZ-HAcvjzu9hifXNFiI8pxE8Fy6tRzePHdXq0-qs-HDJ-GWiEy1O1bhl9_Vm80)


### MINA-SSHD###
While reading MINA-SSHD, I was focusing on how MINA-SSHD extend MINA and how it provide API to customzation. Let start from a code example on how to provide a SSHD demo based on MINA-SSHD

```java

public class SshSessionInstance 
        implements Command, Runnable {
  
  //ANSI escape sequences for formatting purposes 
  ...
  //IO streams for communication with the client
  private InputStream is;
  private OutputStream os;
  
  //Environment stuff
  @SuppressWarnings("unused")
  private Environment environment;
  private ExitCallback callback;
  
  private Thread sshThread;
  
  @Override
  public void start(Environment env) throws IOException 
  {    
    //must start new thread to free up the input stream
    environment = env;
    sshThread = new Thread(this, "EchoShell");
    sshThread.start();
  }

  @Override
  public void run() 
  {
    BufferedReader br = new BufferedReader(new InputStreamReader(is));
    
    //Make sure local echo is on (because password turned it off
   
      os.write(WELECOME_MESSAGE);
      os.flush();
    try {
      
      boolean exit = false;
      String text;
      while ( ! exit ) 
      {
        text = br.readLine();
        ... Handle text 
      }
    } catch (Exception e) {
        ...
    } finally {
        callback.onExit(0);
    }
  }

  @Override
  public void destroy() throws Exception 
  {
    sshThread.interrupt();
  }

 ...

}


```

If we based on MINA-SSHD we could only provide the implemtation of **Command** to handle input and output Stream. It is quite convience becacuse framework has wrapper a lot of steps for use including the authentication, timeout and so on. I want to understand a little more about how it works, that why I need read the code details.





### Reference 

- [Apache Commons File Upload: ](https://commons.apache.org/proper/commons-fileupload/streaming.html)



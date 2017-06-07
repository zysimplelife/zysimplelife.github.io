---
layout: post
title:  "MINA-SSHD source code Understanding"
date: 2017-06-5 15:10:00 +0000
categories: Source Code
---

### MINA-SSHD ####

We were working for a new project to provide an ssh server demon receiving custom request. There is a good example [website](http://javajdk.net/tutorial/apache-mina-sshd-sshserver-example/) for us to start. But in order to understand how it worked, I had to go thought the source code of MINA-SSHD. Following part is based on sshd 0.14 which is a very old one but enough for us. 

### MINA-Architucture ###

Apache MINA-SSHD bases on MINA, a network application framework which helps users develop high performance and high scalability network applications easily. It provides an abstract event-driven asynchronous API over various transports such as TCP/IP and UDP/IP via Java NIO. 

Before understanding how SSHD works we should understand how MINA-SSHD do. MINI framework provides the interface of IoHandler for user to customized there own business logic. There are a lot of [example](http://developer.51cto.com/art/201103/248125_all.htm) on this topic on how to create a new protocol based on MINA. Following picture show the work flow. 

![](https://www.planttext.com/plantuml/img/SoWkIImgAStDuU8goIp9ILNmzVHpL0ZFByfEoyalv-AA3aujAaij2ivCIOrLq2tAJCyeqLM8Tix9JCqh0GkYAR7HrRLJYFRCTqnEJYqeIIsAvKBcmANTCdDWab0cNCeZCIyb1oG7D1d97eWyI85pVbvUQd99FaGxWaUYuLnS3gbvAK2V0m00)
[EDIT](https://www.planttext.com/?text=SoWkIImgAStDuU8goIp9ILNmzVHpL0ZFByfEoyalv-AA3aujAaij2ivCIOrLq2tAJCyeqLM8Tix9JCqh0GkYAR7HrRLJYFRCTqnEJYqeIIsAvKBcmANTCdDWab0cNCeZCIyb1oG7D1d97eWyI85pVbvUQd99FaGxWaUYuLnS3gbvAK2V0m00)


Let's see a example as below 


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


IoFilter will handle low level stream, for example encode decode while handler will handle high level message. MINA has provided several Acceptors or and Filters for developer, so that we can pick up what we need and focus on business logic.  Let us have a look on IoHandler interface

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

There are several session releated event will be trigger by MINA for customized protocol. This Interface is very clear and easy to understand. What I want to know is how the message came to IoHandler


#### IoAccpeter and IoProcessor ####

Actually, in the MINA source code, IoAccepter work together with IoProcessor to dispatch request or message to above work flow. Take the default IoAccepter as example, the major logic locales on "AbstractPollingIoAcceptor". A code snipplet from it show what happens when call the method "bind()"

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
        ...
		...

        for (H handle : boundHandles.values()) {
            newLocalAddresses.add(localAddress(handle));
        }

        return newLocalAddresses;
    }

```


Notice that there will be another executor to launch Accepter to accpet request as following code sniplet

```java
    private class Acceptor implements Runnable {
		...

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
Abstract class "AbstractPollingIoProcessor" is also an Runnable which will be launched but another thread pool. Each Thread is in charge of pooling the Selector. The int returned by the select() methods tells how many channels are ready. That is, how many channels that became ready since last time you called select(). If you call select() and it returns 1 because one channel has become ready, and you call select() one more time, and one more channel has become ready, it will return 1 again. If you have done nothing with the first channel that was ready, you now have 2 ready channels, but only one channel had become ready between each select() call.  Let us how it works

```java

 /**
     * The main loop. This is the place in charge to poll the Selector, and to
     * process the active sessions. It's done in
     * - handle the newly created sessions
     * -
     */
    private class Processor implements Runnable {
        public void run() {
            ...

            for (;;) {
                try {
                   ... Code to handle timeout ...

                  
                   	...
					
					//got select selector
					int selected = select(SELECT_TIMEOUT);
 					...

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

Above code is is quite clear that each process would handle several sessions operation. it will exit if all the session is closed or disposing method is called. Processor handle 3 types of message:create new sesson, remove closed session and process incoming message. The first two part is quite easy so let look at how to process message

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

From the above code we can found processor will call the filter chain metioned at begining to handle input message which stored in the buffer. But how the handler to be called?  That is because there is an default filter named "TailFilter" which will call handler

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

So far we have understanded how the message worked in the MINA server side, but it is better to provide a dragram to descripte the whole work flow, helping us under stand how mina-sshd works later. Because I am not familar how NIO works, all the descirption hasn't went into details which I need make up in future. Anyway,  It could be a good start

![](https://www.planttext.com/plantuml/img/XP7TJiCm38Nl_HHMTmCNVO4AeMqWnAI19lO4QM9QYpJk4fUVjoVGJL1rY3idvtpsYRDCQg8EdGTGLazOxEamKB24jsoQQBe201vPzc9VI5VMKgyIiIolSLKdZSRgJhpdq6paf5Or1mT_oYDyyc8VnL9AzoOuJmachldW2irt-UExAikplg-xt9Sb0DJoZiLMf4SEg6qaumfSRBbfTUq7WddOtPZgc6CZT-oLuarhSeCAdpdIGvPDGqzaYL_9NNJZ-HAcvjzu9hifXNFiI8pxE8Fy6tRzePHdXq0-qs-HDJ-GWiEy1O1bhl9_Vm80)


[EDIT](https://www.planttext.com/?text=XP7TJiCm38Nl_HHMTmCNVO4AeMqWnAI19lO4QM9QYpJk4fUVjoVGJL1rY3idvtpsYRDCQg8EdGTGLazOxEamKB24jsoQQBe201vPzc9VI5VMKgyIiIolSLKdZSRgJhpdq6paf5Or1mT_oYDyyc8VnL9AzoOuJmachldW2irt-UExAikplg-xt9Sb0DJoZiLMf4SEg6qaumfSRBbfTUq7WddOtPZgc6CZT-oLuarhSeCAdpdIGvPDGqzaYL_9NNJZ-HAcvjzu9hifXNFiI8pxE8Fy6tRzePHdXq0-qs-HDJ-GWiEy1O1bhl9_Vm80)


### MINA-SSHD###
While reading MINA-SSHD, I was focusing on how MINA-SSHD extended MINA and how it provided API for customization. Let start from a code example on how to provide a SSHD demo based on MINA-SSHD

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

If we based on MINA-SSHD we need only provide the implemtation of "Command" to handle input and output Stream. It is quite convience becacuse framework will wrapper a lot of steps for us including the authentication, timeout and so on. 


### SSHD Start Code###

Besides the interface of MINA, sshd provided several similar interfaces which need be noticed.  I think all those new interface is to let sshd can not only work based on MINA but also on other types of network application framework or even NIO directly.  Anyway, There are following interface have same name with MINA which made me confused at begin of reading code.


- IoAcceptor

```Java
public interface IoAcceptor extends IoService {

    void bind(Collection<? extends SocketAddress> addresses) throws IOException;

    void bind(SocketAddress address) throws IOException;

    void unbind(Collection<? extends SocketAddress> addresses);

    void unbind(SocketAddress address);

    void unbind();

    Set<SocketAddress> getBoundAddresses();

}
```


- IoHandler

```Java
public interface IoHandler {

    void sessionCreated(IoSession session) throws Exception;

    void sessionClosed(IoSession session) throws Exception;

    void exceptionCaught(IoSession ioSession, Throwable cause) throws Exception;

    void messageReceived(IoSession session, Readable message) throws Exception;

}
```


After noticing above new interface, we can start from the source code. All the sshd server start from the method "start()" to init two object Session Factory and Acceptor in the Class SshServer.  According to the configuration, sshd server can work based on either NiO or MINA.  SshClient is another implementation to work as ssh client. I will not cover it in this article. 

```Java

/**
     * Start the SSH server and accept incoming exceptions on the configured port.
     * 
     * @throws IOException
     */
    public void start() throws IOException {
        checkConfig();
        if (sessionFactory == null) {
            sessionFactory = createSessionFactory();
        }
        sessionFactory.setServer(this);
        acceptor = createAcceptor();

        setupSessionTimeout(sessionFactory);

        ..Check host ...
        acceptor.bind(new InetSocketAddress(port));
           
    }

```

- SessionFactory : It is also an IoHandler of sshd to handle message
- IoAcceptor :  It is a wrapper of MiNA IoAccepter when working based on MINA framework. 

 
Similar to MINA Concept, Acceptor here is used to accept input message then dispatch to session to handle. It is actually an wrapper of MINA IoAcceptor if using MiNA framework. Here is the part of code snippet on how to wrapper it and how it is eventually use a NioSocketAcceptor and SimpleIoProcessorPool to bind SocketAddress as Default MiNA framwork does.


```java
public class MinaAcceptor extends MinaService implements org.apache.sshd.common.io.IoAcceptor, IoHandler {
...
this.ioProcessor = new SimpleIoProcessorPool<NioSession>(NioProcessor.class, getNioWorkers());
...
protected IoAcceptor createAcceptor() {
        NioSocketAcceptor acceptor = new NioSocketAcceptor(ioProcessor);
        acceptor.setCloseOnDeactivation(false);
        acceptor.setReuseAddress(reuseAddress);
        acceptor.setBacklog(backlog);
        configure(acceptor.getSessionConfig());
        return acceptor;
    }

 public void bind(SocketAddress address) throws IOException {
    getAcceptor().bind(address);
}
...
}
```

MinaAcceptor extends MinaService which is also an wrapper of MiNA IoHandler. 

```java
public abstract class MinaService extends IoHandlerAdapter implements org.apache.sshd.common.io.IoService, IoHandler, Closeable {
...
public void sessionCreated(IoSession session) throws Exception {
        org.apache.sshd.common.io.IoSession ioSession = new MinaSession(this, session);
        session.setAttribute(org.apache.sshd.common.io.IoSession.class, ioSession);
        handler.sessionCreated(ioSession);
    }

    public void sessionClosed(IoSession session) throws Exception {
        handler.sessionClosed(getSession(session));
    }

    public void exceptionCaught(IoSession session, Throwable cause) throws Exception {
        handler.exceptionCaught(getSession(session), cause);
    }

    public void messageReceived(IoSession session, Object message) throws Exception {
        handler.messageReceived(getSession(session), MinaSupport.asReadable((IoBuffer) message));
    }
...
}

```



So far we have got that the integration between SSHD and MINA framework is quite simple. SSHD used Factory and Strategy pattern to support both MINA and NIO as the foundation. To make it more clear, I provided the Corresponding diagram between SSHD and MINA

![](https://www.planttext.com/plantuml/img/RP3F2i8m3CRlUOgm-zv0nE4GFMmCTXGFfGkph6j6MnKHtzqEdQj_Uag_V7o_92ldXVMdNWDuvJLX9MGdMdAOufhxWGqPZxaIhHKzmF3iObAeCalm1XZUViUPb3HujeT9g2nBSYvIji8qciB_FiVKxfXF8QNYccL7_YkhLlsWAKgicFNK2u9Yin4o-AyliL16r6JFIj88GvZ7mqNQyCMaIyGV74I8sVUN3kzjPcD4XQZ6T0pv61DWHQO99ty0)
[EDIT](https://www.planttext.com/?text=RP3F2i8m3CRlUOgm-zv0nE4GFMmCTXGFfGkph6j6MnKHtzqEdQj_Uag_V7o_92ldXVMdNWDuvJLX9MGdMdAOufhxWGqPZxaIhHKzmF3iObAeCalm1XZUViUPb3HujeT9g2nBSYvIji8qciB_FiVKxfXF8QNYccL7_YkhLlsWAKgicFNK2u9Yin4o-AyliL16r6JFIj88GvZ7mqNQyCMaIyGV74I8sVUN3kzjPcD4XQZ6T0pv61DWHQO99ty0)



SessionFactory takes most of the take to handle ssh session. It response to create ssh session which is inherited from abstract class "AbstractSession" which handles all the basic SSH protocol such as key exchange, authentication, encoding and decoding. Both server side and client side sessions should inherit from this abstract class. It is a very big but important class which I will spilt it into different part.

Firstly, it is nested in MINA session and should be retrieved every time handle message as follows 

```java
/**
     * Retrieve the session from the MINA session.
     * If the session has not been attached and allowNull is <code>false</code>,
     * an IllegalStateException will be thrown, else a <code>null</code> will
     * be returned
     *
     * @param ioSession the MINA session
     * @param allowNull if <code>true</code>, a <code>null</code> value may be
     *        returned if no session is attached
     * @return the session attached to the MINA session or <code>null</code>
     */
    public static AbstractSession getSession(IoSession ioSession, boolean allowNull) {
        AbstractSession session = (AbstractSession) ioSession.getAttribute(SESSION);
        if (!allowNull && session == null) {
            throw new IllegalStateException("No session available");
        }
        return session;
    }

	/**
     * Attach a session to the MINA session
     *
     * @param ioSession the MINA session
     * @param session the session to attach
     */
    public static void attachSession(IoSession ioSession, AbstractSession session) {
        ioSession.setAttribute(SESSION, session);
    }
```

Secondly, message is encrypted during transforming in ssh protocol. This class provided encode and decode method to handle it. 

```java
    private void encode(Buffer buffer) throws IOException {
		...
	}
   protected void decode() throws Exception {
		...
	}
``` 

Thirdly, this class do not provide authentication part but only have the interface to set if this session has been authenticated

```java
	public void setAuthenticated() throws IOException {
        this.authed = true;
        sendEvent(SessionListener.Event.Authenticated);
    }
	public boolean isAuthenticated() {
        return authed;
    }
```

Lastly, the most important part is handle input message
```java
protected void doHandleMessage(Buffer buffer) throws Exception {
        byte cmd = buffer.getByte();
        switch (cmd) {
            case SSH_MSG_DISCONNECT: {
                ...disconnect
            }
            case SSH_MSG_IGNORE: {
                ...ignore
            }
            case SSH_MSG_UNIMPLEMENTED: {
                ...
            }
            case SSH_MSG_DEBUG: {
                ...print debug message
            }
            case SSH_MSG_SERVICE_REQUEST:
                ... start service according to reference https://www.ietf.org/rfc/rfc4253.txt
				... The default service in SSHD is connection and user authentication
                startService(service);
                ... write message back
                writePacket(response);
                
            case SSH_MSG_SERVICE_ACCEPT:
                ... shouldn't happend in server side 
            case SSH_MSG_KEXINIT:
                ... do key exchange
            case SSH_MSG_NEWKEYS:
                ... do key exchange
            default:
                ... handle message
                currentService.process(cmd, buffer);
                resetIdleTimeout();
                ...
        }
        checkRekey();
    }
```

According to rfc4253, default ssh service are authentication and connection service. authentication service is used to authenticate user before making the connection. It very important but quite straght forword, so we are going to focus on the connection service. SSHD connection service is provided by abstract class "AbstractConnectionService" which is inherited by both client and service implementation. Let continue reading the most important code 

```java
public void process(byte cmd, Buffer buffer) throws Exception {
        switch (cmd) {
            case SSH_MSG_CHANNEL_OPEN:
                channelOpen(buffer);
                break;
            case SSH_MSG_CHANNEL_OPEN_CONFIRMATION:
                channelOpenConfirmation(buffer);
                break;
            case SSH_MSG_CHANNEL_OPEN_FAILURE:
                channelOpenFailure(buffer);
                break;
            case SSH_MSG_CHANNEL_REQUEST:
                channelRequest(buffer);
                break;
            case SSH_MSG_CHANNEL_DATA:
                channelData(buffer);
                break;
            case SSH_MSG_CHANNEL_EXTENDED_DATA:
                channelExtendedData(buffer);
                break;
            case SSH_MSG_CHANNEL_FAILURE:
                channelFailure(buffer);
                break;
            case SSH_MSG_CHANNEL_WINDOW_ADJUST:
                channelWindowAdjust(buffer);
                break;
            case SSH_MSG_CHANNEL_EOF:
                channelEof(buffer);
                break;
            case SSH_MSG_CHANNEL_CLOSE:
                channelClose(buffer);
                break;
            case SSH_MSG_GLOBAL_REQUEST:
                globalRequest(buffer);
                break;
            case SSH_MSG_REQUEST_SUCCESS:
                requestSuccess(buffer);
                break;
            case SSH_MSG_REQUEST_FAILURE:
                requestFailure(buffer);
                break;
            default:
                throw new IllegalStateException("Unsupported command: " + cmd);
        }
    }
```

From the code above we can see this service handle two types of messge: channel and request. In the SSH Protocol service can create one or more channel to reuse a connection or session according to the [description](https://tools.ietf.org/html/rfc4254). Each channel has a type. Usually, you will use “session” channels, but there are also “x11” channels, “forwarded-tcpip” channels, and “direct-tcpip” channels. 

Firstlly, Let see how to create a channel. There are several channel type and session channel is enough for us to create the ssh daemon. 

```java
public abstract class AbstractConnectionService extends CloseableUtils.AbstractInnerCloseable implements ConnectionService {
/** Map of channels keyed by the identifier */
    protected final Map<Integer, Channel> channels = new ConcurrentHashMap<Integer, Channel>();
...
protected void channelOpen(Buffer buffer) throws Exception {
...
 		final Channel channel = NamedFactory.Utils.create(session.getFactoryManager().getChannelFactories(), type);

        ...

        final int channelId = registerChannel(channel);
        channel.open(id, rwsize, rmpsize, buffer).addListener(new SshFutureListener<OpenFuture>() {
            public void operationComplete(OpenFuture future) {
                try {
                    if (future.isOpened()) {
						// write back channel ID to client
                        Buffer buf = session.createBuffer(SshConstants.SSH_MSG_CHANNEL_OPEN_CONFIRMATION);
                        buf.putInt(id);
                        buf.putInt(channelId);
                        buf.putInt(channel.getLocalWindow().getSize());
                        buf.putInt(channel.getLocalWindow().getPacketSize());
                        session.writePacket(buf);
                    } else {
                        ... failed to write back
                    }
                } catch (IOException e) {
                    session.exceptionCaught(e);
                }
            }
        });
...
	}
}
```


After the channel is been created, client will send request with channel id to server. ChannelSession is the implementation in MINA-SSHD. It actually can handle several types of request 

```java
 public Boolean handleRequest(String type, Buffer buffer) throws IOException {
        if ("env".equals(type)) {
            return handleEnv(buffer);
        }
        if ("pty-req".equals(type)) {
            return handlePtyReq(buffer);
        }
        if ("window-change".equals(type)) {
            return handleWindowChange(buffer);
        }
        if ("signal".equals(type)) {
            return handleSignal(buffer);
        }
        if ("break".equals(type)) {
            return handleBreak(buffer);
        }
        if ("shell".equals(type)) {
            if (this.type == null && handleShell(buffer)) {
                this.type = type;
                return true;
            } else {
                return false;
            }
        }
        if ("exec".equals(type)) {
            if (this.type == null && handleExec(buffer)) {
                this.type = type;
                return true;
            } else {
                return false;
            }
        }
        if ("subsystem".equals(type)) {
            if (this.type == null && handleSubsystem(buffer)) {
                this.type = type;
                return true;
            } else {
                return false;
            }
        }
        if ("auth-agent-req@openssh.com".equals(type)) {
            return handleAgentForwarding(buffer);
        }
        if ("x11-req".equals(type)) {
            return handleX11Forwarding(buffer);
        }
        return null;
    }
```

if we need customsize shell behavior we can found how sshd call our method 

```java
	protected boolean handleShell(Buffer buffer) throws IOException {
        ...
        command = ((ServerSession) session).getFactoryManager().getShellFactory().create();
		/**
		In this method, it will set input/output stream and session context in the command
		*/
        prepareCommand();
        command.start(getEnvironment());
        return true;
    }

```

TODO: describe how the pipe stream work together with channel

![](https://www.planttext.com/plantuml/img/TPF1QiCm38RlUWhHUzvWZ8QMiJ5Q0c6diOFdYiRKjeBjT7HZxpudJHedItEAfF__xNmYQn-42utH0445JLW8UH97yfZXXatDbcp0hH979mn0VPtYQgVs-Gf_0EFp_iAvb5G7TXz3et0ioVkayopiGLEiVyUOqbVR8MIlk6HveZ3BAfMfDIM91RCUPh6Xs3u96VMNlhbJLfJapafItya_VN1HqyjlPdScz-R9vKseUeUVMJPiBSaGNTF8JINYGDyIau-IZGzirBTeNSFNbHLfFRrdn6iYaxvwfKjx3R9ATiOs4c4aYm_PWRzizeZuZnGaT4RT8ZYuBM8K9i0WUSUi3PaG1fZMdMH65sPr7xF4Ub5wbppSdNI-wKRQcouTsKddh67gxJIORWpIudhQTNa0i2PxYF_F7m00)

[Edit](https://www.planttext.com/?text=TPF1QiCm38RlUWhHUzvWZ8QMiJ5Q0c6diOFdYiRKjeBjT7HZxpudJHedItEAfF__xNmYQn-42utH0445JLW8UH97yfZXXatDbcp0hH979mn0VPtYQgVs-Gf_0EFp_iAvb5G7TXz3et0ioVkayopiGLEiVyUOqbVR8MIlk6HveZ3BAfMfDIM91RCUPh6Xs3u96VMNlhbJLfJapafItya_VN1HqyjlPdScz-R9vKseUeUVMJPiBSaGNTF8JINYGDyIau-IZGzirBTeNSFNbHLfFRrdn6iYaxvwfKjx3R9ATiOs4c4aYm_PWRzizeZuZnGaT4RT8ZYuBM8K9i0WUSUi3PaG1fZMdMH65sPr7xF4Ub5wbppSdNI-wKRQcouTsKddh67gxJIORWpIudhQTNa0i2PxYF_F7m00)




Above diagram shows how the whole workflow works. From it we can understand that to implement the interface "Command" can we customized the behavior of a shell "Command". 

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




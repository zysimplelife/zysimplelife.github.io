---
layout: post
title:  "How to test multi-thread in JUnit"
date: 2017-08-07 15:10:00 +0000
categories: Java
---

### Introduction
Multi thread should be test in properly, however it is not easy to create such test in Junit.  Creating it from scratch would cost a lot of time, so I tend to skip such test. So I add some code snippet here to remind myself when I need. 

### code example 
 
```java
public class MultiThreadTestLoader {
    private static final Logger LOGGER = LoggerFactory.getLogger(MultiThreadTestLoader.class);

    public static interface ThrowingRunnable extends Runnable{
        @Override
        default void run() {
            try {
                acceptThrows();
            } catch (final Exception e) {
                throw new RuntimeException(e);
            }
        }

        void acceptThrows() throws Exception;
    }

    public void runInMultiThread(int thread,ThrowingRunnable f) throws Exception {
        final long startTime = System.currentTimeMillis();
        final int numThreads = thread;

        final List<Throwable> exceptions = Collections.synchronizedList(new ArrayList<>());
        final ExecutorService threadPool = Executors.newFixedThreadPool(numThreads);

        try {
            final CountDownLatch allExecutorThreadsReady = new CountDownLatch(numThreads);
            final CountDownLatch afterInitBlocker = new CountDownLatch(1);
            final CountDownLatch allDone = new CountDownLatch(numThreads);

            for (int i = 0; i < numThreads; i++) {
                threadPool.submit(() -> {
                    try {
                        allExecutorThreadsReady.countDown();
                        afterInitBlocker.await();
                        f.acceptThrows();
                    } catch (final Throwable e) {
                        exceptions.add(e);
                    } finally {
                        allDone.countDown();
                    }
                });
            }
            assertTrue("Timeout initializing threads! Perform long lasting initializations before passing runnables to assertConcurrent", allExecutorThreadsReady.await(numThreads * 10, TimeUnit.MILLISECONDS));
            // start all test runners
            afterInitBlocker.countDown();

            assertTrue(" timeout! More than" + numThreads * 10 + "seconds", allDone.await(numThreads * 10, TimeUnit.SECONDS));
        } finally {
            threadPool.shutdownNow();
        }

        final long duration = System.currentTimeMillis() - startTime;
        LOGGER.info("final time cost {}", duration);
        LOGGER.debug("Find Exception " + exceptions.size());

        for (Throwable ex : exceptions) {
            LOGGER.error("Got exception",ex);
        }

        assertTrue("failed with exception(s)" + exceptions, exceptions.isEmpty());
    }
}

```


### Usage 

```java

@Test
public void testMultiThread() throws Exception {
    // the way after java 8 
    new MultiThreadTestLoader().runInMultiThread(100, () -> mapExternalError());
}
```


### TODO 

This code should be more easy to use. Maybe a library to test multi thread for myself is needed





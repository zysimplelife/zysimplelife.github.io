---
layout: post
title:  "How to test multi-thread in junit"
date: 2017-08-07 15:10:00 +0000
categories: Java
---

### Introduction
Multi thread should be test in properly, however it is not easy to create such test in Junit.  Creating it from scratch would cost a lot of time, so I tend to skip such test. So I add some code snippet here to remind myself when I need. 

### code example 
 
```java
package com.ericsson.javadataview.examplenl.mml;

import com.ericsson.dve.util.xml.XMLUtil;
import com.ericsson.jdv.dup.mos.FDS.FDSStandardException;
import com.ericsson.jdv.fw.resource.ResourceContext;
import org.junit.Test;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static junit.framework.Assert.assertEquals;
import static org.junit.Assert.assertTrue;


public class ParseCASubscriptionDataTest {

    private String giveResp() {
        return "DP=OCI,ACT=A,SCP=5678,SKEY=.9999,DEFCH=R,CAP=3,ADDDNT=I358501112304&I358501112305&U5747893&U5747894,DNL=8&9&10,TP=E,FCC=MFC,BSC=T10::::LOWPH=1,LOWAC=C,USSD=7,TIF=Y:OUT=N;";
    }

    @Test
    public void testMain() throws FDSStandardException {
        new ParseCASubscriptionData().main(giveResp());
        new ParseCASubscriptionData().main(giveResp());
        assertEquals("1", XMLUtil.getFirstValueByName(ResourceContext.getRuntime().getOutputPayload().getDocumentElement(), "prepaid"));

    }


    @Test
    public void testMultiThread() throws Exception {
        final long startTime = System.currentTimeMillis();
        final int numThreads = 256;
        final int rounds = 100;
        final int timeout =  numThreads * rounds;
        final List<Throwable> exceptions = Collections.synchronizedList(new ArrayList<Throwable>());
        final ExecutorService threadPool = Executors.newFixedThreadPool(numThreads);
        try {
            final CountDownLatch allExecutorThreadsReady = new CountDownLatch(numThreads);
            final CountDownLatch afterInitBlocker = new CountDownLatch(1);
            final CountDownLatch allDone = new CountDownLatch(numThreads);
            for (int i = 0; i < numThreads; i++) {
                threadPool.submit(new Runnable() {
                    public void run() {
                        try {
                            allExecutorThreadsReady.countDown();
                            afterInitBlocker.await();
                            System.out.println("Thread "+ Thread.currentThread() + " start");
                            for (int j = 0; j < rounds; j++) {
                                new ParseCASubscriptionData().main(giveResp());
                            }
                        } catch (final Throwable e) {
                            exceptions.add(e);
                        } finally {
                            System.out.println("Thread "+ Thread.currentThread() + " finished");
                            allDone.countDown();
                        }
                    }
                });
            }
            assertTrue("Timeout initializing threads! Perform long lasting initializations before passing runnables to assertConcurrent", allExecutorThreadsReady.await(numThreads, TimeUnit.SECONDS));            // start all test runners
            afterInitBlocker.countDown();
            assertTrue(" timeout! More than" +timeout + "seconds", allDone.await(timeout, TimeUnit.SECONDS));
        } finally {
            threadPool.shutdownNow();
        }
        final long duration = System.currentTimeMillis() - startTime;
        System.out.println("Find Exception " + exceptions.size());
        for (Throwable ex : exceptions) {
            ex.printStackTrace();
        }
        assertTrue("failed with exception(s)" + exceptions, exceptions.isEmpty());
    }


}

```


### TODO 

This code should be more easy to use. Maybe a library to test multi thread for myself is needed





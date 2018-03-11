---
title: RxJava使用线程池
date: 2018-03-11 13:19:47
tags: java,rxjava
---


最近在瞎折腾rxjava，写了一段自认为能并发执行的代码如下：

```java
// 大小为5的线程池
ThreadPoolExecutor exec = new ThreadPoolExecutor(
    5, 5, 200, TimeUnit.SECONDS, new LinkedBlockingQueue<>());

Flowable.just(1, 2, 3, 4, 5)
        .subscribeOn(Schedulers.from(exec))
        .subscribe(i -> {
            Thread.sleep(1000L);
            System.out.println(i + "\t" + Thread.currentThread().getName());
        });
```

由于我在subscribe中sleep了1s，所以我认为这五个数字会并发的执行到subscribe中去，期待会有如下的输出

```
1	pool-1-thread-1
2	pool-1-thread-2
3	pool-1-thread-3
4	pool-1-thread-4
5	pool-1-thread-5
```

然而事与愿违，实际的输出是这样的
```
1	pool-1-thread-1
2	pool-1-thread-1
3	pool-1-thread-1
4	pool-1-thread-1
5	pool-1-thread-1
```

嗯嗯？为什么没有并发执行subscribe里的代码呢？我以为是我自己的代码有问题，又陆续尝试了内置的一些Scheduler，consumer均是在同一个线程中执行的，好吧，看来是我理解错rxjava的Schedulers了，这货的from方法接收一个Executor参数，并不是指接下来的任务会提交给这个线程池并发的执行。


这大概也是RxJava和CompletableFuture的区别之一吧。搜索了一圈，用rxjava实现并发主要以以下几个方法

1. 在flatMap中使用obseveOn
```java
Flowable.just(1, 2, 3, 4, 5)
        .flatMap(i -> Flowable.just(i).observeOn(Schedulers.from(exec))
                .doOnNext(d -> {
                    System.out.println(d + "\t" + Thread.currentThread().getName());
                    Thread.sleep(1000L);
                }))
        .subscribe();
```

2. 在flatMap中使用Future
```java
Flowable.just(1, 2, 3, 4, 5)
        .flatMap(i -> Flowable.fromFuture(CompletableFuture.completedFuture(i), Schedulers.from(exec))
                .doOnNext(d -> {
                    System.out.println(d + "\t" + Thread.currentThread().getName());
                    Thread.sleep(1000L);
                })
        )
        .subscribe();
```
3. 使用ParallelFlowable
```java
Flowable.just(1, 2, 3, 4, 5)
        .parallel()
        .runOn(Schedulers.from(exec))
        .doOnNext(d -> {
            System.out.println(d + "\t" + Thread.currentThread().getName());
            Thread.sleep(1000L);
        })
        .sequential()
        .subscribe();
```

有两个要注意的地方
1. RxJava在执行并发的时候，并不会使用Executor的maximumPollSize这个属性，corePollSize有多大，那么最大就有多少个线程
2. parallel()有一个重载方法可以传入并发数，默认为cpu核心数，在单核的服务器上这个数字是1，也就是不管Executor有多少个线程，只会用一个线程去执行任务

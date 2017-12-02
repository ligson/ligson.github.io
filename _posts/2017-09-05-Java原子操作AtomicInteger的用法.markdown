---
layout: post
title:  "Java原子操作AtomicInteger的用法"
date:   2017-09-05 14:10:51 +0800
categories: Java
tags: 并发
description: JDK1.5之后的java.util.concurrent.atomic包里，多了一批原子处理类。AtomicBoolean、AtomicInteger、AtomicLong、AtomicReference。主要用于在高并发环境下的高效程序处理,来帮助我们简化同步处理.
---

### 前言:

> JDK1.5之后的``java.util.concurrent.atomic``包里，多了一批原子处理类。``AtomicBoolean``、``AtomicInteger``、``AtomicLong``、``AtomicReference``。主要用于在高并发环境下的高效程序处理,来帮助我们简化同步处理.

### AtomicInteger:

&emsp;&emsp;``AtomicInteger``，一个提供原子操作的Integer的类。在Java语言中，++i和i++操作并不是线程安全的，在使用的时候，不可避免的会用到synchronized关键字。而AtomicInteger则通过一种线程安全的加减操作接口。

我们先来看看AtomicInteger给我们提供了什么接口:

>``public final int get()`` //获取当前的值

>``public final int getAndSet(int newValue)``//获取当前的值，并设置新的值

>``public final int getAndIncrement()``//获取当前的值，并自增

>``public final int getAndDecrement()`` //获取当前的值，并自减

>``public final int getAndAdd(int delta)`` //获取当前的值，并加上预期的值

下面通过两个简单的例子来看一下 AtomicInteger 的优势在哪:

普通线程同步:
```java
class Test2 {
        private volatile int count = 0;

        public synchronized void increment() {
                  count++; //若要线程安全执行执行count++，需要加锁
        }

        public int getCount() {
                  return count;
        }
}
```
使用AtomicInteger:

```java
class Test2 {
        private AtomicInteger count = new AtomicInteger();

        public void increment() {
                  count.incrementAndGet();
        }
   //使用AtomicInteger之后，不需要加锁，也可以实现线程安全。
       public int getCount() {
                return count.get();
        }
}
```

从上面的例子中我们可以看出：使用AtomicInteger是非常的安全的.而且因为AtomicInteger由硬件提供原子操作指令实现的。在非激烈竞争的情况下，开销更小，速度更快。

我们来看看AtomicInteger是如何使用非阻塞算法来实现并发控制的:
AtomicInteger的关键域只有一下3个：

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();

private static final long valueOffset;
static {    
         try {        
                valueOffset = unsafe.objectFieldOffset (AtomicInteger.class.getDeclaredField("value"));   
        } catch (Exception ex) {
               throw new Error(ex);
        }
    }

private volatile int value;
```
&emsp;&emsp;这里， unsafe是java提供的获得对对象内存地址访问的类，注释已经清楚的写出了，它的作用就是在更新操作时提供“比较并替换”的作用。实际上就是AtomicInteger中的一个工具。
valueOffset是用来记录value本身在内存的便宜地址的，这个记录，也主要是为了在更新操作在内存中找到value的位置，方便比较。

&emsp;&emsp;注意：value是用来存储整数的时间变量，这里被声明为volatile，就是为了保证在更新操作时，当前线程可以拿到value最新的值（并发环境下，value可能已经被其他线程更新了）。
这里，我们以自增的代码为例，可以看到这个并发控制的核心算法：

```java
/**
*Atomicallyincrementsbyonethecurrentvalue.
*
*@returntheupdatedvalue
*/
publicfinalintincrementAndGet(){
  for(;;){
    //这里可以拿到value的最新值
    intcurrent=get();
    intnext=current+1;
    if(compareAndSet(current,next))
      returnn ext;
  }
}

publicfinalbooleancompareAndSet(intexpect,intupdate){
//使用unsafe的native方法，实现高效的硬件级别CAS
        return unsafe.compareAndSwapInt(this,valueOffset,expect,update);
}
```

优点总结:

> 最大的好处就是可以避免多线程的优先级倒置和死锁情况的发生，提升在高并发处理下的性能。

---
title: synchronized
abbrlink: edea11bc
date: 2021-12-23 10:47:33
tags:
- Java
- 并发
categories:
- [Java, 并发]
---
## 常见错误  

&emsp;&emsp;最近比较深入的研究了一下synchronized,发现网上百分之90的博客都是错的,下边我首先来总结一下一些常见的错误:

- <font color="red">无锁->偏向->轻量->重量</font>  

    认为锁对象的状态转换是这么个流程的同学请刷个1,这里边存在两个问题
    1. **并不存在无锁到偏向**这么一个过程,偏向锁只能从可偏向状态或者重偏向状态获得.  
    偏向锁模式存在偏向锁延迟机制：

        {% codeblock lang:java %}
        HotSpot 虚拟机在启动后有个 4s 的延迟才会对每个新建的对象开启偏向锁模式。JVM启动时会进行一系列的复杂活动，比如装载配置，系统类初始化等等。在这个过程中会使用大量
        synchronized关键字对对象加锁，且这些锁大多数都不是偏向锁。为了减少初始化时间，JVM默认延时加载偏向锁。
        {% endcodeblock %}

        在过了这个延迟偏向时间后创建出来的新对象都是可偏向状态,并不是无锁状态(除非触发了批量撤销),重偏向是触发了批量重偏向后的逻辑,只有两种获得偏向锁的途径,并不存在无锁->偏向锁  
    2. 偏向->轻量  
        这个也不是一定的,轻量锁会出现在线程**交替执行**的时候.  
        假如一把锁先偏向了线程A,此时A的同步代码块里的内容已执行完,并且A已释放锁(释放偏向锁并不会做什么),但是线程A还存活(如果不存活可能触发jvm层面的线程复用,直接想线程B与Jvm的线程绑定),现在线程B尝试去获取锁,此时线程B就会触发偏向锁撤销,然后将偏向锁升级为一把轻量锁.  

        很多同学认为升级重量锁必须要激烈并发,其实不然,比如上边那个场景,如果线程B去获取锁的时候,线程A还没有释放锁,线程B会尝试获取轻量锁,但是轻量锁获取失败,线程B会直接开始膨胀,创建Monitor对象,升级为重量锁,所以升级重量锁和竞争激烈与否并没有关系,**只要有竞争,就会升级成重量锁**;

    总结下获取锁的过程:首先看锁对象是否是可偏向,或者是否是可重偏向状态,如果是直接获取偏向锁,如果不是尝试获取轻量锁,轻量锁获取失败,直接开始膨胀为重量锁;
- <font color="red">轻量锁自旋</font>  
    轻量锁和偏向锁都不存在自旋,只是会尝试获取,CAS获取失败直接走下一步;

## 阅读源码与实验的一些发现  

- 批量重偏向修改epoch修改的是正在被锁定的对象**和可偏向对象的epoch**,网上很多博客说的都是修改正在被锁定对象的epoch
- 批量重偏向和批量撤销共用一个计数器 都会在25秒(默认值))后归零  

{% codeblock lang:C++ %}
    static HeuristicsResult update_heuristics(oop o, bool allow_rebias) {
        markOop mark = o->mark();
        //如果不是偏向模式直接返回
        if (!mark->has_bias_pattern()) {
            return HR_NOT_BIASED;
        }
        // 锁对象的类
        Klass* k = o->klass();
        // 当前时间
        jlong cur_time = os::javaTimeMillis();
        // 该类上一次批量撤销的时间
        jlong last_bulk_revocation_time = k->last_biased_lock_bulk_revocation_time();
        // 该类偏向锁撤销的次数
        int revocation_count = k->biased_lock_revocation_count();
        // BiasedLockingBulkRebiasThreshold是重偏向阈值（默认20），BiasedLockingBulkRevokeThreshold是批量撤销阈值（默认40），BiasedLockingDecayTime是开启一次新的批量重偏向距离上次批量重偏向的后的延迟时间，默认25000。也就是开启批量重偏向后，经过了一段较长的时间（>=BiasedLockingDecayTime），撤销计数器才超过阈值，那我们会重置计数器。
        if ((revocation_count >= BiasedLockingBulkRebiasThreshold) &&
            (revocation_count <  BiasedLockingBulkRevokeThreshold) &&
            (last_bulk_revocation_time != 0) &&
            (cur_time - last_bulk_revocation_time >= BiasedLockingDecayTime)) {
            // This is the first revocation we've seen in a while of an
            // object of this type since the last time we performed a bulk
            // rebiasing operation. The application is allocating objects in
            // bulk which are biased toward a thread and then handing them
            // off to another thread. We can cope with this allocation
            // pattern via the bulk rebiasing mechanism so we reset the
            // klass's revocation count rather than allow it to increase
            // monotonically. If we see the need to perform another bulk
            // rebias operation later, we will, and if subsequently we see
            // many more revocation operations in a short period of time we
            // will completely disable biasing for this type.
            k->set_biased_lock_revocation_count(0);
            revocation_count = 0;
        }

        // 自增撤销计数器
        if (revocation_count <= BiasedLockingBulkRevokeThreshold) {
            revocation_count = k->atomic_incr_biased_lock_revocation_count();
        }
        // 如果达到批量撤销阈值则返回HR_BULK_REVOKE
        if (revocation_count == BiasedLockingBulkRevokeThreshold) {
            return HR_BULK_REVOKE;
        }
        // 如果达到批量重偏向阈值则返回HR_BULK_REBIAS
        if (revocation_count == BiasedLockingBulkRebiasThreshold) {
            return HR_BULK_REBIAS;
        }
        // 没有达到阈值则撤销单个对象的锁
        return HR_SINGLE_REVOKE;
    }
{% endcodeblock %}

### 验证

{% codeblock lang:java %}
    @Slf4j
    public class Test {

        public static void main(String[] args) throws InterruptedException {
            //延时产生可偏向对象
            Thread.sleep(5000);

            // 创建一个list，来存放锁对象
            List<Test> locks = new ArrayList<>();

            // 线程1
            new Thread(() -> {
                for (int i = 0; i < 30; i++) {
                    // 新建锁对象
                    Test lock = new Test();
                    synchronized (lock) {
                        locks.add(lock);
                    }
                }
                try {
                    //为了防止JVM线程复用，在创建完对象后，保持线程thead1状态为存活
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "thead1").start();

            //睡眠3s钟保证线程thead1创建对象完成
            Thread.sleep(1000);
            // 线程2
            new Thread(() -> {
                for (int i = 0; i < 30; i++) {
                    Test obj = locks.get(i);
                    synchronized (obj) {
                        if(i>=15&&i<=21||i>=38){
                            log.debug(Thread.currentThread().getName()+"-第" + (i + 1) + "次加锁执行中\t"+
                                            ClassLayout.parseInstance(obj).toPrintable());
                        }
                    }
                    if(i==17||i==19){
                        log.debug(Thread.currentThread().getName()+"-第" + (i + 1) + "次释放锁\t"+
                                        ClassLayout.parseInstance(obj).toPrintable());
                    }
                }
                try {
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "thead2").start();
        /*如果下边的20-30个对象任然可以重偏向,则说明批量重偏向和批量撤销共用一个计数器 都会在25秒(默认值)后归零,
          而且最后new的新对象任然是可偏向状态101,如果注释掉的话这行代码则创建的新对象是无锁状态(批量撤销,关闭该Class的偏向锁功能),
          而且下边的20-30个对象会变成轻量锁*/
            Thread.sleep(26000);

            // 创建一个list，来存放锁对象
            List<Test> locks1 = new ArrayList<>();

            // 线程1
            new Thread(() -> {
                for (int i = 0; i < 30; i++) {
                    // 新建锁对象
                    Test lock = new Test();
                    synchronized (lock) {
                        locks1.add(lock);
                    }
                }
                try {
                    //为了防止JVM线程复用，在创建完对象后，保持线程thead1状态为存活
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "thead3").start();

            //睡眠3s钟保证线程thead1创建对象完成
            Thread.sleep(1000);
            // 线程2
            new Thread(() -> {
                for (int i = 0; i < 30; i++) {
                    Test obj = locks1.get(i);
                    synchronized (obj) {
                        if(i>=15&&i<=21||i>=38){
                            log.debug(Thread.currentThread().getName()+"-第" + (i + 1) + "次加锁执行中\t"+
                                            ClassLayout.parseInstance(obj).toPrintable());
                        }
                    }
                    if(i==17||i==19){
                        log.debug(Thread.currentThread().getName()+"-第" + (i + 1) + "次释放锁\t"+
                                        ClassLayout.parseInstance(obj).toPrintable());
                    }
                }
                try {
                    Thread.sleep(100000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }, "thead4").start();
            Thread.sleep(3000);
            Test test = new Test();
            log.debug("新对象:" + (ClassLayout.parseInstance(test).toPrintable()));
            LockSupport.park();
        }
    }
{% endcodeblock %}

其他介绍请参考[github](https://github.com/farmerjohngit/myblog/issues/12),这是一篇非常好的博客,写的很详细,我也就不重复总结了.  

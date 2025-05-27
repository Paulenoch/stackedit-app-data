# 1. 进程和线程
进程是程序的一次执行过程，是系统运行程序的基本单位，因此进程是动态的。系统运行一个程序即是一个进程从创建，运行到消亡的过程。

在 Java 中，当我们启动 main 函数时其实就是启动了一个 JVM 的进程，而 main 函数所在的线程就是这个进程中的一个线程，也称主线程。

线程是一个比进程更小的执行单位。一个进程在其执行的过程中可以产生多个线程

多个线程共享进程的**堆**和**方法区**资源，但每个线程有自己的**程序计数器**、**虚拟机栈**和**本地方法栈**


# 2. 线程的生命周期和状态
-   NEW: 初始状态，线程被创建出来但没有被调用 `start()` 。
-   RUNNABLE: 运行状态，线程被调用了 `start()`等待运行的状态。
-   BLOCKED：阻塞状态，需要等待锁释放。
-   WAITING：等待状态，表示该线程需要等待其他线程做出一些特定动作（通知或中断）。
-   TIME_WAITING：超时等待状态，可以在指定的时间后自行返回而不是像 WAITING 那样一直等待。
-   TERMINATED：终止状态，表示该线程已经运行完毕。

# 为什么要使用多线程
线程可以比作是轻量级的进程，是程序执行的最小单位，线程间的切换和调度的成本远远小于进程。另外，多核 CPU 时代意味着多个线程可以同时运行，这减少了线程上下文切换的开销。

# 单核CPU上运行多线程效率一定高吗
对于CPU密集型线程，效率低下，多个线程同时运行会导致频繁的线程切换，增加了系统的开销
对于IO密集型线程，效率高，多个线程同时运行可以利用 CPU 在等待 IO 时的空闲时间

# 3. 什么是线程死锁，如何避免
```java
public class DeadLockDemo {
    private static Object resource1 = new Object();//资源 1
    private static Object resource2 = new Object();//资源 2

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (resource1) {
                System.out.println(Thread.currentThread() + "get resource1");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource2");
                synchronized (resource2) {
                    System.out.println(Thread.currentThread() + "get resource2");
                }
            }
        }, "线程 1").start();

        new Thread(() -> {
            synchronized (resource2) {
                System.out.println(Thread.currentThread() + "get resource2");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resource1");
                synchronized (resource1) {
                    System.out.println(Thread.currentThread() + "get resource1");
                }
            }
        }, "线程 2").start();
    }
}
```

# 4. 乐观锁和悲观锁

悲观锁：**共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程**
乐观锁：只是在提交修改的时候去验证对应的资源（也就是数据）是否被其它线程修改了（具体方法可以使用版本号机制或 CAS 算法）。

# 5. 如何实现乐观锁
- 版本号机制：一般是在数据表中加上一个数据版本号 `version` 字段，表示数据被修改的次数。当数据被修改时，`version` 值会加一。当线程 A 要更新数据值时，在读取数据的同时也会读取 `version` 值，在提交更新时，若刚才读取到的 version 值为当前数据库中的 `version` 值相等时才更新，否则重试更新操作，直到更新成功。

- CAS（Compare and Swap)
- CAS 涉及到三个操作数：

-   **V**：要更新的变量值(Var)
-   **E**：预期值(Expected)
-   **N**：拟写入的新值(New)
当且仅当 V 的值等于 E 时，CAS 通过原子方式用新值 N 来更新 V 的值。如果不等，说明已经有其它线程更新了 V，则当前线程放弃更新。

无法避免ABA问题

# 6. 并发编程的三个重要特性
- 原子性：
- 可见性：当一个线程对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值。在 Java 中，可以借助`synchronized`、`volatile` 以及各种 `Lock` 实现可见性。如果我们将变量声明为 `volatile` ，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都到主存中进行读取。
- 有序性：`volatile` 关键字可以禁止指令进行重排序优化

# 7. 为什么要有JMM（Java内存模型）
抽象了线程和主内存的关系；规定了从java源代码到CPU可执行指令这个转化过程要遵守的规则

![输入图片说明](/imgs/2025-03-25/dYPtrQed7KL0wNF8.png)

# 8. synchronized关键字
可以保证被它修饰的方法或者代码块在任意时刻只能有一个线程执行

### 8.1 底层原理
`synchronized` 同步语句块的实现使用的是 `monitorenter` 和 `monitorexit` 指令，其中 `monitorenter` 指令指向同步代码块的开始位置，`monitorexit` 指令则指明同步代码块的结束位置。

`synchronized` 修饰的方法并没有 `monitorenter` 指令和 `monitorexit` 指令，取而代之的是 `ACC_SYNCHRONIZED` 标识，该标识指明了该方法是一个同步方法。
**不过，两者的本质都是对对象监视器 monitor 的获取。**

### 8.2 与volatile的关系
-   `volatile` 关键字是线程同步的轻量级实现，所以 `volatile`性能肯定比`synchronized`关键字要好 。但是 `volatile` 关键字只能用于变量而 `synchronized` 关键字可以修饰方法以及代码块 。
-   `volatile` 关键字能保证数据的可见性，但不能保证数据的原子性。`synchronized` 关键字两者都能保证。
-   `volatile`关键字主要用于解决变量在多个线程之间的可见性，而 `synchronized` 关键字解决的是多个线程之间访问资源的同步性。

### 8.3 ReentrantLock

`ReentrantLock` 实现了 `Lock` 接口，是一个可重入且独占式的锁，和 `synchronized` 关键字类似。不过，`ReentrantLock` 更灵活、更强大，增加了轮询、超时、中断、公平锁和非公平锁等高级功能
-   **公平锁** : 锁被释放之后，先申请的线程先得到锁。性能较差一些，因为公平锁为了保证时间上的绝对顺序，上下文切换更频繁。
-   **非公平锁**：锁被释放之后，后申请的线程可能会先获取到锁，是随机或者按照其他优先级排序的。性能更好，但可能会导致某些线程永远无法获取到锁。

## 8.4 synchronized和ReentrantLock有什么区别和联系
联系：
- 二者都是可重入锁，指的是线程可以再次获取自己的内部锁
- synchronized依赖于JVM（对象监视器monitor），ReentrantLock在API层面实现
- 
ReentrantLock新功能：
- 等待可中断：当前线程在等待获取锁的过程中，如果其他线程中断当前线程「 `interrupt()` 」，当前线程就会抛出 `InterruptedException` 异常，可以捕捉该异常进行相应处理。
- 公平锁
- 支持超时：`ReentrantLock` 提供了 `tryLock(timeout)` 的方法，可以指定等待获取锁的最长等待时间，如果超过了等待时间，就会获取锁失败，不会一直等待。


# 9. volatile关键字
在 Java 中，`volatile` 关键字可以保证变量的可见性，如果我们将变量声明为 **`volatile`** ，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都到主存中进行读取。
不能保证数据的原子性

# AQS
`AbstractQueuedSynchronizer` ，翻译过来的意思就是抽象队列同步器
AQS 就是一个抽象类，主要用来构建锁和同步器。
简单来说，AQS 是一个抽象类，为同步器提供了通用的 **执行框架**。它定义了 **资源获取和释放的通用流程**，而具体的资源获取逻辑则由具体同步器通过重写模板方法来实现。 因此，可以将 AQS 看作是同步器的 **基础“底座”**，而同步器则是基于 AQS 实现的 **具体“应用”**。

# AQS为什么使用CLH锁队列的变体
CLH锁是一种自旋锁的优化实现
自旋锁通过线程不断对一个原子变量执行 `compareAndSet`（简称 `CAS`）操作来尝试获取锁。在高并发场景下，多个线程会同时竞争同一个原子变量，容易造成某个线程的 `CAS` 操作长时间失败，从而导致 **“饥饿”问题**（某些线程可能永远无法获取锁）。

CLH 锁通过引入一个队列来组织并发竞争的线程，对自旋锁进行了改进：

-   每个线程会作为一个节点加入到队列中，并通过自旋监控前一个线程节点的状态，而不是直接竞争共享变量。
-   线程按顺序排队，确保公平性，从而避免了 “饥饿” 问题。

AQS（AbstractQueuedSynchronizer）在 CLH 锁的基础上进一步优化，形成了其内部的 **CLH 队列变体**。主要改进点有以下两方面：

1.  **自旋 + 阻塞**： CLH 锁使用纯自旋方式等待锁的释放，但大量的自旋操作会占用过多的 CPU 资源。AQS 引入了 **自旋 + 阻塞** 的混合机制：
    -   如果线程获取锁失败，会先短暂自旋尝试获取锁；
    -   如果仍然失败，则线程会进入阻塞状态，等待被唤醒，从而减少 CPU 的浪费。
2.  **单向队列改为双向队列**：CLH 锁使用单向队列，节点只知道前驱节点的状态，而当某个节点释放锁时，需要通过队列唤醒后续节点。AQS 将队列改为 **双向队列**，新增了 `next` 指针，使得节点不仅知道前驱节点，也可以直接唤醒后继节点，从而简化了队列操作，提高了唤醒效率。

# AQS中节点的不同状态

AQS 中的 `waitStatus` 状态类似于 **状态机** ，通过不同状态来表明 Node 节点的不同含义，并且根据不同操作，来控制状态之间的流转。

-   状态 `0` ：新节点加入队列之后，初始状态为 `0` 。
    
-   状态 `SIGNAL` ：当有新的节点加入队列，此时新节点的前继节点状态就会由 `0` 更新为 `SIGNAL` ，表示前继节点释放锁之后，需要对新节点进行唤醒操作。如果唤醒 `SIGNAL` 状态节点的后续节点，就会将 `SIGNAL` 状态更新为 `0` 。即通过清除 `SIGNAL` 状态，表示已经执行了唤醒操作。
    
-   状态 `CANCELLED` ：如果一个节点在队列中等待获取锁锁时，因为某种原因失败了，该节点的状态就会变为 `CANCELLED` ，表明取消获取锁，这种状态的节点是异常的，无法被唤醒，也无法唤醒后继节点。
    ![输入图片说明](/imgs/2025-04-26/vCYTTT3W8qZzVDQX.png)

# AQS常用同步工具类

### Semaphore
```java
// 初始共享资源数量
final Semaphore semaphore = new Semaphore(5);
// 获取1个许可
semaphore.acquire();
// 释放1个许可
semaphore.release();
```

### CountDownLatch
`CountDownLatch` 允许 `count` 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。
当线程调用 `countDown()` 时，其实使用了`tryReleaseShared`方法以 CAS 的操作来减少 `state`，直至 `state` 为 0 。当 `state` 为 0 时，表示所有的线程都调用了 `countDown` 方法，那么在 `CountDownLatch` 上等待的线程就会被唤醒并继续执行。


# 10. ThreadLocal有什么用
![输入图片说明](/imgs/2025-03-25/dGCdkDkakSuK3a1X.png)

# 11. ThreadLocal原理
每一个线程都有一个ThreadLocalMap

# ThreadLocal内存泄漏问题
-   **key 是弱引用**：`ThreadLocalMap` 中的 key 是 `ThreadLocal` 的弱引用 (`WeakReference<ThreadLocal<?>>`)。 这意味着，如果 `ThreadLocal` 实例不再被任何强引用指向，垃圾回收器会在下次 GC 时回收该实例，导致 `ThreadLocalMap` 中对应的 key 变为 `null`。
-   **value 是强引用**：即使 `key` 被 GC 回收，`value` 仍然被 `ThreadLocalMap.Entry` 强引用存在，无法被 GC 回收。

在使用完 `ThreadLocal` 后，务必调用 `remove()` 方法。 这是最安全和最推荐的做法。 `remove()` 方法会从 `ThreadLocalMap` 中显式地移除对应的 entry，彻底解决内存泄漏的风险。 即使将 `ThreadLocal` 定义为 `static final`，也强烈建议在每次使用后调用 `remove()`。

# 如何跨线程传递ThreadLocal的值
- InheritableThreadLocal
-   JDK 自带方案，继承自`ThreadLocal`。
    
-   在父线程创建子线程时，会自动把父线程中的`InheritableThreadLocal`值传递给子线程。
    
-   **局限性**：对于线程池的场景不适用（线程复用，子线程只继承一次值，后续不会更新）。

- TransmittableThreadLocal
`TransmittableThreadLocal`（简称TTL）是阿里巴巴开源的一个库，专门用于跨线程池传递`ThreadLocal`的上下文信息。

### 特点：

-   支持线程池场景，解决了`InheritableThreadLocal`的不足。
    
-   性能和扩展性良好，广泛用于生产环境。

# 12. 为什么要用线程池
- 降低反复创建和销毁线程带来的消耗
- 提高响应速度
- 提高线程的可管理性

# 13. 为什么不推荐使用内置线程池
线程池参数不明确，如任务队列长度，线程最大数量等参数可能为无限大，带来隐患

# 14. 线程池的参数
-   **`corePoolSize`（核心线程数）**
    
    -   线程池中始终保留的线程数量（即使处于空闲状态，也不会被回收，除非 `allowCoreThreadTimeOut(true)` 被设置）。
        
    -   超过这个数目的任务会放入队列，除非当前线程数还没达到 `maximumPoolSize`。
        
-   **`maximumPoolSize`（最大线程数）**
    
    -   线程池能创建的最大线程数。
        
    -   当队列满了，而且线程数小于这个值时，会创建新线程来处理任务。
        
-   **`keepAliveTime`（线程存活时间）**
    
    -   非核心线程在空闲状态下保持多长时间后会被回收。
        
    -   如果 `allowCoreThreadTimeOut(true)`，那么核心线程也会按照这个规则回收。
        
-   **`unit`（时间单位）**
    
    -   与 `keepAliveTime` 配合使用，指定时间单位（如：`TimeUnit.SECONDS`）。
        
-   **`workQueue`（任务队列）**
    
    -   用于保存等待执行的任务的阻塞队列。
        
    -   常用的实现：
        
        -   `ArrayBlockingQueue`：有界队列。
            
        -   `LinkedBlockingQueue`：无界队列（实际有上限，但很大）。
            
        -   `SynchronousQueue`：不存储任务，提交的任务必须直接交给线程执行。
            
-   **`threadFactory`（线程工厂）**
    
    -   用于创建新线程。可以自定义线程名、是否为守护线程等。
        
    -   通常使用 `Executors.defaultThreadFactory()`。
        
-   **`handler`（拒绝策略）**
    
    -   当线程池和队列都满了，无法接收新任务时，使用的拒绝策略。常见策略包括：
        
        -   `AbortPolicy`：抛出异常（默认）。
            
        -   `CallerRunsPolicy`：由调用者线程来执行该任务。
            
        -   `DiscardPolicy`：直接丢弃任务，不抛异常。
            
        -   `DiscardOldestPolicy`：丢弃队列中最老的任务，然后尝试执行新任务。

# 核心线程会被回收吗
默认不会，这是为了减少创建线程的开销，因为核心线程通常是要长期保持活跃的。
可以考虑将 `allowCoreThreadTimeOut(boolean value)` 方法的参数设置为 `true`，这样就会回收空闲（时间间隔由 `keepAliveTime` 指定）的核心线程了。

# 线程池的拒绝策略
-   `ThreadPoolExecutor.AbortPolicy`：抛出 `RejectedExecutionException`来拒绝新任务的处理。
-   `ThreadPoolExecutor.CallerRunsPolicy`：调用执行者自己的线程运行任务，也就是直接在调用`execute`方法的线程中运行(`run`)被拒绝的任务，如果执行程序已关闭，则会丢弃该任务。因此这种策略会降低对于新任务提交速度，影响程序的整体性能。如果你的应用程序可以承受此延迟并且你要求任何一个任务请求都要被执行的话，你可以选择这个策略。
-   `ThreadPoolExecutor.DiscardPolicy`：不处理新任务，直接丢弃掉。
-   `ThreadPoolExecutor.DiscardOldestPolicy`：此策略将丢弃最早的未处理的任务请求。

# 线程池中线程异常后，销毁还是复用
简单来说：使用`execute()`时，未捕获异常导致线程终止，线程池创建新线程替代；使用`submit()`时，异常被封装在`Future`中，线程继续复用。

这种设计允许`submit()`提供更灵活的错误处理机制，因为它允许调用者决定如何处理异常，而`execute()`则适用于那些不需要关注执行结果的场景。

# 如何设定线程池的大小
-   **CPU 密集型任务(N+1)：** 这种任务消耗的主要是 CPU 资源，可以将线程数设置为 N（CPU 核心数）+1。比 CPU 核心数多出来的一个线程是为了防止线程偶发的缺页中断，或者其它原因导致的任务暂停而带来的影响。一旦任务暂停，CPU 就会处于空闲状态，而在这种情况下多出来的一个线程就可以充分利用 CPU 的空闲时间。
-   **I/O 密集型任务(2N)：** 这种任务应用起来，系统会用大部分的时间来处理 I/O 交互，而线程在处理 I/O 的时间段内不会占用 CPU 来处理，这时就可以将 CPU 交出给其它线程使用。因此在 I/O 密集型任务的应用中，我们可以多配置一些线程，具体的计算方法是 2N。


# 15. 线程池处理任务的流程
提交任务——核心池是否已满——等待队列是否已满——最大线程池是否已满——依据拒绝策略处理
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc1Mzc4MjA3LDE5NjgwMTc0MDQsOTAxOD
gwMDI1LDIzODk1MjkzNywtMTE2NzQ2MTE4NiwyMTAxMzc0MzMs
NTgxNTExOTM4LC0xNDEyNzE1Mzg4LDExNTQyODc1MTQsNzg0Mz
E3MzY1LC0xNTY2MzE2NjQ4XX0=
-->
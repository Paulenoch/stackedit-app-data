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
- 可见性：当一个线程对共享变量进行了修改，那么另外的线程都是立即可以看到修改后的最新值。
在 Java 中，可以借助`synchronized`、`volatile` 以及各种 `Lock` 实现可见性。
如果我们将变量声明为 `volatile` ，这就指示 JVM，这个变量是共享且不稳定的，每次使用它都到主存中进行读取。


<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA1ODcwNTQ0MV19
-->
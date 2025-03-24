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
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTQ2NzM1NzAyM119
-->
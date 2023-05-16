---
description: https://developer.android.google.cn/reference/kotlin/java/lang/Thread?hl=en
---

# 线程：Thread

**Thread定义**

> `Thread`是程序中的执行线程。Java虚拟机允许应用程序同时运行多个执行线程。每个线程都有优先级。优先级较高的线程优先于优先级较低的线程执行。每个线程也可以标记为守护进程，也可以不标记为守护进程。当在某个线程中运行的代码创建一个新线程对象时，新线程的优先级最初设置为等于创建线程的优先级，并且仅当创建线程是守护进程时才是守护进程线程。

**Thread的几种状态**

`新建状态（new）`：实例化之后进入该状态；\
`就绪状态（Runnable）`：线程调用start()之后就绪状态等待cpu执行，注意这时只是表示可以运行并不代表已经运行；\
`运行状态（Running）`：线程获得cpu的执行，开始执行run()方法的代码；\
`阻塞状态（Blocked）`：线程由于各种原因进入阻塞状态：join()、sleep()、wait()、等待触发条件、等待由别的线程占用的锁；\
`死亡状态（Dead）`：线程运行完毕或异常退出，可使用isAlive()获取状态。

**Thread主要方法**

本文主要涉及`run()`、`interrupt()`、`yield()`、`join()`，创建`Thread`的两种方法,使用已有线程类以及自定义线程

```java
//直接新建启动线程
new Thread(new Runnable() {
            @Override
            public void run() {
            }
        }).start();
//启动自定义线程
new DefineThread().start();

class DefineThread extends Thread {
        @Override
        public void run() {
            super.run();
        }
    }
```

### `1.Thread.interrupt()`

打断此线程，`前提条件`：如果此线程在调用对象类或该类的join（）、join（long）、join（long）、join（long）、sleep（long）、sleep（long）或sleep（long，int）方法的object wait（long）、object wait（long）或object wait（long，int）方法时被阻塞，则其中断状态将被清除，并且它将收到interruptedException。`这里需要注意`：当线程在阻塞状态被interrupt打断时仅仅会抛出`InterruptedException`，想要结束该线程还是需要我们自己来处理

1.让线程处于非中断状态，调用`mDefineThread.interrupt()`是打断不了

```java
class DefineThread extends Thread {
        @Override
        public void run() {
            super.run();
            while (true) {
                Log.e(TAG, "running~~");
            }
        }
    }
```

> 2019-10-25 14:11:28.996 20448-20482/com.code.testcontentprovider E/MainActivity: running\~\~\
> 2019-10-25 14:11:29.062 20448-20482/com.code.testcontentprovider E/MainActivity: running\~\~

2.让线程`sleep`一定时间，确保调用`mDefineThread.interrupt()`处于中断状态

```java
class DefineThread extends Thread {
        @Override
        public void run() {
            super.run();
            while (true) {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Log.e(TAG, "InterruptedException~~");
                     //跳出当前线程
                    break;
                }
                Log.e(TAG, "running~~");
            }
        }
    }
```

> 2019-10-25 14:27:51.656 21781-21807/com.code.testcontentprovider E/MainActivity: InterruptedException\~\~

我们收到了`InterruptedException`并自己跳出了当前线程，对以上方法进行改进，增加一层状态校验

```java
class DefineThread extends Thread {
        @Override
        public void run() {
            super.run();
            //增加状态校验
            while (!isInterrupted()) {
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Log.e(TAG, "InterruptedException~~");
                    //跳出当前线程
                    break;
                }
                Log.e(TAG, "running~~");
            }
        }
    }
```

### `2.Thread.yield()`

向调度程序提示当前线程愿意放弃当前对处理器的使用。调度程序可以忽略此提示。yield是一个启发式的尝试，旨在改进线程之间的相对进程，否则会过度使用cpu。它的使用应该与详细的概要分析和基准测试相结合，以确保它实际具有预期的效果。使用这种方法是不合适的。它可能对调试或测试有用，因为它可能有助于由于竞争条件而重现错误。在设计并发控制结构（如java.util.concurrent.locks包中的结构）时，它可能也很有用。 这里的解释是引用了官网的原话，我可以理解成`yield`并没什么实际用处么？

```java
Thread thread = new Print0Thread("李白");
        Thread thread1 = new Print0Thread("杜甫");
        thread.setPriority(5);
        thread1.setPriority(1);
        thread.start();
        thread1.start();

class Print0Thread extends Thread {
        private String name;
        public Print0Thread(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            super.run();
            while (i < 10) {
                Log.e(TAG, name + ":" + i);
                i++;
                if (i == 2) {
                    Thread.yield();
                    Log.e(TAG, name + " yield:" + i);
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
```

> 2019-10-25 15:20:40.069 29972-29995/? E/MainActivity: 李白:0\
> 2019-10-25 15:20:40.070 29972-29995/? E/MainActivity: 李白:1\
> 2019-10-25 15:20:40.070 29972-29995/? E/MainActivity: 李白 yield:2\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:2\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:3\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:4\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:5\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:6\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:7\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:8\
> 2019-10-25 15:20:40.070 29972-29996/? E/MainActivity: 杜甫:9

经过多次测试，`yield`只是让出当前的CPU，一旦让出之后后面就获取不到CPU资源了...

### `3.Thread.join()`

等待此线程死亡，假如有两条线程`Print0Thread`、`Print1Thread`，后者在执行的过程中调用了前者的`join()`，那么从此刻开始，后续的工作需要等到前者线程完全结束才能继续进行

```java
Print0Thread mPrint0Thread = new Print0Thread("李白");
    Print1Thread mPrint1Thread = new Print1Thread("杜甫");
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mPrint0Thread.start();
        mPrint1Thread.start();
    }


    class Print0Thread extends Thread {
        private int i = 0;
        private String name;
        public Print0Thread(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            super.run();
            while (i++ < 5) {
                Log.e(TAG, name + ":" + i);
            }
        }
    }

    class Print1Thread extends Thread {
        private int i = 0;
        private String name;
        public Print1Thread(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            super.run();
            try {
                mPrint0Thread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            while (i++ < 5) {
                Log.e(TAG, name + ":" + i);
            }
        }
    }
```

> 2019-10-25 15:41:21.460 31160-31186/com.code.testcontentprovider E/MainActivity: 李白:1\
> 2019-10-25 15:41:21.460 31160-31186/com.code.testcontentprovider E/MainActivity: 李白:2\
> 2019-10-25 15:41:21.460 31160-31186/com.code.testcontentprovider E/MainActivity: 李白:3\
> 2019-10-25 15:41:21.460 31160-31186/com.code.testcontentprovider E/MainActivity: 李白:4\
> 2019-10-25 15:41:21.460 31160-31186/com.code.testcontentprovider E/MainActivity: 李白:5\
> 2019-10-25 15:41:21.461 31160-31187/com.code.testcontentprovider E/MainActivity: 杜甫:1\
> 2019-10-25 15:41:21.461 31160-31187/com.code.testcontentprovider E/MainActivity: 杜甫:2\
> 2019-10-25 15:41:21.461 31160-31187/com.code.testcontentprovider E/MainActivity: 杜甫:3\
> 2019-10-25 15:41:21.462 31160-31187/com.code.testcontentprovider E/MainActivity: 杜甫:4\
> 2019-10-25 15:41:21.462 31160-31187/com.code.testcontentprovider E/MainActivity: 杜甫:5

看一下`join()`的源码，如果调用线程一直活着，那么会一直`阻塞`当前线程知道调用线程完成工作

```
 public final void join(long millis) throws InterruptedException {
        synchronized(lock) {
        if (millis == 0) {
            //如果线程还活着，一直阻塞
            while (isAlive()) {
                lock.wait(0);
            }
        } 
        ...
    }
```

### `4.Thread.suspend()`

此方法在API级别15中被弃用。此方法旨在挂起线程，但它本身容易死锁。如果目标线程在监视器上持有一个锁，以在挂起关键系统资源时保护该资源，则在恢复目标线程之前，任何线程都无法访问该资源。如果要恢复目标线程的线程在调用resume之前试图锁定此监视器，则会导致死锁

### `5.Thread.resume()`

同样在API级别15中被弃用，不做过多介绍

### `6.Thread.stop()`

同样在API级别15中被弃用，不做过多介绍

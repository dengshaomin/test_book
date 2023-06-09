# Exception

在code的过程中，引发程序crash的原因大体分为两类`Error`、`Exception`，他们都继承`Throwable`\


![](<../.gitbook/assets/image (54).png>)

**Error**

无法处理的错误，表示程序碰到较为严重的问题。大多数错误与上层的应用程序无直接关系。而是底层系统或是JVM（Java虚拟机）出现的问题。例如Java虚拟机运行错误（VirtualMachineError），当JVM内存空间不足时，将出现OutOfMemoryError。当Error出现后，上层程序是无法控制的，所以，出现这类错误时，本质上也是不应该试图去处理它所发生的异常状况。

**Exception**

Exception主要分为两大类`Unchecked Exception（Runtime Exception）`以及`Checked Exception（非Runtime Exception）`\
\
Unchecked 与Checked 区别在于对于CheckedException，必须添加try…catch…捕获异常、或者throw 抛出异常并处理

```java
//Checked Exception手动抛出异常
private void openFile() throws FileNotFoundException {
        File file = new File("");
        FileInputStream fileInputStream = new FileInputStream(file);
    }

//Checked Exception通过try...catch捕获
private void openFile() {
        File file = new File("");
        try {
            FileInputStream fileInputStream = new FileInputStream(file);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
            //处理异常
        }
    }
```

UncheckedException，非强制性处理，在多线程环境下建议处理该异常，不然容易出现crash

```java
//选择性异常处理，不强制
        try {
            List<String> list = new ArrayList<>();
            list.get(1);
        } catch (IndexOutOfBoundsException e) {
            //处理异常
        }

public E get(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));

        return (E) elementData[index];
    }
```

以上对异常的种类大致做了个归纳，接下来有几个问题需要弄清楚：

1. 异常是由谁触发的？
2. 异常触发为什么app会退出？
3. 异常触发之后android studio的log是谁输出的？

在想了解以上3个问题之前先了解一下：`UncaughtExceptionHandler`

### UncaughtExceptionHandler

当线程由于未捕获的异常而终止时，java虚拟机将在线程中查询UncaughtExceptionHandler的使用，并调用这个方法，传递线程和异常作为参数；如果当前线程没有设置UncaughtExceptionHandler，将调用ThreadGroup对象的UncaughtExceptionHandler；`-- from source code notes`

也就是说所有未捕获的异常都会回调到Thread或ThreadGroup的UncaughtExceptionHandler中；如果我们想自定义一个异常捕捉框架一般的做法如下：t为抛出异常的线程，e为具体抛出的Throwable

```kotlin
Thread.setDefaultUncaughtExceptionHandler { t, e ->
}
```

这是个静态方法全局仅有一个，接受注入参数的UncaughtExceptionHandler也应该是个静态变量

```java
#Thread.java
private static volatile UncaughtExceptionHandler defaultUncaughtExceptionHandler;
public static void setDefaultUncaughtExceptionHandler(UncaughtExceptionHandler eh) {
         defaultUncaughtExceptionHandler = eh;
     }
```

 如果当我们没有主动去给Thread设置UncaughtExceptionHandler时异常又是怎么被捕捉到的呢？其实在进程被fork出来时framwork层就已经设置了好了UncaughtExceptionHandler。代码的调用在`RuntimeInit.java`中

```java
#RunTimeInit.java
@UnsupportedAppUsage
public static final void main(String[] argv) {
       ...
        commonInit();
        ...
    }
    

@UnsupportedAppUsage
protected static final void commonInit() {
    ...
    LoggingHandler loggingHandler = new LoggingHandler();
    RuntimeHooks.setUncaughtExceptionPreHandler(loggingHandler);
    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler(loggingHandler));
    ...
}
```

 main为RuntimeInit的入口函数，当Zygote进程启动后, 便会执行到frameworks/base/cmds/app\_process/App\_main.cpp文件的main()方法。

```java
int main(int argc, char* const argv[])
{
   ...
    if (zygote) {
        // 启动AppRuntime，见小节[3.2]
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    }
}
```

用如下代码抛出一个简单的异常来回答上面提出的3个问题

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val s: String? = null
        s!!.length
    }
}
```

### 1.dispatchUncaughtException由jvm层调用

Thread内部的UncaughtExceptionHandler除了内部方法dispatchUncaughtException并没有找到其他调用入口，在这个方法打上断点然后出发一个异常，从java层调用堆栈来看也找不到java层的调用入口，说明这个dispatchUncaughtException应该是由jvm层调用的。

```java
public final void dispatchUncaughtException(Throwable e) {
    // BEGIN Android-added: uncaughtExceptionPreHandler for use by platform.
    Thread.UncaughtExceptionHandler initialUeh =
            Thread.getUncaughtExceptionPreHandler();
    if (initialUeh != null) {
        try {
            //先调用preHandler
            initialUeh.uncaughtException(this, e);
        } catch (RuntimeException | Error ignored) {
            // Throwables thrown by the initial handler are ignored
        }
    }
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

### 2.KillApplicationHandler杀死app

在调用commonInit方法时一共设置了两个UncaughtExceptionHandler：uncaughtExceptionPreHandler和defaultUncaughtExceptionHandler，preHander是一个loggingHandler并将这个handler注入到KillApplicationHandler，当APP异常时uncaughtException被jvm层回调：输入日志->通知 asm->杀死进程

```java
private static class KillApplicationHandler implements Thread.UncaughtExceptionHandler {
    private final LoggingHandler mLoggingHandler;
    public KillApplicationHandler(LoggingHandler loggingHandler) {
        this.mLoggingHandler = Objects.requireNonNull(loggingHandler);
    }

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        try {
            //先输出log
            ensureLogging(t, e);
            if (mCrashing) return;
            mCrashing = true;
            if (ActivityThread.currentActivityThread() != null) {
                ActivityThread.currentActivityThread().stopProfiling();
            }

            // 通知asm程序异常
            ActivityManager.getService().handleApplicationCrash(
                    mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
        } catch (Throwable t2) {
            ...
        } finally {
            // 杀死进程
            Process.killProcess(Process.myPid());
            System.exit(10);
        }
    }
     private void ensureLogging(Thread t, Throwable e) {
            if (!mLoggingHandler.mTriggered) {
                try {
                    //回调loggingHandler
                    mLoggingHandler.uncaughtException(t, e);
                } catch (Throwable loggingThrowable) {
                    // Ignored.
                }
            }
        }
```

### 3.LoggingHandler输出日志

在KillApplicationHandler中的uncaughException被触发时先调用了loggerHander进行日志的输出记录

```java

private static class LoggingHandler implements Thread.UncaughtExceptionHandler {
    public volatile boolean mTriggered = false;

    @Override
    public void uncaughtException(Thread t, Throwable e) {
        mTriggered = true;
        //判断是否crash
        if (mCrashing) return;
        //判断是否系统crash
        if (mApplicationObject == null && (Process.SYSTEM_UID == Process.myUid())) {
            Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
        } else {
            //输出app crash堆栈
            StringBuilder message = new StringBuilder();
            message.append("FATAL EXCEPTION: ").append(t.getName()).append("\n");
            final String processName = ActivityThread.currentProcessName();
            if (processName != null) {
                message.append("Process: ").append(processName).append(", ");
            }
            message.append("PID: ").append(Process.myPid());
            Clog_e(TAG, message.toString(), e);
        }
    }
}
```

到这里第三个问题就明白了，日志是由LoggingHandler输出的，Android Studio的log面板如下：

```java
2020-09-11 15:08:27.051 30608-30608/com.code.acrademo E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.code.acrademo, PID: 30608
    java.lang.RuntimeException: Unable to start activity ComponentInfo{com.code.acrademo/com.code.acrademo.MainActivity}: kotlin.KotlinNullPointerException
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3270)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3409)
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83)
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016)
        at android.os.Handler.dispatchMessage(Handler.java:107)
        at android.os.Looper.loop(Looper.java:214)
        at android.app.ActivityThread.main(ActivityThread.java:7356)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930)
     Caused by: kotlin.KotlinNullPointerException
        at com.code.acrademo.MainActivity.onCreate(MainActivity.kt:12)
        at android.app.Activity.performCreate(Activity.java:7802)
        at android.app.Activity.performCreate(Activity.java:7791)
        at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1299)
        at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3245)
        at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:3409) 
        at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:83) 
        at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135) 
        at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95) 
        at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2016) 
        at android.os.Handler.dispatchMessage(Handler.java:107) 
        at android.os.Looper.loop(Looper.java:214) 
        at android.app.ActivityThread.main(ActivityThread.java:7356) 
        at java.lang.reflect.Method.invoke(Native Method) 
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:492) 
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:930) 
```

 

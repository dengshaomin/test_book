# Android性能优化

本章的主要内容包括

* 布局优化
* 绘制优化
* 内存优化
* 响应速度优化
* ListView优化
* Bitmap优化
* 线程优化等等

### 1 布局优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#1) <a href="#1" id="1"></a>

1. **布局优化的思想就是尽量减少布局文件的层次。**\
   首先删除布局中无用的控件和层级。\
   其次有选择性的使用性能较低的`ViewGroup`，比如`RelativeLayout`。如果布局中既可以使用`LinearLayout`也可以使用`RelativeLayout`,那么就采用`LinearLayout`。这是因为`RelativeLayout`的功能比较复杂，它的布局过程需要话费更多的CPU时间。`FrameLayout`和`LinearLayout`一样都是一种简单高效的`ViewGroup`，因此可以考虑使用它们。但是很多时候无法单纯的通过一个`LinearLayout`或者`FrameLayout`无法实现产品效果，需要通过嵌套的方式来实现。这种情况还是建议采用`RelativeLayout`，因为`ViewGroup`的嵌套相当于增加了布局的层级，同样会降低程序的性能。\
   **可以考虑将**`RelativeLayout`**替换成**`ConstraintLayout`**，更加强大的布局方式，也足够扁平。**
2.  **布局优化的外一种手段是采用**`include`、`merge`**标签和**`ViewStub`。\
    `include`标签主要用于布局重用，`merge`一般和`include`配合使用，它可以降低减少布局的层级，而`ViewStub`则提供了按需加载的功能，当需要时才会将`ViewStub`中的布局加载到内存，这样提高了程序的初始化效率。\
    `merge`标签一般和`include`标签一起使用从而减少布局的层级。比如，在一个竖直的线性布局中，如果被包含的布局文件中也采用竖直的`LinearLayout`，那么被包含的布局文件中的`LinearLayout`是多余的，通过`merge`标签就可以去掉多余的一层`LinearLayout`。\
    `ViewStub`继承至`View`，它非常轻量级且宽高都为0，因此它本身不参与任何的布局和绘制过程。`ViewStub`的意义在于按需加载所需的布局文件。比如网络异常时的界面，这个时候没有必要在整个界面初始化的时候将其加载进来，通过`ViewStub`就可以在使用的时候在加载，提高了程序初始化时的性能。

    > 我们可以覆盖任何被include的布局的根布局的layout属性(`android:layout_*`)，当然我们必须指定`android:layout_width`和`android:layout_height`属性，这样其他覆盖的属性才会生效。

    下面是`ViewStub`的定义：\


    ```
    <ViewStub
        android:id="@+id/stub_import"
        android:inflatedId="@+id/panel_import"
        android:layout="@layout/progress_overlay"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom" />
    ```

    其中`stub_import`是`ViewStub`的`id`，而`panel_import`就是`progress_overlay`这个布局文件根元素的`id`。\
    在加载`ViewStub`中的布局时，可以按照以下方式进行：\


    ```
    findViewById(R.id.stub_import)).setVisibility(View.VISIBLE);
    // or
    View importPanel = ((ViewStub) findViewById(R.id.stub_import)).inflate();
    ```

    > **Note**: `inflate`方法会返回被填充的`View`，因此我们需要与此layout进行交互时，不需要在调用`findViewById`方法。

    一旦`ViewStub`调用上面的方法后，`ViewStub`会被替换掉，此时`ViewStub`不再是整个布局结构的一部分了。此外，`ViewStub`还不 支持`merge`标签。

### 2 绘制优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#2) <a href="#2" id="2"></a>

绘制优化的分为两部分，一部分是避免过度绘制，另一部分指`View#onDraw`方法要避免执行大量的操作。

**`View#onDraw`方法优化：**

1.  `onDraw`方法不要创建新的局部对象\
    Android Studio的lint有一项会检查此项：

    > Avoid object allocations during draw/layout operations (preallocate and reuse instead) less... (⌘F1)\
    > Inspection info:You should avoid allocating objects during a drawing or layout operation. These are called frequently, so a smooth UI can be interrupted by garbage collection pauses caused by the object allocations. The way this is generally handled is to allocate the needed objects up front and to reuse them for each drawing operation. Some methods allocate memory on your behalf (such as Bitmap.create), and these should be handled in the same way.\
    > Issue id: DrawAllocation
2. `onDraw`方法不要做耗时的任务，也不能执行成千上万次的循环操作。

`View`的绘制帧率保证60fps最佳，这要求每帧的绘制时间不超过16ms(1000/60)。虽然程序很难保证16ms这个时间，但是尽量降低`onDraw`方法的复杂度总是切实有效的。

**避免过度绘制**

官方提供了一个修复过度绘制的一些要点：[Fix overdraw](https://developer.android.com/topic/performance/rendering/overdraw.html#fixing)，其中的内容有：

1. 减少布局中不必要的背景
2. 扁平view层级（这就是上面一节说到的布局优化了）
3. 减少透明度的使用\
   在屏幕上渲染透明像素（称为alpha渲染）是过度绘制的关键因素。与标准过度绘制不同，系统通过在现有绘制像素上方绘制多绘制一层不透明像素来完全隐藏现有绘制像素，透明对象需要首先绘制现有像素，以便可以出现正确的混合效果。透明动画、淡出和阴影等视觉效果都涉及某种透明度，因此对于过度绘制有着显著的影响。\
   您可以通过减少渲染的透明对象的数量来改善这些情况下的过度绘制。例如，您可以通过在TextView中绘制黑色文本并在其上设置半透明的alpha值来获取灰色文本。但是，通过简单地以灰色绘制文本，您可以获得相同的效果和更好的性能。

开发者模式中的一些设置可以在我们进行绘制优化时提供很大的帮助：[Inspect GPU rendering speed and overdraw](https://developer.android.com/studio/profile/inspect-gpu-rendering)，现以下面两节呈现。

#### 2.1 Profile GPU Rendering[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#21-profile-gpu-rendering) <a href="#21-profile-gpu-rendering" id="21-profile-gpu-rendering"></a>

Android 6.0（API Level 23）设备上Profile GPU Rendering图表示例如下：

![Profile GPU Rendering 图表](https://blog.yorek.xyz/assets/images/android/profile\_gpu\_rendering.png)

以下是有关输出的几点注意事项：

* 对于每个可见应用，此工具将显示一个图表。
* 沿水平轴的每个竖条都代表一个帧，每个竖条的高度表示渲染该帧所花的时间（单位：毫秒）。
* 水平绿线表示 16 毫秒。 要实现每秒 60 帧，代表每个帧的竖条需要保持在此线以下。 当竖条超出此线时，可能会使动画出现暂停。
* 此工具通过加宽对应的竖条并降低透明度来突出显示超出 16 毫秒阈值的帧。
* 每个竖条都有与渲染管道中某个阶段对应的彩色区段。 区段数因设备的 API 级别而异。

下表介绍了使用运行 Android 6.0 及更高版本的设备时分析器输出中某个竖条的每个区段。Android 6.0 及更高版本中的竖条区段

| 竖条区段 | 渲染阶段         | 说明                                                                                            |
| ---- | ------------ | --------------------------------------------------------------------------------------------- |
|      | 交换缓冲区        | 表示 CPU 等待 GPU 完成其工作的时间。 如果此竖条升高，则表示应用在 GPU 上执行太多工作。                                           |
|      | 命令问题         | 表示 Android 的 2D 渲染器向 OpenGL 发起绘制和重新绘制显示列表的命令所花的时间。 此竖条的高度与它执行每个显示列表所花的时间的总和成正比—显示列表越多，红色条就越高。 |
|      | 同步和上传        | 表示将位图信息上传到 GPU 所花的时间。 大区段表示应用花费大量的时间加载大量图形。                                                   |
|      | 绘制           | 表示用于创建和更新视图显示列表的时间。 如果竖条的此部分很高，则表明这里可能有许多自定义视图绘制，或 onDraw 函数执行的工作很多。                          |
|      | 测量/布局        | 表示在视图层次结构中的 onLayout 和 onMeasure 回调上所花的时间。 大区段表示此视图层次结构正在花很长时间进行处理。                           |
|      | 动画           | 表示评估运行该帧的所有动画程序所花的时间。 如果此区段很大，则表示您的应用可能在使用性能欠佳的自定义动画程序，或因更新属性而导致一些意料之外的工作。                    |
|      | 输入处理         | 表示应用执行输入 Event 回调中的代码所花的时间。 如果此区段很大，则表示此应用花太多时间处理用户输入。 考虑将此处理任务分流到另一个线程。                      |
|      | 其他时间/VSync延迟 | 表示应用执行两个连续帧之间的操作所花的时间。 它可能表示界面线程中进行的处理太多，而这些处理任务本可以分流到其他线程。                                   |

4.0（API 级别 14）和 5.0（API 级别 21）之间的 Android 版本具有蓝色、紫色、红色和橙色区段。 低于 4.0 的 Android 版本只有蓝色、红色和橙色区段。 下表显示的是 Android 4.0 和 5.0 中的竖条区段。Android 4.0 及 5.0 中的竖条区段

| 竖条区段 | 渲染阶段 | 说明                                                                                            |
| ---- | ---- | --------------------------------------------------------------------------------------------- |
|      | 进程   | 表示 CPU 等待 GPU 完成其工作的时间。 如果此竖条升高，则表示应用在 GPU 上执行太多工作。                                           |
|      | 执行   | 表示 Android 的 2D 渲染器向 OpenGL 发起绘制和重新绘制显示列表的命令所花的时间。 此竖条的高度与它执行每个显示列表所花的时间的总和成正比—显示列表越多，红色条就越高。 |
|      | XFer | 表示将位图信息上传到 GPU 所花的时间。 大区段表示应用花费大量的时间加载大量图形。 此区段在运行 Android 4.0 或更低版本的设备上不可见。                  |
|      | 更新   | 表示用于创建和更新视图显示列表的时间。 如果竖条的此部分很高，则表明这里可能有许多自定义视图绘制，或 onDraw 函数执行的工作很多。                          |

**注：** 尽管此工具名为 Profile GPU Rendering，但所有受监控的进程实际上发生在 CPU 中。 通过将命令提交到 GPU 触发渲染，GPU 异步渲染屏幕。 在某些情况下，GPU 会有太多工作要处理，在它可以提交新命令前，您的 CPU 必须等待。 在等待时，您将看到橙色条和红色条中出现峰值，且命令提交将被阻止，直到 GPU 命令队列腾出更多空间。

#### 2.2 Debug GPU Overdraw[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#22-debug-gpu-overdraw) <a href="#22-debug-gpu-overdraw" id="22-debug-gpu-overdraw"></a>

当应用在同一帧中多次绘制相同像素时，便会发生过度绘制。

Android 按如下方法为界面元素设置颜色，以便确定过度绘制的次数：

* **True color**：没有过度绘制
* ![](https://blog.yorek.xyz/assets/images/android/overdraw-blue.png)**蓝色**：过度绘制1次
* ![](https://blog.yorek.xyz/assets/images/android/overdraw-green.png)**绿色**：过度绘制2次
* ![](https://blog.yorek.xyz/assets/images/android/overdraw-pink.png)**粉色**：过度绘制3次
* ![](https://blog.yorek.xyz/assets/images/android/overdraw-red.png)**红色**：过度绘制4次及以上

![某个应用正常时的样子（左侧），以及它在 GPU 过度绘制后的样子（右侧）](https://blog.yorek.xyz/assets/images/android/gpu-overdraw-before.png)某个应用正常时的样子（左侧），以及它在 GPU 过度绘制后的样子（右侧）

![大量过度绘制的应用（左侧）以及很少过度绘制的应用（右侧）的示例](https://blog.yorek.xyz/assets/images/android/gpu-overdraw-after.png)大量过度绘制的应用（左侧）以及很少过度绘制的应用（右侧）的示例

请记住，有些过度绘制是不可避免的。在优化您的应用的界面时，应尝试达到大部分显示true color或仅有 1 次过度绘制（蓝色）的视觉效果。

### 3 内存优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#3) <a href="#3" id="3"></a>

内存优化一方面避免发生内存泄漏([JVM基础知识](https://blog.yorek.xyz/jvm/jvm-content/))，一方面注意内存的管理([Manage your app's memory](https://developer.android.com/topic/performance/memory))。

#### 3.1 常见内存泄漏[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#31) <a href="#31" id="31"></a>

**造成内存泄漏的根本原因是生命周期较短的某个对象被生命周期更长的对象所持有，导致该对象不能及时释放。**

> **LeakCanary 1.5.1 检测内存泄漏原理**：\
> 在Activity destroy后将Activity的弱引用关联到ReferenceQueue中，这样Activity将要被GC前，会出现在ReferenceQueue中。\
> 随后，会向主线程的MessageQueue添加一个`IdleHandler`，用于在idle时触发一个发生在HandlerThread的等待5秒后开始检测内存泄漏的代码。\
> 这段代码首先会判断是否对象已经被回收，如果有，则没有内存泄漏，结束；否则，手动调用`Runtime.getRuntime().gc()`进行GC，等待100ms后再次判断是否已经被GC，若还没有被回收，那么说明有内存泄漏，开始dump hprof。\
> 关于LeakCanary的源码分析，可以参考[LeakCanary2源码解析](https://blog.yorek.xyz/android/3rd-library/leakcanary/)

**3.1.1 静态变量导致的内存泄漏¶**

因为静态变量生命周期等于应用程序的生命周期，所以静态变量引用的变量不会被回收掉。这里涉及到[GC Roots](https://blog.yorek.xyz/jvm/java-gc/#322)的概念。

下面是两种明显的内存泄漏：

```
public class MyCouponActivity extends BaseActivity  {

    private static Context sContext;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my_coupon);
        sContext = this;
    }
}

// or

public class MyCouponActivity extends BaseActivity  {

    private static View sView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_my_coupon);
        sView = new View(this);
    }
}
```

这种内存泄漏，Android Studio会有提示：\
![memory\_leak\_static](https://blog.yorek.xyz/assets/images/android/memory\_leak\_static.png)

**3.1.2 非静态内部类（匿名类）内存泄露¶**

注意一下静态匿名内部类和非静态匿名内部类的区别，一句话总结：**非静态匿名内部类会持有外部class的强引用**。这就是Handler需要使用static修饰，且持有Activity时需要持有`WeakReference`的缘故。_看Handler写的业务代码的时候很烦，往往msg还是诸如0、1这样的魔法值，还要在Handler实现的位置与发送的位置之间互相转换，实在令人头疼。吹爆RxJava，只从学了RxJava，写复杂一点的逻辑真是越来越轻松了，可读性还好_静态匿名内部类和非静态匿名内部类的区别

|                    | static inner class | non static inner class |
| ------------------ | ------------------ | ---------------------- |
| 与外部class引用关系       | 没有引用关系             | 自动获得强引用                |
| 被调用时需要外部实例         | 不需要                | 需要                     |
| 能否调用外部class中的变量和方法 | 不能                 | 能                      |
| 生命周期               | 自主的生命周期            | 依赖于外部累，甚至比外部类更长        |

**3.1.2.1 HANDLER内存泄漏¶**

使用非静态内部类来实现`Handler`，lint就会给出警告。这涉及到`Handler`的原理。

> 如果`Handler`中有延迟的任务或者是等待执行的任务队列过长，都有可能因为`Handler`继续执行而导致`Activity`发生泄漏。\
> 1\. 首先，非静态的`Handler`类会默认持有外部类的引用，包含`Activity`等。\
> 2\. 然后，还未处理完的消息（`Message`）中会持有`Handler`的引用。\
> 3\. 还未处理完的消息会处于消息队列中，即消息队列`MessageQueue`会持有`Message`的引用。\
> 4\. 消息队列`MessageQueue`位于`Looper`中，`Looper`的生命周期跟应用一致。

因此，此时的引用关系链是`Looper` -> `MessageQueue` -> `Message` -> `Handler` -> `Activity`。所以，这时退出`Activity`的话，由于存在上述的引用关系，垃圾回收器将无法回收`Activity`，从而造成内存泄漏。

**3.1.2.2 多线程引起的内存泄露¶**

我们一般使用匿名类等来启动一个线程，如下：

```
new Thread(new Runnable() {
    @Override
    public void run() {

    }
}).start();
```

同样，匿名`Thread`类里持有了外部类的引用。当`Activity`退出时，`Thread`有可能还在后台执行，这时就会发生了内存泄露。

解决方案和上面Handler类似：要不就是变成静态内部类，引用外面资源时使用`WeakReference`；要不就是在`Activity`退出时，结束线程。

**3.1.3 其他情况造成的内存泄漏¶**

1. 集合类内存泄露\
   集合类添加元素后，将会持有元素对象的引用，导致该元素对象不能被垃圾回收，从而发生内存泄漏。
2. 属性动画导致的内存泄漏\
   属性动画中有一类无限循环的动画，如果`Activity`中播放此类动画且没有在`onDestory`方法中去停止动画，那么动画会一直播放下去。我们需要在`Activity#onDestory`中调用`animator.cancel()`方法来停止动画。
3. 网络、文件等流忘记关闭
4. 手动注册广播时，退出时忘记`unregisterReceiver()`
5. Service执行完后忘记`stopSelf()`
6. EventBus等观察者模式的框架忘记手动解除注册

#### 3.2 内存管理[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#32) <a href="#32" id="32"></a>

内存管理可以看[Manage your app's memory](https://developer.android.com/topic/performance/memory)

实现`ComponentCallbacks2`接口，根据`onTrimMemory`中的level做出不同的响应。

```
import android.content.ComponentCallbacks2
// Other import statements ...

class MainActivity : AppCompatActivity(), ComponentCallbacks2 {

    // Other activity code ...

    /**
     * Release memory when the UI becomes hidden or when system resources become low.
     * @param level the memory-related event that was raised.
     */
    override fun onTrimMemory(level: Int) {

        // Determine which lifecycle or system event was raised.
        when (level) {

            ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
                /*
                   Release any UI objects that currently hold memory.

                   The user interface has moved to the background.
                */
            }

            ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE,
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW,
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
                /*
                   Release any memory that your app doesn't need to run.

                   The device is running low on memory while the app is running.
                   The event raised indicates the severity of the memory-related event.
                   If the event is TRIM_MEMORY_RUNNING_CRITICAL, then the system will
                   begin killing background processes.
                */
            }

            ComponentCallbacks2.TRIM_MEMORY_BACKGROUND,
            ComponentCallbacks2.TRIM_MEMORY_MODERATE,
            ComponentCallbacks2.TRIM_MEMORY_COMPLETE -> {
                /*
                   Release as much memory as the process can.

                   The app is on the LRU list and the system is running low on memory.
                   The event raised indicates where the app sits within the LRU list.
                   If the event is TRIM_MEMORY_COMPLETE, the process will be one of
                   the first to be terminated.
                */
            }

            else -> {
                /*
                  Release any non-critical data structures.

                  The app received an unrecognized memory level value
                  from the system. Treat this as a generic low-memory message.
                */
            }
        }
    }
}
```

### 4 响应速度优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#4) <a href="#4" id="4"></a>

响应速度优化的核心思想是 **避免在主线程中做耗时操作** ，常见的就是IO操作以及计算量大的操作等。

另外，优化App启动时间也是一种学问，详见[App startup time](https://developer.android.com/topic/performance/vitals/launch-time)。\
App启动可以分为三种情况：冷启动(cold start)、温启动(warm start)、热启动(hot start)。

#### 4.1 冷启动[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#41) <a href="#41" id="41"></a>

冷启动是指应用程序从头开始：系统的进程在此开始之前没有创建应用程序的进程。冷启动发生在设备启动后首次启动的应用程序或应用程序被系统终止后。这种类型的启动在最小化启动时间方面提出了最大的挑战，因为系统和应用程序比其他启动状态有更多的工作要做。

在冷启动开始时，系统有三个任务。这些任务是：

1. 加载并启动应用程序
2. 启动后立即为应用程序显示一个空白启动窗口
3. 创建应用程序进程

一旦系统创建了应用程序进程，应用程序进程就会负责下一个阶段：

1. 创建应用程序对象
2. 启动主线程
3. 创建主Activity
4. 加载View
5. 在屏幕上进行布局
6. 执行初始的绘制

应用程序进程完成第一次绘制后，系统进程会将当前显示的背景窗口替换为主Activity。此时，用户可以开始使用该应用程序。

下图展示了系统和应用程序之间是如何协作的。

![应用程序冷启动的重要部分的直观展示](https://blog.yorek.xyz/assets/images/android/cold-launch.png)

性能问题可能出现在创建Application和创建Activity期间。

**Application创建**

当Application启动时，空白的启动窗口将保留在屏幕上，直到系统首次完成绘制应用程序。此时，系统进程会交换应用程序的启动窗口，允许用户开始与应用程序进行交互。

如果我们在应用中重载了`Application.onCreate()`方法，系统会调用我们的Application对象的`onCreate()`方法。之后，应用程序会spawns（_为什么会是这个词，感觉与Zygote有关_）出主线程（也称为UI线程），并通过创建主Activity来执行后续任务。

从现在开始，系统、App级别的进程就会按照[应用生命周期阶段](https://developer.android.com/guide/components/activities/process-lifecycle)来执行。

**Activity创建**

应用程序进程创建Activity后，Activity将执行以下操作：

1. 初始化值
2. 调用构造器
3. 调用诸如`Activity.onCreate()`这类回调方法，根据Activity当前的生命周期状态

通常，`onCreate()`方法对加载时间的影响最大，因为它以最高的开销执行这些任务：加载和inflate视图、初始化Activity运行所需的对象。

#### 4.2 热启动[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#42) <a href="#42" id="42"></a>

应用程序的热启动比冷启动更简单，开销更低。在热启动中，系统所做的全部事情就是将您的Activity带到前台。如果您的所有应用程序的Activity仍然驻留在内存中，那么应用程序可以避免重复对象初始化，布局加载和渲染。

但是，如果为了响应内存修整事件（例如`onTrimMemory()`）而清除了某些内存，则需要重新创建这些对象来响应热启动事件。

热启动与冷启动一样显示相同的屏幕行为：系统进程将会显示一个空白屏幕，直到应用程序完成了Activity的渲染。

#### 4.3 温启动[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#43) <a href="#43" id="43"></a>

温启动包括一些冷启动期间发生的操作的子集；同时，它比冷启动表示更少的开销。有许多潜在的状态可以被视为温启动。例如：

1. 用户退出您的应用，但随后重新启动它。该进程可能会继续运行，但应用程序必须通过调用`onCreate()`从头开始重新创建Activity。
2. 系统将您的应用程序从内存中逐出，然后用户重新启动它。进程和Activity需要被重新启动，但是任务可以从传递给`onCreate()`的saved instance state bundle中获益。

#### 4.4 启动时间优化标准及方式[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#44) <a href="#44" id="44"></a>

以下情况，Android vitals认为app启动耗时过多（也就是慢）：

* 冷启动需要5秒或更长时间。
* 温启动需要2秒或更长时间。
* 热启动需要1.5秒或更长时间。

**4.4.1 启动时间的诊断¶**

**初始显示的时间**

在Android 4.4（API级别19）及更高版本中，logcat包含一个包含名为`Displayed`的值的输出行。该值表示启动进程和完成在屏幕上绘制相应Activity之间所经过的时间量。经过的时间包括以下事件序列：

1. 启动进程
2. 初始化对象
3. 创建、初始化Activity
4. 填充布局
5. 首次绘制应用程序

报告的日志行类似于下面的示例：

```
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms
```

logcat输出中的`Displayed`度量标准不一定包括所有资源都加载并显示完成的总时间：它不包括布局文件中未引用的资源或应用程序在对象初始化过程中创建的资源。它排除了这些资源，因为加载它们是一个内联过程，并不会block应用程序的初始显示。

有时，logcat输出中的`Displayed`行包含总时间的附加字段。例如：

```
ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms (total +1m22s643ms)
```

在这种情况下，第一个测量时间仅适用于首次绘制的Activity。`total`时间测量值从应用程序进程开始时开始，可能包括另一个首先启动但未向屏幕显示任何内容的Activity。仅在单个Activity与总启动时间之间存在差异时才显示`total`时间测量值。

您还可以使用[ADB Shell Activity Manager](https://developer.android.com/studio/command-line/shell.html#am)命令运行应用程序来测量初始显示的时间。这是一个例子：

```
adb [-d|-e|-s <serialNumber>] shell am start -S -W com.example.app/.MainActivity -c android.intent.category.LAUNCHER -a android.intent.action.MAIN
```

`Displayed`的度量标准与以前一样出现在logcat输出中。您的终端窗口还应显示以下内容：

```
Starting: Intent Activity: com.example.app/.MainActivity ThisTime: 2044 TotalTime: 2044 WaitTime: 2054 Complete
```

* ThisTime: 一连串启动 Activity 的最后一个 Activity 的启动耗时
* TotalTime: 新应用启动的耗时，包括新进程的启动和 Activity 的启动，但不包括前一个应用 Activity pause 的耗时。\
  也就是说，开发者一般只要关心 TotalTime 即可，这个时间才是自己应用真正启动的耗时。
* WaitTime: 总的耗时，包括前一个应用 Activity pause 的时间和新应用启动的时间

**完全显示的时间**

您可以使用`Activity.reportFullyDrawn()`方法来测量应用程序启动到完全显示所有资源和视图层次结构之间所用的时间。在应用程序执行延迟加载的情况下，这可能很有用。在延迟加载中，应用程序不会block窗口的初始绘制，而是异步加载资源并更新视图层次结构。

如果由于延迟加载，应用程序的初始显示不包含所有资源，您可以将所有资源和View完成加载并显示视为单独的度量标准：例如，您的UI可能已完全加载，并绘制了一些文本，但尚未显示应用必须从网络中获取的图像。

要解决此问题，您可以手动调用`reportFullyDrawn()`，让系统知道您的Activity已完成其延迟加载。使用此方法时，logcat显示的值是从创建应用程序对象到调用`reportFullyDrawn()`的时间。这是logcat输出的一个例子：

```
system_process I/ActivityManager: Fully drawn {package}/.MainActivity: +1s54ms
```

logcat输出有时包括`total`时间，如初始显示的时间中所述。

如果您了解到显示时间比您想要的慢，您可以继续尝试识别启动过程中的瓶颈。

**识别瓶颈**

寻找瓶颈的好方法是使用Android Studio CPU Profiler。有关信息，请参阅使用[Inspect CPU activity with CPU Profiler](https://developer.android.com/studio/profile/cpu-profiler)。

您还可以通过内置跟踪应用程序和活动的`onCreate()`方法来深入了解潜在的瓶颈。要了解内置跟踪工具，请参阅[Trace](https://developer.android.com/reference/android/os/Trace.html)功能的文档以及[Systrace](https://developer.android.com/studio/profile/systrace-commandline.html)工具。

**4.4.2 启动时间的优化¶**

本节讨论通常会影响应用程序启动性能的几个问题。这些问题主要涉及初始化应用程序和Activity对象，以及loading屏幕。

**1. 过重的App初始化**

当override `Application`对象，并且在初始化该对象时执行繁重的工作或复杂的逻辑时，启动性能会受到影响。如果Application子类执行不需要执行的初始化，则您的应用程序可能会在启动期间浪费时间。某些初始化可能完全没有必要：例如，初始化主Activity的状态信息，当应用实际启动以响应intent时。根据intent，应用程序仅使用先前初始化的状态数据的子集。

应用程序初始化期间有影响或数量众多的其他挑战包括垃圾收集事件，或者与初始化同时发生的磁盘I/O等，进一步阻止初始化过程。垃圾收集尤其是Dalvik运行时垃圾收集，是一个考虑因素; Art运行时并行执行垃圾收集，可以最大限度地减少操作的影响。

**问题诊断**

您可以使用method tracing或inline tracing来尝试诊断问题。

**method tracing**

运行CPU Profiler会发现`callApplicationOnCreate()`方法最终会调用`com.example.customApplication.onCreate`方法。如果该工具显示这些方法需要很长时间才能完成执行，那么您应该进一步探索来确定是哪个正在进行的工作引起的。

**inline tracing**

使用inline tracing来调查可能的罪魁祸首，包括：

* 您应用的初始`onCreate()`函数。
* 您的应用初始化的任何全局单例对象。
* 在瓶颈期间可能发生的任何磁盘I/O，反序列化或tight loops。

**问题的解决方案**

无论问题是否在于不必要的初始化还是磁盘I/O，解决方案都是调用懒初始化对象：仅初始化那些立即需要的对象。例如，不是创建全局静态对象，而是移动到单例模式，这样应用程序仅在第一次访问对象时初始化对象。此外，考虑使用像[Dagger](https://dagger.dev/)这样的依赖注入框架，它们会在第一次注入时创建对象和依赖项。

**2. 过重的Activity初始化**

创建Activity通常需要大量高额开销。通常，有机会优化这项工作以实现性能改进。这些常见问题包括：

* 填充大型或复杂的布局
* 阻碍屏幕渲染的磁盘、网络IO事件
* 加载、解码Bitmap
* 栅格化`VectorDrawable`对象
* 初始化Activity的其他子系统。

**问题诊断**

在这种情况下，method tracing和inline tracing都可以证明是有用的。

**method tracing**

使用CPU Profiler时，请注意应用程序的`Application`子类构造函数和`com.example.customApplication.onCreate()`方法。

如果该工具显示这些方法需要很长时间才能完成执行，那么您应该进一步探索来确定是哪个正在进行的工作引起的。

**inline tracing**

使用inline tracing来调查可能的罪魁祸首，包括：

* 您应用的初始`onCreate()`函数。
* 您的应用初始化的任何全局单例对象。
* 在瓶颈期间可能发生的任何磁盘I/O，反序列化或tight loops。

**问题的解决方案**

存在许多潜在的瓶颈，但有两个常见问题和补救措施如下：

* View层次结构越大，应用程序对其进行inflate的时间就越长。您可以采取的两个步骤来解决此问题：
* 通过减少冗余或嵌套布局来展平View层次结构。
* 会在启动期间不填充的部分UI内容不需要显示。相反，使用`ViewStub`对象作为子层次结构的占位符，应用程序可以在更合适的时间inflate
* 在主线程上进行所有资源初始化也会降低启动速度。您可以按如下方式解决此问题：
* 移动所有资源初始化，以便应用程序可以在另一个线程上加载它。
* 允许应用加载并显示您的View，然后更新依赖于Bitmap和其他资源的可视属性。

**3. 主题化启动页**

您可能希望以应用程序的加载体验为主题，以便应用程序的启动屏幕在主题上与应用程序的其余部分保持一致，而不是系统主题。这样做可以隐藏Activity启动的缓慢。

实现主题启动屏幕的常用方法是使用`windowDisablePreview`主题属性来关闭启动应用程序时系统进程绘制的初始空白屏幕。但是，与不抑制预览窗口的应用程序相比，此方法可能会导致启动时间更长。此外，它会强制用户在Activity启动时无反馈的等待，这使他们疑惑应用程序是否在正常运行。

**问题诊断**

您可以通过在用户启动应用时观察缓慢的响应来诊断此问题。在这种情况下，屏幕似乎被冻结，或者已经停止响应输入。

**问题的解决方案**

我们建议您不要禁用预览窗口，而是遵循常见的[Material Design](http://www.google.com/design/spec/patterns/launch-screens.html)模式。您可以使用activity的`windowBackground`主题属性为初始Activity提供简单的自定义drawable。

例如，您可以创建一个新的drawable文件，并在布局XML和应用程序清单文件中引用它，如下所示：

Layout XML file:

```
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" android:opacity="opaque">
  <!-- The background color, preferably the same as your normal theme -->
  <item android:drawable="@android:color/white"/>
  <!-- Your product logo - 144dp color version of your app icon -->
  <item>
    <bitmap
      android:src="@drawable/product_logo_144dp"
      android:gravity="center"/>
  </item>
</layer-list>
```

Manifest file:

```
<activity ...
android:theme="@style/AppTheme.Launcher" />
```

过渡回正常主题的最简单方法是在`super.onCreate()`和`setContentView()`之前调用`setTheme(R.style.AppTheme)`：

```
class MyMainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        // Make sure this is before calling super.onCreate
        setTheme(R.style.Theme_MyApp)
        super.onCreate(savedInstanceState)
        // ...
    }
}
```

### 5 Apk包大小优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#5-apk) <a href="#5-apk" id="5-apk"></a>

[Reduce your app size](https://developer.android.com/topic/performance/reduce-apk-size)

在讨论如何减小应用的大小之前，了解应用 APK 的结构会很有帮助。APK 文件由 Zip 压缩文件（其中包含构成应用的所有文件）组成。这些文件包括 Java 类文件、资源文件和包含已编译资源的文件。

APK 包含以下目录：

* `META-INF/`：包含 `CERT.SF` 和 `CERT.RSA` 签名文件，以及 `MANIFEST.MF` 清单文件。
* `assets/`：包含应用的资源；应用可以使用 `AssetManager` 对象检索这些资源。
* `res/`：包含未编译到 `resources.arsc` 中的资源。
* `lib/`：包含特定于处理器软件层的编译代码。此目录包含每种平台类型的子目录，如 `armeabi`、`armeabi-v7a`、`arm64-v8a`、`x86`、`x86_64` 和 `mips`。

APK 还包含以下文件。在这些文件中，只有 `AndroidManifest.xml` 是必需的。

* `resources.arsc`：包含已编译的资源。此文件包含 `res/values/` 文件夹的所有配置中的 XML 内容。打包工具会提取此 XML 内容，将其编译为二进制文件形式，并将相应内容进行归档。此内容包括语言字符串和样式，以及未直接包含在 `resources.arsc` 文件中的内容（例如布局文件和图片）的路径。
* `classes.dex`：包含以 Dalvik/ART 虚拟机可理解的 DEX 文件格式编译的类。
* `AndroidManifest.xml`：包含核心 Android 清单文件。此文件列出了应用的名称、版本、访问权限和引用的库文件。该文件使用 Android 的二进制 XML 格式。

#### 5.1 减少资源数量和大小[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#51) <a href="#51" id="51"></a>

APK 的大小会影响应用加载速度、使用的内存量以及消耗的电量。减小 APK 大小的一种简单方法是减少其包含的资源数量和大小。具体来说，您可以移除应用不再使用的资源，并且可以用可伸缩的 Drawable 对象取代图片文件。此部分将讨论上述这些方法，以及另外几种可减少应用中的资源以减小 APK 总大小的方法。

**5.1.1 移除未使用的资源¶**

[lint](https://developer.android.com/studio/write/lint.html)工具是 Android Studio 中附带的静态代码分析器，可检测到 `res/` 文件夹中未被代码引用的资源。当 `lint` 工具发现项目中有可能未使用的资源时，会显示一条消息，如下例所示。

```
res/layout/preferences.xml: Warning: The resource R.layout.preferences appears to be unused [UnusedResources]
```

**注意：lint** 工具不会扫描 **assets/** 文件夹、通过反射引用的资源或已链接到应用的库文件。此外，它也不会移除资源，只会提醒您它们的存在。

您添加到代码的库可能包含未使用的资源。如果您在应用的 `build.gradle` 文件中启用了 [shrinkResources](https://developer.android.com/studio/build/shrink-code.html)，则 Gradle 可以代表您自动移除资源。

```
    android {
        // Other settings

        buildTypes {
            release {
                minifyEnabled true
                shrinkResources true
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }
```

要使用 `shrinkResources`，您必须启用代码压缩功能。在编译过程中，首先，[ProGuard](https://developer.android.com/studio/build/shrink-code.html) 会移除未使用的代码，但会保留未使用的资源。然后，Gradle 会移除未使用的资源。

在 Android Gradle Plugin 0.7 及更高版本中，您可以声明应用支持的配置。Gradle 会使用 `resConfig` 和 `resConfigs` 风格以及 `defaultConfig` 选项将这些信息传递给编译系统。随后，编译系统会阻止来自其他不受支持配置的资源出现在 APK 中，从而减小 APK 的大小。有关此功能的详情，请参见[移除未使用的备用资源](https://developer.android.com/studio/build/shrink-code.html#unused-alt-resources)。

**5.1.2 最大限度减少库中的资源使用¶**

如果库是为服务器或桌面设备设计的，则它可能包含应用不需要的许多对象和方法。要仅包含您的应用所需的库部分，您可以编辑库的文件（如果相应的许可允许您修改库）。您还可以使用其他适合移动设备的库来为应用添加特定功能。

**注意：**[ProGuard](https://developer.android.com/studio/build/shrink-code.html) 可以清理随库导入的一些不必要代码，但它无法移除库的大型内部依赖项。

**5.1.3 仅支持特定密度¶**

Android 支持数量非常广泛的设备（包含各种屏幕密度）。在 Android 4.4（API 级别 19）及更高版本中，框架支持各种密度：`ldpi`、`mdpi`、`tvdpi`、`hdpi`、`xhdpi`、`xxhdpi` 和 `xxxhdpi`。尽管 Android 支持所有这些密度，但您无需将资源导出到每个密度。

如果您知道只有一小部分用户拥有具有特定密度的设备，请考虑是否需要将这些密度打包到您的应用中。如果您不添加用于特定屏幕密度的资源，Android 会自动扩缩最初为其他屏幕密度设计的现有资源。

如果您的应用仅需要扩缩的图片，则可以通过在 `drawable-nodpi/` 中使用图片的单个变体来节省更多空间。我们建议每个应用至少包含一个 `xxhdpi` 图片变量。

有关屏幕密度的详情，请参见[屏幕尺寸和密度](https://developer.android.com/about/dashboards/index.html#Screens)。不同像素密度的配置限定符

| Density qualifier | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ldpi              | Resources for low-density (ldpi) screens (\~120dpi).                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| mdpi              | Resources for medium-density (mdpi) screens (\~160dpi). (This is the baseline density.)                                                                                                                                                                                                                                                                                                                                                                                                                       |
| hdpi              | Resources for high-density (hdpi) screens (\~240dpi).                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| xhdpi             | Resources for extra-high-density (xhdpi) screens (\~320dpi).                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| xxhdpi            | Resources for extra-extra-high-density (xxhdpi) screens (\~480dpi).                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| xxxhdpi           | Resources for extra-extra-extra-high-density (xxxhdpi) uses (\~640dpi).                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| nodpi             | Resources for all densities. These are density-independent resources. The system does not scale resources tagged with this qualifier, regardless of the current screen's density.                                                                                                                                                                                                                                                                                                                             |
| tvdpi             | Resources for screens somewhere between mdpi and hdpi; approximately 213dpi. This is not considered a "primary" density group. It is mostly intended for televisions and most apps shouldn't need it—providing mdpi and hdpi resources is sufficient for most apps and the system will scale them as appropriate. If you find it necessary to provide tvdpi resources, you should size them at a factor of 1.33\*mdpi. For example, a 100px x 100px image for mdpi screens should be 133px x 133px for tvdpi. |

**5.1.4 使用可绘制对象¶**

某些图片不需要静态图片资源；framework可以在运行时动态地绘制图片。`Drawable`对象（XML 中为 ）会占用 APK 中的少量空间。此外，XML `Drawable` 对象会生成符合 Material Design 准则的单色图片。

**5.1.5 重复使用资源¶**

您可以为图片的变体添加单独的资源，例如同一图片经过tinted、shaded或rotated的版本。不过，我们建议您重复使用同一组资源，并在运行时根据需要对其进行自定义。

Android 提供了一些实用工具来更改资源的颜色，每个实用工具在 Android 5.0（API 级别 21）及更高版本上都使用 `android:tint` 和 `tintMode` 属性。对于较低版本的平台，则使用 [ColorFilter](https://developer.android.com/reference/android/graphics/ColorFilter.html) 类。

您还可以忽略仅是另一个资源的旋转等效的资源。以下代码段提供了一个示例，展示了通过绕图片中心位置旋转 180 度，将“拇指向上”变为“拇指向下”：

```
<?xml version="1.0" encoding="utf-8"?>
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_thumb_up"
    android:pivotX="50%"
    android:pivotY="50%"
    android:fromDegrees="180" />
```

**5.1.6 从代码进行渲染¶**

您还可以通过按一定程序渲染图片来减小 APK 大小。按一定程序渲染可以释放空间，因为您不再在 APK 中存储图片文件。

**5.1.7 压缩 PNG 文件¶**

`aapt`工具可以在编译过程中通过无损压缩来优化放置在 `res/drawable/` 中的图片资源。例如，`aapt` 工具可以通过调色板将不需要超过 256 种颜色的真彩色 PNG 转换为 8 位 PNG。这样做会生成质量相同但内存占用量更小的图片。

请记住，`aapt` 具有以下限制：

* `aapt` 工具不会压缩 `asset/` 文件夹中包含的 PNG 文件。
* 图片文件需要使用 256 种或更少的颜色才可供 `aapt` 工具进行优化。
*   `aapt` 工具可能会增大已压缩的 PNG 文件。为防止出现这种情况，您可以使用 Gradle 中的 `cruncherEnabled` 标记为 PNG 文件停用此过程：

    ```
    aaptOptions {
        cruncherEnabled = false
    }
    ```

**5.1.8 压缩 PNG 和 JPEG 文件¶**

您可以使用 [pngcrush](https://pmt.sourceforge.io/pngcrush/)、[pngquant](https://pngquant.org/) 或 [zopflipng](https://github.com/google/zopfli) 等工具减小 PNG 文件的大小，同时不损失画质。所有这些工具都可以减小 PNG 文件的大小，同时保持肉眼感知的画质不变。

`pngcrush` 工具尤为有效：该工具会迭代 PNG 过滤器和 zlib (Deflate) 参数，使用过滤器和参数的每个组合来压缩图片。然后，它会选择可产生最小压缩输出的配置。

要压缩 JPEG 文件，您可以使用 [packJPG](http://www.elektronik.htw-aalen.de/packjpg/) 和 [guetzli](https://github.com/google/guetzli) 等工具。

**5.1.9 使用 WebP 文件格式¶**

如果以 Android 3.2（API 级别 13）及更高版本为目标，您还可以使用 [WebP](https://developers.google.com/speed/webp/) 文件格式的图片（而不是使用 PNG 或 JPEG 文件）。WebP 格式提供有损压缩（如 JPEG）以及透明度（如 PNG），不过与 JPEG 或 PNG 相比，这种格式可以提供更好的压缩效果。

您可以使用 Android Studio 将现有 BMP、JPG、PNG 或静态 GIF 图片转换为 WebP 格式。有关详情，请参见[使用 Android Studio 创建 WebP 图片](https://developer.android.com/studio/write/convert-webp.html)。

注意：仅当[启动器图标](https://material.io/guidelines/style/icons.html#icons-product-icons)使用 PNG 格式时，Google Play 才会接受 APK。

**5.1.10 使用矢量图形¶**

您可以使用矢量图形创建与分辨率无关的图标和其他可伸缩媒体。使用这些图形可以极大地减少 APK 占用的空间。矢量图片在 Android 中以 `VectorDrawable` 对象的形式表示。借助 `VectorDrawable` 对象，100 字节的文件可以生成与屏幕大小相同的清晰图片。

不过，系统渲染每个 `VectorDrawable` 对象需要花费大量时间，而较大的图片则需要更长的时间才能显示在屏幕上。因此，请考虑仅在显示小图片时使用这些矢量图形。

有关使用 `VectorDrawable` 对象的详情，请参见[使用可绘制资源](https://developer.android.com/training/material/drawables.html)。

**5.1.11 将矢量图形用于动画图片¶**

请勿使用 `AnimationDrawable` 创建逐帧动画，因为这样做需要为动画的每个帧添加单独的位图文件，而这会大大增加 APK 的大小。

您应改为使用 `AnimatedVectorDrawableCompat` 创建[动画矢量可绘制资源](https://developer.android.com/training/material/animations.html#AnimVector)。

#### 5.2 减少原生和 Java 代码[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#52-java) <a href="#52-java" id="52-java"></a>

您可以使用多种方法来减小应用中 Java 和原生代码库的大小。

**5.2.1 移除不必要的生成代码¶**

确保了解自动生成的任何代码所占用的空间。例如，许多协议缓冲区工具会生成过多的方法和类，这可能会使应用的大小增加一倍或两倍。

**5.2.2 避免使用枚举¶**

单个枚举会使应用的 `classes.dex` 文件增加大约 1.0 到 1.4 KB 的大小。这些增加的大小会快速累积，产生复杂的系统或共享库。如果可能，请考虑使用 `@IntDef` 注释和 [ProGuard](https://developer.android.com/studio/build/shrink-code.html) 移除枚举并将它们转换为整数。此类型转换可保留枚举的各种安全优势。

**5.2.3 减小原生二进制文件的大小¶**

如果您的应用使用原生代码和 Android NDK，您还可以通过优化代码来减小发布版本应用的大小。移除调试符号和不提取原生库是两项很实用的技术。

**移除调试符号**

如果应用正在开发中且仍需要调试，则使用调试符号非常合适。您可以使用 Android NDK 中提供的 `arm-eabi-strip` 工具从原生库中移除不必要的调试符号。之后，您便可以编译发布版本。

**避免解压缩原生库**

在编译应用的发布版本时，您可以通过在应用清单的 `<application>` 元素中设置 `android:extractNativeLibs="false"`，打包 APK 中未压缩的 `.so` 文件。停用此标记可防止 `PackageManager` 在安装过程中将 `.so` 文件从 APK 复制到文件系统，并具有减小应用更新的额外好处。

#### 5.3 维持多个精简 APK[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#53-apk) <a href="#53-apk" id="53-apk"></a>

APK 可能包含用户下载但从不使用的内容，例如其他语言或针对屏幕密度的资源。要确保为用户提供最小的下载文件，您应该[使用 Android App Bundle](https://developer.android.com/topic/performance/reduce-apk-size#app\_bundle) 将应用上传到 Google Play。通过上传 App Bundle，Google Play 能够针对每位用户的设备配置生成并提供经过优化的 APK，因此用户只需下载运行您的应用所需的代码和资源。您无需再编译、签署和管理多个 APK 以支持不同的设备，而用户也可以获得更小、更优化的下载文件包。

如果您不打算将应用发布到 Google Play，则可以将应用细分为多个 APK，并按屏幕尺寸或 GPU texture支持等因素进行区分。

当用户下载您的应用时，他们的设备会根据设备的功能和设置接收正确的 APK。这样，设备不会接收用于设备所不具备的功能的资源。例如，如果用户具有 hdpi 设备，则不需要您可能会为具有更高密度显示器的设备提供的 xxxhdpi 资源。

有关详情，请参见[Configure APK Splits](https://developer.android.com/studio/build/configure-apk-splits.html)和[Maintaining Multiple APKs](https://developer.android.com/google/play/publishing/multiple-apks)。

### 6 ListView和Bitmap优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#6-listviewbitmap) <a href="#6-listviewbitmap" id="6-listviewbitmap"></a>

`ListView`和`GridView`的优化主要分为三个方面：

1. 要采用`ViewHolder`并避免在`getView`中执行耗时操作。
2. 根据列表的滑动状态来控制任务的执行频率，如当列表快速滑动时显然不太适合开启大量的异步任务。
3. 可以尝试开启硬件加速来使`ListView`的滑动更加流畅。

不过目前都在使用RecyclerView，自带了缓存优化。[ListView和RecyclerView的缓存原理](https://blog.yorek.xyz/android/other/recyclerview-cache/)需要了解一下。

`Bitmap`的优化主要是通过`BitmapFactory.Options`来根据需要对图片进行采样，采样过程主要采用到了`BitmapFactory.Options#inSampleSize`参数。

### 7 线程优化[¶](https://blog.yorek.xyz/android/framework/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96/#7) <a href="#7" id="7"></a>

线程优化的思想是采用线程池，避免在程序中使用大量的`Thread`。可以考虑使用线程池。

[Android线程与线程池](https://blog.yorek.xyz/android/framework/Android%E7%BA%BF%E7%A8%8B%E4%B8%8E%E7%BA%BF%E7%A8%8B%E6%B1%A0/)

线程池的好处可以概括为一下三点：

* 重用线程池中的线程，可以避免因为线程的创建和销毁带来的性能开销
* 能有效控制线程池的最大并发数，避免大量的线程之间因为互相抢占系统资源而导致的阻塞现象
* 能够对线程进行简单的管理，并提供定时执行以及指定间隔循环执行等功能

`ThreadPoolExecutor`执行任务时大致遵循以下规则：

1. 如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个核心线程来执行任务。
2. 如果线程池中的线程数量已经达到或者超过了核心线程的数量，那么任务会被插入到队列中排队等待执行。
3. 如果无法插入到队列中，这说明任务队列已满。这时候如果线程未达到线程池规定的最大值，那么会立刻启动一个非核心线程来执行任务。
4. 如果步骤3中的线程数量已经达到了线程池规定的最大值，那么就会拒绝执行此任务，线程池会调用RejectedExecutionHandler#rejectedExecution来通知调用者。

最后更新: 2020年3月13日\

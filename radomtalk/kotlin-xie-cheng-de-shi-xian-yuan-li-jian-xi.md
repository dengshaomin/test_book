# Kotlin 协程的实现原理简析



![大纲](https://image.youcute.cn/FlTdIzchzNTr9PPR7aSFS7kI46xV)



### suspend <a href="#suspend" id="suspend"></a>

​使用 suspend 表示函数支持挂起操作，目的在于告诉编译器，该方法可能产生阻塞(因为普通方法也能使用 suspend 标记，但是实际不会起作用)。suspend 方法会被编译为继承 SuspendLambda 的类

### 创建协程 <a href="#chuang-jian-xie-cheng" id="chuang-jian-xie-cheng"></a>

*   launch

    返回 _Job_，​能够获取和控制当前 Scope 下的协程的状态，比如取消(cancel)协程、获取是否运行(isActive)。
*   async

    ​返回 _Deferred_，Deferred 继承自 Job，它拥有 Job 的功能。另外 Deferred 还类似 Java UTC 中的 Future 接口，通过 Deferred 能够获取协程执行完毕之后的结果。
*   withContext

    ​切换代码块的执行上下文

### 结构化并发 <a href="#jie-gou-hua-bing-fa" id="jie-gou-hua-bing-fa"></a>

​将多个协程放到一个可以管理的空间里面，这个空间的名字就叫 CoroutineScope。通过 Scope 就能够统一的管理内部的协程，方便对多个协程整体上做取消(cancel)、等待结果(join)等操作。

## 实现原理 <a href="#shi-xian-yuan-li" id="shi-xian-yuan-li"></a>

Kotlin 的协程实现属于[有限状态机编程](https://zh.wikipedia.org/wiki/%E8%87%AA%E5%8A%A8%E6%9C%BA%E7%BC%96%E7%A8%8B), 有限状态机编程是[编程范式](https://zh.wikipedia.org/wiki/%E7%BC%96%E7%A8%8B%E8%8C%83%E5%9E%8B)的一种。是指利用有限状态机来进行编程实现。

Kotlin 在 JVM 上实现的协程机制，本身没有超脱出 JVM 的范畴，也不能够超脱，所以代码本质还是运行在操作系统级别的线程上的。所以 Kotlin Coroutine 的实现就是要在 JVM 上实现一套代码(任务)的挂起和恢复机制，而挂起和恢复刚好能抽取为两种状态，于是要实现代码运行的挂起和恢复的需求，就转变为了实现一种控制状态转移的需求。而有限状态下的编程，使用有限状态机编程范式再合适不过了。

Kotlin 协程本质还是运行在线程上的，所以如果从代码的运行角度来看，并没有太多的魔法，代码的运行机制和传统一样，虚拟机按行读取指令和操作数，然后执行操作。所以 Kotlin 协程的魔法更多是在编译期就开始了。

### ​执行的最小单元 CodeBlock <a href="#zhi-hang-de-zui-xiao-dan-yuan-codeblock" id="zhi-hang-de-zui-xiao-dan-yuan-codeblock"></a>

在 Kotlin 代码编译时，_launch_/_async_ 和 _suspend_ 方法中的代码会根据挂起点被拆分到多个 **Code Block** 中。Code Block 会被封装为 `Runnable`，被封装的 `Runnable` 会被 _Dispatcher_ 执行。使用哪种 _Dispatcher_ 则由传递给 _launch_/_async_ 的 _CoroutineContext_ 参数决定。

_例子代码:_

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre class="language-kotlin"><code class="lang-kotlin">class RunBlockingDemo {
    fun demo() {
        runBlocking {
            println(1) // block 1
            launch { // block 1
                println(3) // block 2
                suspendFunc() // block 2
                println(6) // block 3
            }
            println(2) // block 1
        }
    }

    private suspend fun suspendFunc() {
        println(4) // block 4
        delay(500) // block 4
        println(5) // block 5
    }
}
</code></pre> |

例子代码中，使用的 `runBlocking` 来创建协程，`runBlocking` 会为内部的协程提供一个阻塞当前线程的 _EventLoop_ 队列，等队列中的所有协程都执行完之后，`runBlocking` 才会执行完毕。例子代码中的 `launch` 没有传递额外的 CoroutineContext 参数，所以它会继承 `runBlocking` 的 context 去使用。

例子代码的执行过程可以简述为:

| 运行流                                                        | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ![](https://image.youcute.cn/FuYgwCHQXViqtkj1ZjZywAvMu70g) | <p>1. 调用 <code>runBlocking</code>，<em>block 1</em> 入队，然后开始循环从队列中消费任务<br>2. <em>block 1</em> 出队执行，输出 <code>1</code><br>3. <code>launch</code> 被调用，<em>block 2</em> 入队<br>4. 输出 <code>3</code><br>5. <em>block 1</em> 执行完成，执行继续从队列中取下一个任务<br>6. <em>block 2</em> 出队执行，输出 <code>3</code>，调用 <em>suspendFunc</em> 方法<br>8. <em>suspendFunc</em> 方法在编译之后，会在调用时先持有它自己的代码执行完成之后将要继续执行的代码块(<em>block 3</em>)的引用(在它自己的代码执行完成之后，就会恢复执行 <em>block 3</em>)<br>9. 执行 <em>suspendFunc</em> ，输出 <code>4</code><br>10. 调用 <em>delay</em> ，<em>block 5</em> 会被封装为一个 delay task 并入队<br>11. <em>block 2</em> 和 <em>block 4</em> 执行完毕，执行继续从队列中取下一个任务<br>12. 这时队列中存在的任务是 <em>block 5</em> ，循环会一直循环等待，直到到满足了 <em>block 5</em> delay 的时间时就将 <em>block 5</em> 出队<br>13. <em>block 5</em> 出队执行，输出 <code>5</code><br>14. <em>block 5</em> 执行完成就代表 <em>suspendFunc</em> 执行完毕了，就会恢复执行 <em>suspendFunc</em> 在进入时持有的 <em>block 3</em><br>15. <em>block 3</em> 执行，输出 <code>6</code><br>16. <em>runBlocking</em> 中的所有代码执行完毕，程序执行完毕</p> |

通过上面这个例子能看出，虽然所有代码都是在同一个线程执行的，但是 Kotlin 协程却实现了非阻塞的运行(`println(2)` 不会被 `println(3)` 阻塞)，而协程内部又是按照同步的方式执行的(`println(5)` 在 `delay` 完成之后才会被执行)。

正是由于编译器将来自不同协程的代码块相互交错的插入到事件循环队列中，才让仅使用一个线程就能实现代码块的挂起和恢复得以实现。

其他的 CoroutineContext 或许使用不同的 Dispatcher 在不同的线程上采用不同的策略去执行协程，但是其过程与这个例子是类似的。

### ​维护执行状态的 Continuation <a href="#wei-hu-zhi-hang-zhuang-tai-de-continuation" id="wei-hu-zhi-hang-zhuang-tai-de-continuation"></a>

到目前为止，我们已经了解了使代码块执行的逻辑概念了。但是，在该概念是如何实现上面还存在一些疑惑。

1. 如何在已编译的 Java 字节码中实现这个逻辑概念？
2. 如何跟踪代码块的执行状态？或者说如何决定一个代码块执行完成之后该执行的哪一个代码块？

![生成的类](https://image.youcute.cn/FtN\_6GhoaFek8QMzuoLIEJgAbz3a)

我尝试通过 JD-GUI 去反编译 _RunBlockingDemo_ 类，但是 _suspendFuncA_ 方法 JD-GUI 反编译失败了，没有显示出来。于是我先通过 `javap RunBlockingDemo` 查看了 _RunBlockingDemo_ 类会有哪些方法，发现 _suspendFuncA_ 被编译为了下面的样子：

|                                                                                                                                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre class="language-java"><code class="lang-java">final java.lang.Object suspendFuncA(kotlin.coroutines.Continuation&#x3C;? super kotlin.Unit>);
</code></pre> |

然后我再使用 `javap -c RunBlockingDemo` 反编译得到编译之后的字节码:

\*suspendFuncA\* 方法的反编译字节码太长, 点击可展开

下面是我根据反编译出来的字节码推测出的 _RunBlockingDemo_ 类的 _suspendFuncA_ 方法经过编译器处理之后的源码:

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <pre class="language-java"><code class="lang-java">final Object suspendFuncA(Continuation&#x3C;?> continuation){
    if(!continuation istanceOf RunBlockingDemo$suspendFuncA$1 || 
        (RunBlockingDemo$suspendFuncA$1)continuation.label &#x26; Integer.MIN_VALUE == 0 ){
        continuation = new RunBlockingDemo$suspendFuncA$1(this);
    }
    continuation.label = contination.label - Integer.MIN_VALUE;
    Object obj = IntrinsicsKt.getCOROUTINE_SUSPEND();
    switch(continuation.label) {
        case 0: 
            println(4);
            continuation.L$0 = this;
            continuation.label = 1;
            if(delay(500, continuation) == obj) {
                return obj;
            }
        case 1:
            ResultKt.throwOnFailure(continuation.result);
            println(5);
            return Unit.INSTANCE;
        default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
    }
}
</code></pre> |

从源码中能得出两个有趣的地方:

1. 例子中的 `supendFuncA` 是没有参数的，但是编译之后，编译器为它增加了一个 _Continuation_ 类型的参数
2. `suspendFuncA` 内部通过一个 _switch-case_ 去调度执行不同的代码块，而 _switch_ 的参数则是传入的 _Continuation_ 对象的 _label_ 字段

因此不难推测出，编译器为 `suspendFuncA` 方法增加的 _Continuation_ 参数就是用来跟踪协程运行状态的对象，而且运行状态就保存在一个简单的 _int_ 类型的 `label` 字段中。

再回头看看 `suspendFuncA` 方法是如何被调用的。

* 首先编译器会为 _suspend_ 方法生成一个实现了 _ContinuationImpl_ 的包装类，然后在包装类的 _invokeSuspend_ 方法中调用 `suspendFuncA`。\
  下面就是编译器为 `suspendFuncA` 方法生成的包装类(使用 JD-GUI 查看):

|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| <pre class="language-java"><code class="lang-java">class RunBlockingDemo$suspendFuncA$1 extends ContinuationImpl {
    int label;

    Object L$0;

    @Nullable
    public final Object invokeSuspend(@NotNull Object $result) {
        this.result = $result;
        this.label |= Integer.MIN_VALUE;
        return RunBlockingDemo.this.suspendFuncA(this);
    }

    RunBlockingDemo$suspendFuncA$1(Continuation paramContinuation) {
        super(paramContinuation);
    }
}
</code></pre> |

![继承关系](https://image.youcute.cn/FmW5j3UdDZLvF2Zwk6An6gI01dUA)


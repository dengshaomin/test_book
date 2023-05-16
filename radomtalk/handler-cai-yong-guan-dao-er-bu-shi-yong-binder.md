# Handler采用管道而不使用Binder



1. 关于**Binder**，来自知乎的讨论：[https://www.zhihu.com/question/39440766/answer/89210950](https://www.zhihu.com/question/39440766/answer/89210950)
2. 关于**Handler**为何不采用Binder， 先总结说一句，Handler完全可以通过BInder，但是杀鸡焉用牛刀。 后面再补充...

如果仅仅从上层来看待这个问题，那是盲人摸象。这个问题需要从上往下追溯才能看清本原，重头戏也在底层，这里的底层不仅仅是Native层，还需要对Linux所有了解，才能真正理解透彻。看到前面大家的回答，我觉得有必要尽早回复一下，以免有些回答者误导更多新人。

**（1）**有人说管道是Linux IPC机制，IPC英文全称Inter-Process Communication，意思是进程间通信，而Handler是用于同一进程内的线程间通信，怎么会用到管道呢？这样的回答应该是看了一些教科书，但并没有真正理解透彻，这是不对的。**Handler底层的确是采用管道机制。**

先简单说说管道，其本质是也是文件，但又和普通的文件会有所不同：管道缓冲区大小一般为1页，即4K字节。管道分为读端和写端，读端负责从管道拿数据，当数据为空时则阻塞；写端向管道写数据，当管道缓存区满时则阻塞。

**（2）**再说说Handler是如何使用管道的\
有人说Handler所涉及的MessageQueue/Message/Looper/Handler这4个类都是采用Java实现，哪来的底层采用管道的机会？这是不对的

在Looper.loop方法，会不断循环处理Message，其中消息的获取是通过

```
Message msg = queue.next(); //用于获取消息队列中的下一条消息

```

该方法中会调用nativePollOnce()方法，这便是一个native方法，再通过JNI调用进入Native层，便采用了管道，比如epoll\_create/epoll\_wait/epoll\_ctl，这里通过一副图来让大家看清楚Java层与native的联系。

![](<../.gitbook/assets/image (9).png>)

流程就不细说了，直接去看代码或者我的博客 [Android消息机制2-Handler(Native层)](https://link.zhihu.com/?target=http%3A//gityuan.com/2015/12/27/handler-message-native/%23nativewake)。

**（3）**到这里有人可能好奇，既然是同一个进程间的线程通信，为何需要管道呢？

线程之间内存共享，通过Handler通信，消息池的内容并不需要从一个线程拷贝到另一个线程，\
因为两线程可使用的内存时同一个区域，都有权直接访问，当然也存在线程私有区域ThreadLocal（这里不涉及）。即然不需要拷贝内存，那管道是何作用呢？

**Handler机制中管道作用**就是当一个线程A准备好Message，并放入消息池，这时需要通知另一个线程B去处理这个消息。线程A向管道的写端写入数据1（对于老的Android版本是写入字符`W`），管道有数据便会唤醒线程B去处理消息。管道主要工作是用于通知另一个线程的，这便是最核心的作用。

**（4）**有了这些关于Handler的准备，再加上[为什么 Android 要采用 Binder 作为 IPC 机制？ - Gityuan 的回答](https://www.zhihu.com/question/39440766/answer/89210950)，再来说说handler为何采用管道而非Binder？

handler不采用Binder，并非binder完成不了这个功能，而是太浪费CPU和内存资源了。\
Binder采用C/S架构，往往用于不同进程间的通信

* 从内存角度：通信过程中还涉及一次内存拷贝，handler机制中的Message根本不需要拷贝，本身就是在同一个内存。Handler需要的仅仅是告诉另一个线程数据有了。
* 从CPU角度，为了Binder通信底层驱动还需要为何一个binder线程池，每次通信涉及binder线程的创建和内存分配等比较浪费CPU资源。

Handler不宜采用Binder，杀鸡焉用牛刀。

**（5）**Binder Vs Handler

Binder用于进程间通信，而Handler消息机制用于同进程的线程间通信。

* 有人可能会疑惑，为何Binder/Socket用于进程间通信，能否用于线程间通信呢？答案是肯定，对于两个具有独立地址空间的进程通信都可以，当然也能用于共享内存空间的两个线程间通信，这就好比杀鸡用牛刀。
* 接着可能还有人会疑惑，那handler消息机制能否用于进程间通信？答案是不能，Handler只能用于共享内存地址空间的两个线程间通信，即同进程的两个线程间通信。

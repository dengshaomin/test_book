# 理解Java中synchronized关键词

### Question[¶](https://blog.yorek.xyz/android/paid/zsxq/week1-synchronized/#question) <a href="#question" id="question"></a>

理解Java中的synchronized关键字。\
指标：理解synchronized的含义、明确synchronized关键字修饰普通方法、静态方法和代码块时锁对象的差异。

有如下一个类A

```
class A {
    public synchronized void a() {
    }

    public synchronized void b() {
    }
}
```

然后创建两个对象

```
A a1 = new A();
A a2 = new A();
```

然后在两个线程中并发访问如下代码：

```
Thread1                       Thread2
a1.a();                       a2.a();
```

请问二者能否构成线程同步？

如果A的定义是下面这种呢？

```
class A {
    public static synchronized void a() {
    }

    public static synchronized void b() {
    }
}
```

### Answer[¶](https://blog.yorek.xyz/android/paid/zsxq/week1-synchronized/#answer) <a href="#answer" id="answer"></a>

问题1 ：不能同步\
问题2 ：能同步

关于`synchronized`关键词作用，可以参考[Java常见概念——线程——第3点](https://blog.yorek.xyz/java/java-foundation/#6)

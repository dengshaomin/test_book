# jvm之方法内联优化

### 前言

在日常中工作中，我们时不时会代码进行一些优化，比如用新的算法，简化计算逻辑，减少计算量等。对于java程序来说，除了开发者本身对代码优化之外，还有一个"人"也在背后默默的优化我们的代码，这个"人"就是jvm。jvm会帮我们分析出热点代码，优化代码逻辑。其中jvm最常做的优化之一就是：**方法内联优化**。

### 方法内联

什么是方法内联？又可以叫做函数内联，java中方法可等同于其它语言中的函数。关于方法内联维基百科上面解释是：

> 在[计算机科学](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%25E9%259B%25BB%25E8%2585%25A6%25E7%25A7%2591%25E5%25AD%25B8)中，**内联函数**（有时称作**在线函数**或**编译时期展开函数**）是一种\<u style="text-decoration: none; border-bottom: 1px dashed grey;">[编程语言](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%25E7%25B7%25A8%25E7%25A8%258B%25E8%25AA%259E%25E8%25A8%2580)\</u>结构，用来建议[编译器](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%25E7%25B7%25A8%25E8%25AD%25AF%25E5%2599%25A8)对一些特殊[函数](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%25E5%2587%25BD%25E6%2595%25B8)进行[内联扩展](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/w/index.php%3Ftitle%3D%25E5%2585%25A7%25E8%2581%25AF%25E6%2593%25B4%25E5%25B1%2595%26action%3Dedit%26redlink%3D1)（有时称作**在线扩展**）；也就是说建议编译器将指定的函数体插入并取代每一处调用该函数的地方（[上下文](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%25E4%25B8%258A%25E4%25B8%258B%25E6%2596%2587)），从而节省了每次调用函数带来的额外时间开支。

简单通俗的讲就是把方法内部调用的其它方法的逻辑，嵌入到自身的方法中去，变成自身的一部分，之后不再调用该方法，从而节省调用函数带来的额外开支。

### 函数调用开销

之所以出现方法内联是因为函数调用除了执行自身逻辑的开销外，还有一些不为人知的额外开销。**这部分额外的开销主要来自方法栈帧的生成、参数字段的压入、栈帧的弹出、还有指令执行地址的跳转**。比如有下面这样代码：

```
 public static void function_A(int a, int b){
        //do something
        function_B(a,b);
    }

    public static void function_B(int c, int d){
        //do something
    }

    public static void main(String[] args){
         function_A(1,2);
    }
```

则代码的执行过程如下：![image](https://upload-images.jianshu.io/upload\_images/6793917-d1f2fd6c24d8d1b2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)\
image

所以如果java中方法调用嵌套过多或者方法过多，这种额外的开销就越多。

试想一下想get/set这种方法调用：

```
public int getI() {
        return i;
    }

public void setI(int i) {
        this.i = i;
    }
```

**很可能自身执行逻辑的开销还比不上为了调用这个方法的额外开锁。如果类似的方法被频繁的调用，则真正相对执行效率就会很低，虽然这类方法的执行时间很短。这也是为什么jvm会在热点代码中执行方法内联的原因，这样的话就可以省去调用调用函数带来的额外开支。**

**这里举个内联的可能形式：**

```
 public int  add(int a, int b , int c, int d){
          return add(a, b) + add(c, d);
    }

    public int add(int a, int b){
        return a + b;
    }
```

**内联之后：**

```
public int  add(int a, int b , int c, int d){
          return a + b + c + d;
    }
```

**这样除了本身的相加逻辑的开销，比内联前减少了二次调用函数带来的额外开销。**

### 内联条件

一个方法如果满足以下条件就很可能被jvm内联。

1、热点代码。 如果一个方法的执行频率很高就表示优化的潜在价值就越大。那代码执行多少次才能确定为热点代码？这是根据编译器的编译模式来决定的。如果是客户端编译模式则次数是1500，服务端编译模式是10000。次数的大小可以通过-XX:CompileThreshold来调整。

2、方法体不能太大。jvm中被内联的方法会编译成机器码放在code cache中。如果方法体太大，则能缓存热点方法就少，反而会影响性能。

3、如果希望方法被内联，**尽量用private、static、final修饰**，这样jvm可以直接内联。如果是public、protected修饰方法jvm则需要进行类型判断，因为这些方法可以被子类继承和覆盖，jvm需要判断内联究竟内联是父类还是其中某个子类的方法。

所以了解jvm方法内联机制之后，会有助于我们工作中写出能让jvm更容易优化的代码，有助于提升程序的性能。

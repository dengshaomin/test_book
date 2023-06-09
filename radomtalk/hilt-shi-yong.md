# Hilt使用

官方文档：[https://developer.android.com/training/dependency-injection/hilt-android?hl=zh-cn#application-class](https://developer.android.com/training/dependency-injection/hilt-android?hl=zh-cn#application-class)

Hilt译为手把、刀柄，基于Dagger，取名意在让Dagger更容易使用；可以说是Dagger在Android平台的特性化框架。

`@HiltAndroidApp` 会触发 Hilt 的代码生成操作，生成的代码包括应用的一个基类，该基类充当应用级依赖项容器。

&#x20;`@AndroidEntryPoint` 如果某个类中有需要被注入的对象，那么这个需要标记上该注解。这个类通常可以被称为`入口`，Hilt当前支持的入口类型为:

* `Application`（通过使用 `@HiltAndroidApp`）
* `Activity`
* `Fragment`
* `View`
* `Service`
* `BroadcastReceiver`

## 对象注入

假设在一个Activity中有很多个fragment，这些fragment都是一个列表且使用相同的adapter，这个时候我不想每个adapter中都去使用new来创建adapter，那hilt注入使用步骤如下

### 注解Application

```kotlin
@HiltAndroidApp
class HiltApplication : Application() {
}
```

###  注解要注入对象的类

使用@AndroidEntryPoint表示此类内部有对象需要注入，同时使用@Inject注解哪个对象需要被自动注入

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var sampleAdapter: SampleAdapter
}
```

### 注入对象的提供者

使用@Inject标记SampleAdapter表示需要被注入的对象由我提供；大部分地方都需要使用到Context，这个Context在每个Activity又是不一样的，所以Hilt提供`@ActivityContext`来注解 这个Context是由注入对象的外部类提供，如果被注入的对象分别存在Actiivty0和Activity1中，那么这个context就分别是Activity0和Activity1。

```kotlin
class SampleAdapter @Inject constructor(@ActivityContext var context: Context) :
    RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    }
```

整个流程到这里就结束了，比Dagger用起来方便多了。

## interface的注入

interface的注入需要多一个@Module，来绑定接口以及实现类。

{% tabs %}
{% tab title="写一个接口" %}
```kotlin
interface IMainPresent {
    fun doMainWork()
    fun doItemWork(index: Int)
}
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="再来一个实现类" %}
```kotlin
class IMainPresentImp @Inject constructor() :IMainPresent {
    override fun doMainWork() {
        System.out.println("main work")
    }

    override fun doItemWork(index: Int) {
        System.out.println("item work $index")
    }
}
```
{% endtab %}
{% endtabs %}

这时候并不能顺利完成自动注入，需要再写一个@Module标记的类，来进行接口和实例的绑定

```kotlin
@Module
@InstallIn(ActivityComponent::class)
abstract class IMainPresentModule {
    @Binds
    abstract fun bind(iMainPresentImp: IMainPresentImp): IMainPresent
}
```

这样就可以完成自动注入，并在需要的类中自动注入了

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var iMainPresent: IMainPresent
    }
```

## 对象共享

开发过程中经常使用MVP结构来处理Activity中的业务以及数据接口，经常碰到的一种场景

* Activity持有Present对象，调用接口获取数据
* 在Activity中RecyclerView的Item或生命周期内的其他对象中要是用到这个Present

通过的处理方式有两种:

1. 将context强转成Activity对象并获取其中的Present
2. 向内部对象提供接口，通过接口回调

这样处理的话不是很优雅，而却传递过程很漫长。那么我们就可以用到Hitl的特性来做对象的共享，前面interface注入应帮我们在Activity中注入了IMainPresent，接下来就是我们需要Item中自动注入IMainPresent

```kotlin
@AndroidEntryPoint
class SampleItem @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {
    @Inject
    lateinit var iMainPresent: IMainPresent
    ...
    }
```

由于IMainPresentImp已经提供了自动注入，item中的示例已经可以直接使用了，但是这个实例对象和Activity中的并不是一个，这样在代码安全和内存上都是很不好的表现，可以通过`@ActivityScoped`注解来标记注解对象提供者的作用域

```kotlin
@ActivityScoped
class IMainPresentImp @Inject constructor() :IMainPresent {
    override fun doMainWork() {
        System.out.println("main work")
    }

    override fun doItemWork(index: Int) {
        System.out.println("item work $index")
    }
}
```

这样Activity和item中使用的就是同一个present，由此可以猜测出对象共享的策略应该是：在作用域范围内创建一个对象并在作用域内共享，作用域之外的会另外再新建对象。为了验证这个问题在Activity中注入Users对象，每2s输出一次Users.name，并在3s之后改变name对象内部的值，

{% tabs %}
{% tab title="Users" %}
```kotlin
@ActivityScoped
class Users constructor(var name: String) {
    @Inject
    constructor() : this("java") {
    }
}
```
{% endtab %}
{% endtabs %}

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject
    lateinit var sampleAdapter: SampleAdapter
     @Inject
    lateinit var users: Users
        override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        GlobalScope.launch {
            while (true) {
                delay(2000)
                System.out.println("activity     ${users.name}")
            }
        }
        GlobalScope.launch {
            delay(3000)
            users.name = "kotlin"
        }
        
    }
}
```

{% tabs %}
{% tab title="Adapter" %}
```kotlin
class SampleAdapter @Inject constructor(@ActivityContext var context: Context) :
    RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    @Inject
    lateinit var users: Users

    init {
        GlobalScope.launch {
            while (true) {
                delay(2000)
                System.out.println("adapter    ${users.name}")
            }
        }
    }
```
{% endtab %}
{% endtabs %}

先去掉User类的作用域注解@ActivityScoped的，activity和adapter中在值被改变之后两边的输入name不一致：

![](<../.gitbook/assets/image (195).png>)

当把作用于注解添加回去的话，在内部属性被改变后，两侧输出的值都是改变后的值

![](<../.gitbook/assets/image (175).png>)

Hilt提供类似的作用域注解还有以下几种类型：

| Android 类                             | 生成的组件                       | 作用域                      |
| ------------------------------------- | --------------------------- | ------------------------ |
| `Application`                         | `ApplicationComponent`      | `@Singleton`             |
| `View Model`                          | `ActivityRetainedComponent` | `@ActivityRetainedScope` |
| `Activity`                            | `ActivityComponent`         | `@ActivityScoped`        |
| `Fragment`                            | `FragmentComponent`         | `@FragmentScoped`        |
| `View`                                | `ViewComponent`             | `@ViewScoped`            |
| 带有 `@WithFragmentBindings` 注释的 `View` | `ViewWithFragmentComponent` | `@ViewScoped`            |
| `Service`                             | `ServiceComponent`          | `@ServiceScoped`         |

## Activity、Fragment共享viewmodel

fragment使用 `@AndroidEntryPoint`注解标记，viewmodel使用`activityViewModels`

```kotlin
@AndroidEntryPoint
class HomeChildFragment {
    private val cityViewModel by activityViewModels<CityViewModel>()
}
```

## 本章Demo

[https://github.com/dengshaomin/Hilt](https://github.com/dengshaomin/Hilt)

# findviewbyid的一些替代方案

findviewbyid一直是费时间而且重复的工作，替代的方案也不断进化，前有ButterKnife，后有Kotlin-android-extension（下面简称KAE），最后又有ViewBinding（简称VB），从个角度比较以上方案

![](<../.gitbook/assets/image (13).png>)

### ButterKnife

[GitHub](https://github.com/JakeWharton/butterknife)

出自[JakeWharton](https://github.com/JakeWharton)之手，整个过程是在项目编译阶段完成的，主要用到了 annotationProcessor 和 JavaPoet 技术，使用时通过生成的辅助类完成操作，并不是在项目运行时通过注解加反射实现的，所以并不会影响项目运行时的性能。在KAE出来之后就变得不香了。

### KAE

使用：在对应模块的build.gradle加入两个插件

```
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions' 
```

KAE是基于`Kotlin Compiler Plugin` + `Intellij Idea Plugin`实现的，IDEA Plugin让我们可以在编辑器中使用语法糖进行试图访问，Compiler Plugin可以在编译期对语法糖脱糖。整个过程是一个编译期的的闭环。 KAE的主要缺陷：一是安全，在多个布局中有同样的ID经常会导错布局，编译没问题，运行时报错；二是不能跨模块引用，会报错：Kotlin Unresolved reference

### ViewBinding

[官方介绍](https://developer.android.com/topic/libraries/view-binding?hl=zh-cn#setup)

为某个模块启用视图绑定功能后，系统会为该模块中包含的每个 XML 布局文件各生成一个绑定类。每个绑定类均包含对根视图以及具有 ID 的所有视图的引用。系统会通过以下方式生成绑定类的名称：将 XML 文件的名称转换为驼峰式大小写，并在末尾添加“Binding”一词。

![](<../.gitbook/assets/image (68).png>)

原理就是Google在那个用来编译的gradle插件中增加了新功能，当某个module开启`ViewBinding`功能后，编译的时候就去扫描此模块下的`layout`文件，生成对应的binding类。

VB目前看起来比较有优势，但是用起来略显啰嗦，没有KAE那么舒服，为了解决VB的所以需要对VB进行封装。

{% tabs %}
{% tab title="BaseBindingActivity" %}
```kotlin
open class BaseBindingActivity<VB : ViewBinding> : AppCompatActivity() {

  protected val binding: VB by lazy {
    //使用反射得到viewbinding的class
    val type = javaClass.genericSuperclass as ParameterizedType
    val aClass = type.actualTypeArguments[0] as Class<*>
    val method = aClass.getDeclaredMethod("inflate", LayoutInflater::class.java)
    method.invoke(null, layoutInflater) as VB
  }

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(binding.root)
  }
}
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="BaseBindingFragment" %}
```kotlin
open class BaseBindingFragment<VB : ViewBinding> : Fragment() {
  var binding: VB? = null

  override fun onCreateView(
    inflater: LayoutInflater,
    container: ViewGroup?,
    savedInstanceState: Bundle?
  ): View? {
    val type = javaClass.genericSuperclass as ParameterizedType
    val aClass = type.actualTypeArguments[0] as Class<*>
    val method = aClass.getDeclaredMethod(
        "inflate", LayoutInflater::class.java, ViewGroup::class.java, Boolean::class.java
    )
    binding = method.invoke(null, layoutInflater, container, false) as VB
    return binding?.root
  }

  override fun onDestroyView() {
    super.onDestroyView()
    binding = null
  }
}
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="BaseBindingView" %}
```kotlin
open class BaseBindingView<VB : ViewBinding> @JvmOverloads constructor(
  context: Context,
  attrs: AttributeSet? = null,
  defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {
  var binding: VB? = null

  init {
    val type = javaClass.genericSuperclass as ParameterizedType
    val aClass = type.actualTypeArguments[0] as Class<*>
    val method = aClass.getDeclaredMethod(
        "inflate", LayoutInflater::class.java, ViewGroup::class.java, Boolean::class.java
    )
    binding = method.invoke(null, LayoutInflater.from(context), this, true) as VB
  }

  override fun onDetachedFromWindow() {
    super.onDetachedFromWindow()
    binding = null
  }
}
```
{% endtab %}
{% endtabs %}

以上对Actiivty、Fragment以及自定义View中进行了VB的封装，原理是通过反射 inflate方法，由于反射比较消耗性能（一次大概60ms左右），加入在一个比较复杂的页面这一块消耗的时间就比较多了，这种基类反射封装的方式在这种情况下就不推荐使用。

### 总结

* ButterKnife现在基本用的很少了&#x20;
* KAE是真的好用，不安全其实还能接受，主要是不能跨模块
* ViewBinding Google官方推荐取代KAE，用起来有点啰嗦，基类反射封装消耗性能

目前还是没有一款完美替代findviewbyid的方案。每种方案都有优缺点，且在写法和实现方式上都不同，在项目中大规模使用要慎重，毕竟迁移成本比较大。

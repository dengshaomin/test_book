# Kotlin Unresolved reference

目前**kotlin-android-extensions**暂时还不支持跨模块引用布局，可以通过以下方式解决

1.定义布局变量，通过findviewbyid找到布局并赋值

```kotlin
abstract class FastRecyclerViewActivity : BaseTitleActivity(), IFastRecyclerViewCb {
  lateinit var fast_rcv_view: FastRecyclerView
  override fun setContentLayout(): Int {
    return layout.activity_fast_recyclerview
  }

  override fun initView() {
    fast_rcv_view = findViewById(R.id.fast_rcv_view)
    ...
  }

```

2.将要是用的布局加入到本模块中

现在项目大部分使用模块化、组件化，基类引用基本无法避免，暂时推荐第一种做法。

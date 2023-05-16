# AppBarLayout混淆导致的崩溃

appbar设置滚动监听事件，正常代码如下：

```kotlin
app_bar.addOnOffsetChangedListener(object : OnOffsetChangedListener {
      override fun onOffsetChanged(appBarLayout: AppBarLayout?, verticalOffset: Int) {
        ...
      }
    })
```

当使用lambda优化之后代码如下：

```kotlin
app_bar.addOnOffsetChangedListener(OnOffsetChangedListener { appBarLayout, verticalOffset ->
      ...
    })
```

神奇的是这个时候release包竟然crash了

![](<../.gitbook/assets/image (163).png>)

kotlin的锅?

java的锅?

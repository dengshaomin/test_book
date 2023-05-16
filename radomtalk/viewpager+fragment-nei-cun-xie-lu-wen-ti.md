# ViewPager+fragment 内存泄露问题

```kotlin
class CreatePageAdapter : FragmentStateAdapter {
        constructor(fragmentActivity: FragmentActivity) : super(fragmentActivity)
        constructor(fragment: Fragment) : super(fragment)
        constructor(fragmentManager: FragmentManager, lifecycle: Lifecycle) : super(
            fragmentManager,
            lifecycle
        )

        var ids = mutableListOf<Long>(
            TAB_RECOMMEND.toLong(),
            TAB_UPNEW.toLong(), TAB_ATTENTION.toLong()
        )
        var createIds = mutableListOf<Long>()
        private var recommendFragment: RecommendFragment? = null
        private var upNewFragment: UpNewFragment? = null
        private var sigleFragment: SigleFragment? = null
        private var attentionFragment: AttentionFragment? = null
        var fragmentCount = 3;
        override fun getItemCount(): Int = fragmentCount
        override fun createFragment(position: Int): Fragment {
            var id = ids[position]
            createIds.add(id)
            when (id) {
                TAB_RECOMMEND.toLong() -> {
                    L.d(TAG, "createFragment: 0")
                    if (recommendFragment == null) {
                        recommendFragment = RecommendFragment()
                    }
                    return recommendFragment!!
                }
                TAB_UPNEW.toLong() -> {
                    L.d(TAG, "createFragment: 1")
                    if (upNewFragment == null) {
                        upNewFragment = UpNewFragment()
                    }
                    return upNewFragment!!
                }
                TAB_SIGLE.toLong() -> {
                    if (sigleFragment == null) {
                        sigleFragment = SigleFragment()
                    }
                    return sigleFragment!!
                }
                TAB_ATTENTION.toLong() -> {
                    if (attentionFragment == null) {
                        attentionFragment = AttentionFragment()
                    }
                    return attentionFragment!!
                }
                else -> throw UnsupportedOperationException("fragment unsupported position $position")
            }
        }
        ...
}
```

这种createFragment写法看起来貌似也没什么问题，但是LeakCanary会检测出Adapter中的Fragment会被Hook住无法释放，原因是因为FragmentPagerStateAdapter使用List存放多个fragment造成的。更改createFragment方法如下可避免：

```kotlin
override fun createFragment(position: Int): Fragment {
            var id = ids[position]
            createIds.add(id)
            when (id) {
                TAB_RECOMMEND.toLong() -> {
                    L.d(TAG, "createFragment: 0")
                    return RecommendFragment()
                }
                TAB_UPNEW.toLong() -> {
                    L.d(TAG, "createFragment: 1")
                    return UpNewFragment()
                }
                TAB_SIGLE.toLong() -> {
                    return SigleFragment()
                }
                TAB_ATTENTION.toLong() -> {
                    return AttentionFragment()
                }
                else -> throw UnsupportedOperationException("fragment unsupported position $position")
            }
        }
```

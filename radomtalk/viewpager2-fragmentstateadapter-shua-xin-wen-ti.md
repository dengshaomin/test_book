# ViewPager2 FragmentStateAdapter 刷新问题

假如有这么一种业务场景，首页0，1，2，3总共4个fragment，第三个fragment需要根据后端接口决定是否显示，正常代码如下：

```kotlin
class MainAdapter(val activity: MainActivity) : FragmentStateAdapter(activity) {

    override fun getItemCount(): Int {
        return if (isShow3) 4 else 3
    }

    override fun createFragment(position: Int): Fragment {
        if (position == 0)
            return Fragment1()
        if (position == 1)
            return Fragment2()
        if (position == 2 && isShow3)
            return Fragment3()
        else
            return Fragment4()
        if (position == 3)
            return Fragment4()
    }
}
```

这种写法通过 notifyDataSetChanged 是没办法做到刷新的，而且会抛出异常

```kotlin
java.lang.IllegalStateException: Fragment already added
```

 通过观看源码，需要对每个fragment设置id并重写containsItem才能达到刷新目的

```kotlin
class MainAdapter(val activity: MainActivity) : FragmentStateAdapter(activity) {

    private val fid1 = 111.toLong()
    private val fid2 = 222.toLong()
    private val fid3 = 333.toLong()
    private val fid4 = 444.toLong()

    private val ids: ArrayList<Long>
        get() = if (AppConfig.isShowRSS) {
            arrayListOf(fid1, fid2, fid3, fid4)
        } else {
            arrayListOf(fid1, fid2, fid4)
        }

    private val createdIds = hashSetOf<Long>()

    override fun getItemCount(): Int {
        return if (AppConfig.isShowRSS) 4 else 3
    }

    override fun getItemId(position: Int): Long {
        return ids[position]
    }

    override fun containsItem(itemId: Long): Boolean {
        return createdIds.contains(itemId)
    }

    override fun createFragment(position: Int): Fragment {
        val id = ids[position]
        createdIds.add(id)
        return when (id) {
            fid4 -> MyFragment()
            fid3 -> RssFragment()
            fid2 -> ExploreFragment()
            else -> BookshelfFragment()
        }
    }
}
```

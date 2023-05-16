---
description: 转：https://www.jianshu.com/p/266861496508
---

# ViewPager刷新问题

![](<../.gitbook/assets/image (174).png>)

## 一、PagerAdapter介绍

先看效果图

![](https://upload-images.jianshu.io/upload\_images/1233754-1948d0b3d08847a6.png?imageMogr2/auto-orient/strip|imageView2/2/w/288/format/webp)

### PagerAdapter简介

ListView 大家应该都很熟悉吧！ListView 一般都需要一个 Adapter 来填充数据，如 ArrayAdapter、SimpleAdapter。PagerAdapter 就是 ViewPager 的 Adapter，与 ListView 的 Adapter 作用一样。

**ViewPager->PageAdapter == ListView->BaseAdapter**

先看下**官方介绍**

#### 官方介绍

![](<../.gitbook/assets/image (275).png>)

PageAdapter 继承自 Object，继承结构参考意义不大，那老实看文档。文档上没有提供示例代码，只是说了下要自定义 PageAdapter 需要实现下面四个方法：

* &#x20;**instantiateItem(ViewGroup container, int position)：**该方法的功能是创建指定位置的页面视图。适配器有责任增加即将创建的 View 视图到这里给定的 container 中，这是为了确保在 finishUpdate(viewGroup) 返回时 this is be done!\
  &#x20;返回值：返回一个代表新增视图页面的 Object（Key），这里没必要非要返回视图本身，也可以这个页面的其它容器。其实我的理解是可以代表当前页面的任意值，只要你可以与你增加的 View 一一对应即可，比如 position 变量也可以做为 Key
* &#x20;**destroyItem(ViewGroup container, int position, Object object)：**该方法的功能是移除一个给定位置的页面。适配器有责任从容器中删除这个视图，这是为了确保在 finishUpdate(viewGroup) 返回时视图能够被移除
* &#x20;**getCount()：**返回当前有效视图的数量
* &#x20;**isViewFromObject(View view, Object object)：**该函数用来判断 instantiateItem() 函数所返回来的 Key 与一个页面视图是否是代表的同一个视图(即它俩是否是对应的，对应的表示同一个 View)\
  &#x20;返回值：如果对应的是同一个View，返回 true，否则返回 false

上面对 PageAdapter 的四个抽象方法做了简要说明，下面看看如何使用

#### 简单使用

```
mContentVP.setAdapter(new PagerAdapter() {
    @Override
    public int getCount() {
        return dataList.size();
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        View view = View.inflate(SimpleDemoActivity.this, R.layout.item_vp_demopageradapter, null);
        TextView pageNumTV = (TextView) view.findViewById(R.id.tv_pagenum);
        pageNumTV.setText("DIY-PageNum-" + dataList.get(position));
        container.addView(view);
        return view;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView((View) object);
    }

});
```

可以看到实现 PagerAdapter 与 BaseAdapter 很类似，只是 PagerAdapter 的 isViewFromObject() 与 instantiateItem() 方法需要好好理解下。这里为了简化 PagerAdapter 的使用，我做了个简单的封装：

```
public abstract class APagerAdapter<T> extends PagerAdapter {

    protected LayoutInflater mInflater;
    protected List<T> mDataList;
    private SparseArray<View> mViewSparseArray;

    public APagerAdapter(Context context, List<T> dataList) {
        mInflater = LayoutInflater.from(context);
        mDataList = dataList;
        mViewSparseArray = new SparseArray<View>(dataList.size());
    }

    @Override
    public int getCount() {
        if (mDataList == null) return 0;
        return mDataList.size();
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        View view = mViewSparseArray.get(position);
        if (view == null) {
            view = getView(position);
            mViewSparseArray.put(position, view);
        }
        container.addView(view);
        return view;
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView(mViewSparseArray.get(position));
    }

    public abstract View getView(int position);

}
```

APagerAdapter 类模仿 ListView 的 BaseAdapter，抽象出一个 getView() 方法，在内部使用 SparesArray 缓存所有显示过的 View。这样使用就很简单了，继承 APagerAdapter，实现 getView() 方法即可（可以参考：DemoPagerAdapter.java）。

### PagerAdapter 刷新的问题

#### 提出问题

在使用 ListView 的时候，我们往往习惯了更新 Adapter 的数据源，然后调用 Adapter 的 notifyDataSetChanged() 方法来刷新列表（有没有点 MVC 的感觉）。

PagerAdapter 也有 notifyDataSetChanged() 方法，那我们按照这个流程来试试，看有没有什么问题。（ListView 的示例就不在这里演示了，感兴趣的可以自己去试试，非常简单）

那么我的问题是：“**ViewPager 的 PagerAdapter 在数据源更新后，能否自动刷新视图？**”

带着问题，我们做一些实验，下面实验的思路是：修改数据源，然后通知 PagerAdapter 更新，查看视图的变化。

#### 实验环境准备

看看实验环境，上代码：

```
private void initData() {
    // 数据源
    mDataList = new ArrayList<>(5);
    mDataList.add("Java");
    mDataList.add("Android");
    mDataList.add("C&C++");
    mDataList.add("OC");
    mDataList.add("Swift");

    // 很简单的一个 PagerAdapter
    this.mContentVP.setAdapter(mPagerAdapter = new PagerAdapter() {
        @Override
        public int getCount() {
            return mDataList.size();
        }

        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view == object;
        }

        @Override
        public Object instantiateItem(ViewGroup container, int position) {
            View view = View.inflate(SimpleDemoActivity.this, R.layout.item_vp_demopageradapter, null);
            TextView pageNumTV = (TextView) view.findViewById(R.id.tv_pagenum);
            pageNumTV.setText("DIY-PageNum-" + mDataList.get(position));
            container.addView(view);
            return view;
        }

        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            container.removeView((View) object);
        }

    });
}
```

ViewPager 的 Item：item\_vp\_demopageradapter.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center">
    <ImageView
        android:id="@+id/iv_img"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="@dimen/activity_horizontal_margin"
        android:src="@mipmap/ic_launcher" />
    <!-- 用于显示文本，数据更新体现在这里 -->
    <TextView
        android:id="@+id/tv_pagenum"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_margin="@dimen/activity_horizontal_margin"
        android:textAppearance="?android:attr/textAppearanceLarge"
        android:text="DIY-Page-" />
</LinearLayout>
```

很简单的代码，并且加了注释，直接往下看实验。

#### PagerAdapter 刷新实验

**1、更新数据源中的某项**

![](https://upload-images.jianshu.io/upload\_images/1233754-c0f4aa6f56a04621.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

对应代码：

```
private void refresh() {
    mDataList.set(0, "更新数据源测试");
    mPagerAdapter.notifyDataSetChanged();
}
```

**问题描述：**在演示动画中可以看到，更新数据源之后视图并没有立即刷新，多滑动几次再次回到更新的 Item 时才更新（这里先看问题，下面会细说）。

**2、往数据源中添加数据**

![](https://upload-images.jianshu.io/upload\_images/1233754-b0077d673b581238.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

对应代码：

```
private void add() {
    mDataList.add("这是新添加的Item");
    mPagerAdapter.notifyDataSetChanged();
}
```

**问题描述：**没什么问题，数据源添加数据后通知 PagerAdapter 刷新，ViewPager 中就多了一个 Item。

**3、从数据源中删除数据**

![](https://upload-images.jianshu.io/upload\_images/1233754-7221be0e86e9ea8d.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

```
private void delete() {
    mDataList.remove(0);
    mPagerAdapter.notifyDataSetChanged();
}
```

**问题描述：**这个问题就较多了，**首先，**如果是删除当前 Item，那么会看到没有任何反应；**其次，**如果删除的不是当前 Item，会发现出现了数据错乱，并且后面有 Item 滑不过去，但是按住往后滑的时候可以看到后面的 Item。

**4、将数据源清空**

![](https://upload-images.jianshu.io/upload\_images/1233754-45de0d5ed5af8dba.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

```
private void clean() {
    mDataList.clear();
    mPagerAdapter.notifyDataSetChanged();
}
```

**问题描述：**从上面的动图可以看到，清空数据源之后，会残留一个 Item。

**说明：**先不要计较上面所写的 PagerAdapter 是否有问题，这里只是想引出问题来，下面会针对 PagerAdapter、FragmentPagerAdapter 以及 FragmentStatePagerAdapter 来分析问题原因和给出解决方案。

## 二、PagerAdapter

从上面的实验可以看出 ViewPager 不同于 ListView，如果单纯的调用 ViewPager.getAdapter().notifyDataSetChanged() 方法（即 PagerAdapter 的 notifyDataSetChanged()方法）页面并没有刷新。

PagerAdapter 用于 ViewPager 的 Item 为普通 View的情况，这个相对简单，所以最先介绍。

相信很多同学都搜过类似的问题 —— “PagerAdapter 的 notifyDataSetChanged() 不刷新？”。有的说这是 bug，有的则认为 Google 是特意这样设计的，个人倾向后一种观点（我觉得这是 Google 为了 ViewPager 性能考虑而设计的，毕竟 ViewPager 需要显示“很多**大**的”视图，而且要防止用户滑动时觉得卡顿）。

### ViewPager 刷新分析

先来了解下 ViewPager 的刷新过程：\
&#x20;**1、刷新的起始**\
&#x20;ViewPager 的刷新是从调用其 PagerAdapter 的 notifyDataSetChanged() 方法开始的，那先看看该方法的源码（在源码面前一切无所遁形...）：

```
/**
 * This method should be called by the application if the data backing this adapter has changed
 * and associated views should update.
 */
public void notifyDataSetChanged() {
    synchronized (this) {
        if (mViewPagerObserver != null) {
            mViewPagerObserver.onChanged();
        }
    }
    mObservable.notifyChanged();
}
```

**2、DataSetObservable 的 notifyChanged()**\
&#x20;上面的方法中出现了两个关键的成员变量：

```
private final DataSetObservable mObservable = new DataSetObservable();
private DataSetObserver mViewPagerObserver;
```

观察者模式，有没有？先不着急分析这个是不是观察者模式，来看看 mObservable.notifyChanged() 做了些什么工作：

```
/**
 * Invokes {@link DataSetObserver#onChanged} on each observer.
 * Called when the contents of the data set have changed.  The recipient
 * will obtain the new contents the next time it queries the data set.
 */
public void notifyChanged() {
    synchronized(mObservers) {
        // since onChanged() is implemented by the app, it could do anything, including
        // removing itself from {@link mObservers} - and that could cause problems if
        // an iterator is used on the ArrayList {@link mObservers}.
        // to avoid such problems, just march thru the list in the reverse order.
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged(); 
        }
    }
}
```

notifyChanged() 方法中是很典型的观察者模式中遍历所有的 Observer，通知 变化发生了的代码。代码很简单，那关键是这个 mObservers 包含哪些 Observer 呢？

**3、DataSetObserver**\
&#x20;直接从 mObservers 点进去你会发现这个：

```
protected final ArrayList<T> mObservers = new ArrayList<T>();
```

\-\_-'，这是个泛型，坑了！还好 DataSetObservable 的 notifyChanged() 的注释中写了这些 Observer 是 DataSetObserver。那去看看 DataSetObserver：

```
public abstract class DataSetObserver {
    /**
     * This method is called when the entire data set has changed,
     * most likely through a call to {@link Cursor#requery()} on a {@link Cursor}.
     */
    public void onChanged() {
        // Do nothing
    }

    /**
     * This method is called when the entire data becomes invalid,
     * most likely through a call to {@link Cursor#deactivate()} or {@link Cursor#close()} on a
     * {@link Cursor}.
     */
    public void onInvalidated() {
        // Do nothing
    }
}
```

一个抽象类，里面两个空方法，这个好办，找他的子类（AndroidStudio 中 将光标放到类名上，按 F4）：![](https://upload-images.jianshu.io/upload\_images/1233754-7bf9fb60c74364d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/536/format/webp)DataSetObserver继承结构

总算找到你了，就是用红线框出来的那条，双击，定位过去。

**4、PagerObserver 内部类**\
&#x20;PagerObserver 是 ViewPager 中的一个内部类，实现也很简单，就是调用了 ViewPager 中的 dataSetChanged() 方法，真正的关键来了。

```
private class PagerObserver extends DataSetObserver {
    @Override
    public void onChanged() {
        dataSetChanged();
    }
    @Override
    public void onInvalidated() {
        dataSetChanged();
    }
}
```

**5、ViewPager 的 dataSetChanged()**\
&#x20;这个方法的实现较长，里面的逻辑看上去挺复杂的，这里就不展示全部的源码了，列下关键点：

```
...
for (int i = 0; i < mItems.size(); i++) {
    final ItemInfo ii = mItems.get(i);
    final int newPos = mAdapter.getItemPosition(ii.object);

    if (newPos == PagerAdapter.POSITION_UNCHANGED) {
        continue;
    }

    if (newPos == PagerAdapter.POSITION_NONE) {
        ...
        continue;
    }
    ...
}
...
```

上面截取的代码中 for 循环里面有两个 continue 语句，这可能是比较关键的代码，幸好不用我们继续深入了，官方给出了解释：

Called when the host view is attempting to determine if an item's position has changed. Returns POSITION\_UNCHANGED if the position of the given item has not changed or POSITION\_NONE if the item is no longer present in the adapter.The default implementation assumes that items will never change position and always returns POSITION\_UNCHANGED.

**大致的意思是：**\
&#x20;如果 Item 的位置如果没有发生变化，则返回 POSITION\_UNCHANGED。如果返回了 POSITION\_NONE，表示该位置的 Item 已经不存在了。默认的实现是假设 Item 的位置永远不会发生变化，而返回 POSITION\_UNCHANGED。（参考自：[追溯源码解决android疑难有关问题1-Viewpager之notifyDataSetChanged无刷新](https://link.jianshu.com/?t=http://www.educity.cn/wenda/179816.html)）

上面在源码里面跟了一大圈是不是还是感觉没有明朗，因为还有一个很关键的类 —— PagerAdapter 没有介绍，再给点耐心，继续。

**6、PagerAdapter 的工作流程**\
&#x20;其实就是 PagerAdapter 中方法的执行顺序，来看看 [Leo8573](https://link.jianshu.com/?t=http://my.csdn.net/leo8573) 的分析（个人感觉基本说到位了，所以直接拷过来了）：

PagerAdapter 作为 ViewPager 的适配器，无论 ViewPager 有多少页，PagerAdapter 在初始化时也只初始化开始的2个 View，即调用2次instantiateItem 方法。而接下来每当 ViewPager 滑动时，PagerAdapter 都会调用 destroyItem 方法将距离该页2个步幅以上的那个 View 销毁，以此保证 PagerAdapter 最多只管辖3个 View，且当前 View 是3个中的中间一个，如果当前 View 缺少两边的 View，那么就 instantiateItem，如里有超过2个步幅的就 destroyItem。

简易图示：

```
                 *
   ------+---+---+---+------
     ... 0 | 1 | 2 | 3 | 4 ...
   ------+---+---+---+------
```

当前 View 为2号 View，所以 PagerAdapter 管辖1、2、3三个 View，接下来向左滑动-->

```
                 *
   ------+---+---+---+------
     ... 1 | 2 | 3 | 4 | 5 ...
   ------+---+---+---+------
```

滑动后，当前 View 变为3号 View，PagerAdapter 会 destroyItem 0号View，instantiateItem 5号 View，所以 PagerAdapter 管辖2、3、4三个 View。（参考自：[关于ViewPager的数据更新问题小结](https://link.jianshu.com/?t=http://blog.csdn.net/leo8573/article/details/7893841)）

**总结一下：** Viewpager 的刷新过程是这样的，在每次调用 PagerAdapter 的 notifyDataSetChanged() 方法时，都会激活 getItemPosition(Object object) 方法，该方法会遍历 ViewPager 的所有 Item（由缓存的 Item 数量决定，默认为当前页和其左右加起来共3页，这个可以自行设定，但是至少会缓存2页），为每个 Item 返回一个状态值（POSITION\_NONE/POSITION\_UNCHANGED），如果是 POSITION\_NONE，那么该 Item 会被 destroyItem(ViewGroup container, int position, Object object) 方法 remove 掉，然后重新加载，如果是 POSITION\_UNCHANGED，就不会重新加载，默认是 POSITION\_UNCHANGED，所以如果不重写 getItemPosition(Object object)，修改返回值，就无法看到 notifyDataSetChanged() 的刷新效果。

### 最简单的解决方案

那就是直接一刀切：重写 PagerAdapter 的 getItemPosition(Object object) 方法，将返回值固定为 POSITION\_NONE。

先看看效果：

!\[最简单解决方案示例]\([http://upload-images.jianshu.io/upload\_images/1233754-0071612440ec3200.gif?imageMogr2/auto-orient/strip](https://link.jianshu.com/?t=http://upload-images.jianshu.io/upload\_images/1233754-0071612440ec3200.gif?imageMogr2/auto-orient/strip) ”最简单解决方案示例“)

上代码（PagerAdapterActivity.java）：

```
@Override
public int getItemPosition(Object object) {
    // 最简单解决 notifyDataSetChanged() 页面不刷新问题的方法
    return POSITION_NONE;
}
```

**该方案的缺点：**有个很明显的缺陷，那就是会刷新所有的 Item，这将导致系统资源的浪费，所以这种方式不适合数据量较大的场景。

**注意：**\
&#x20;这种方式还有一个需要注意的地方，就是重写 destoryItem() 方法：

```
@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    // 把 Object 强转为 View，然后将 view 从 ViewGroup 中清除
    container.removeView((View) object);
}
```

### 最简方案的优化

这里提供一个思路，毕竟场景太多，相信大家理解了思路要实现就很简单了，闲话不多说。

**思路：**在 instantiateItem() 方法中给每个 View 添加 tag（使用 setTag() 方法），然后在 getItemPosition() 方法中通过 View.getTag() 来判断是否是需要刷新的页面，是就返回 POSITION\_NONE，否就返回 POSITION\_UNCHANGED。 （参考自：[ViewPager刷新单个页面的方法](https://link.jianshu.com/?t=http://lovelease.iteye.com/blog/2107296)）

**注意：**这里有一点要注意的是，当清空数据源的时候需要返回 POSITION\_NONE，可用如下代码：

```
if (mDataList != null && mDataList.size()==0) {
    return POSITION_NONE;
}
```

关于 PagerAdapter 的介绍就到这里了，虽然 FragmentPagerAdapter 与 FragmentStatePagerAdapter 都是继承自 PagerAdapter。但是，这两个是专门为以 Fragment 为 Item 的 ViewPager 所准备的，所以有其特殊性。且看下面的介绍。

## 三、FragmentPagerAdapter

### 简介

上面通过使 getItemPosition() 方法返回 POSITION\_NONE 到达数据源变化（也就是调用 notifyDataSetChanged()）时，刷新视图的目的。但是当我们使用 Fragment 作为 ViewPager 的 Item 时，就需要多考虑一些了，而且一般是使用 FragmentPagerAdapter 或者 FragmentStatePagerAdapter。

这里不展开讨论 FragmentPagerAdapter 与 FragmentStatePagerAdapter 的异同和使用场景了，感兴趣的可以看看这篇文章：[FragmentPagerAdapter与FragmentStatePagerAdapter区别](https://link.jianshu.com/?t=http://www.cnblogs.com/lianghui66/p/3607091.html)。

下面先来看看使用 FragmentPagerAdapter 时，如何在数据源发生变化时，刷新 Fragment 或者动态改变 Items 的数量。

### 方案：清除 FragmentManager 中缓存的 Fragment

先看效果：

![](https://upload-images.jianshu.io/upload\_images/1233754-2116ad752d0ea8f0.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

实现上图效果的关键代码：\
&#x20;**1、FPagerAdapter1Activity.java**

```
private void refresh() {
    if (checkData()) return;
    mDataList.set(0, 7); // 修改数据源
    mPagerAdapter.updateData(mDataList); // 通知 Adapter 更新
}

private void add() {
    mDataList.add(7);
    mPagerAdapter.updateData(mDataList);
}

private void delete() {
    if (checkData()) return;
    mDataList.remove(0);
    mPagerAdapter.updateData(mDataList);
}

private void clear() {
    if (checkData()) return;
    mDataList.clear();
    mPagerAdapter.updateData(mDataList);
}
```

**2、FPagerAdapter1.java**

```
public class FPagerAdapter1 extends FragmentPagerAdapter {

    private ArrayList<Fragment> mFragmentList;
    private FragmentManager mFragmentManager;
    
    public FPagerAdapter1(FragmentManager fm, List<Integer> types) {
        super(fm);
        this.mFragmentManager = fm;
        mFragmentList = new ArrayList<>();
        for (int i = 0, size = types.size(); i < size; i++) {
            mFragmentList.add(FragmentTest.instance(i));
        }
        setFragments(mFragmentList);
    }

    public void updateData(List<Integer> dataList) {
        ArrayList<Fragment> fragments = new ArrayList<>();
        for (int i = 0, size = dataList.size(); i < size; i++) {
            Log.e("FPagerAdapter1", dataList.get(i).toString());
            fragments.add(FragmentTest.instance(dataList.get(i)));
        }
        setFragments(fragments);
    }

    private void setFragments(ArrayList<Fragment> mFragmentList) {
        if(this.mFragmentList != null){
            FragmentTransaction fragmentTransaction = mFragmentManager.beginTransaction();
            for(Fragment f:this.mFragmentList){
                fragmentTransaction.remove(f);
            }
            fragmentTransaction.commit();
            mFragmentManager.executePendingTransactions();
        }
        this.mFragmentList = mFragmentList;
        notifyDataSetChanged();
    }

    @Override
    public int getCount() {
        return this.mFragmentList.size();
    }
    
    public int getItemPosition(Object object) {
        return POSITION_NONE;
    }

    @Override
    public Fragment getItem(int position) {
        return mFragmentList.get(position);
    }
}
```

**3、思路分析**\
&#x20;上面的代码思路很简单，就是当数据源发生变化时，先将 FragmentManger 里面所有缓存的 Fragment 全部清除，然后重新创建，这样达到刷新视图的目的。

但是，这样做有一个缺点，那就是会造成不必要的浪费，会影响性能。还有就是必须使用一个 List 缓存所有的 Fragment，这也得占用不少内存...

思路挺简单，这里不再赘述，那看看有没有什么可以优化的。

### 优化：通过 Tag 获取缓存的 Fragment

先看效果：

![](https://upload-images.jianshu.io/upload\_images/1233754-716892b521db8b1e.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

从上面的动图上可以看到，更新某一个 Fragment 没有问题，清空数据源的时候也没有，添加当然也没什么问题；请注意删除的效果，虽然，目的 Fragment 确实从 ViewPager 中移除了，但是滑动后面的页面会发现出现了数据错乱。

分析一下**优化的思路**：

先来了解 FragmentPagerAdapter 中是如何管理 Fragment 的，这里涉及到 FragmentPagerAdapter 中的 instantiateItem() 方法：

```
@Override
public Object instantiateItem(ViewGroup container, int position) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}
```

从源码中可以看到在从 FragmentManager 中取出 Fragment 时调用了 findFragmentByTag() 方法，而这个 Tag 是由 makeFragmentName() 方法生成的。继续往下可以看到每一个 Fragment 都打上了一个标签（在 mCurTransaction.add() 方法中）。

也就是说是 FragmentManager 通过 Tag 找相应的 Fragment，从而达到缓存 Fragment 的目的。如果可以找到，就不会创建新的 Fragment，Fragment 的 onCreate()、onCreateView() 等方法都不会再次调用。

那**优化的思路**就有了：\
&#x20;**首先，**需要缓存所有 Fragment 的 Tag，代码如下：

```
private List<String> mTagList; // 用来存放所有的 Tag

// 生成 Tag
// 直接从 FragmentPageAdapter 源码里拷贝 Fragment 生成 Tag 的方法
private String makeFragmentName(int viewId, int index) {
    return "android:switcher:" + viewId + ":" + index;
}

// 将 Tag 缓存到 List 中
@Override
public Object instantiateItem(ViewGroup container, int position) {
    mTagList.add(position, makeFragmentName(container.getId(),
            (int) getItemId(position)));
    return super.instantiateItem(container, position);
}
```

**其次，**在更新 Fragment 时，使用相应的 Tag 去 FragmentManamager 中找相应的 Fragment，如果存在，就直接更新，代码如下：

```
public void update(int position, String str) {
    Fragment fragment = mFragmentManager.findFragmentByTag(mTagList.get(position));
    if (fragment == null) return;
    if (fragment instanceof FragmentTest) {
        ((FragmentTest)fragment).update(str);
    }
    notifyDataSetChanged();
}
```

该方法需要自行在 Fragment 中提供。

**最后，**对于动态改变 ViewPager 中 Fragment 的数量，如果是添加，那没什么要注意的；但是删除有点棘手。

在上面的动态上看到，删除一个 Fragment 后会出现混乱，这里没有进一步去研究了，这里仅提供一个示例供参考（这个示例代码有问题，仅供参考）

```
public void remove(int position) {
    mDataList.remove(position);
    isDataSetChange = true;
    Fragment fragment = mFragmentManager.findFragmentByTag(mTagList.get(position));
    mTagList.remove(position);
    if (fragment == null) {
        notifyDataSetChanged();
        return;
    }
    FragmentTransaction fragmentTransaction = mFragmentManager.beginTransaction();
    fragmentTransaction.remove(fragment);
    fragmentTransaction.commit();
    mFragmentManager.executePendingTransactions();
    notifyDataSetChanged();
}
```

**注意：**\
&#x20;这个”优化“示例，仅仅适用于在只需要更新某个 Fragment 的场景，关于动态删除 Fragment，该”优化“方案并不适用，也不推荐使用。

## 四、FragmentStatePagerAdapter

先看效果：

![](https://upload-images.jianshu.io/upload\_images/1233754-cb6bf82c7debda0e.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

### 简介

[FragmentStatePagerAdapter](https://link.jianshu.com/?t=http://developer.android.com/reference/android/support/v4/app/FragmentStatePagerAdapter.html) 与 FragmentPagerAdapter 类似，这两个类都继承自 PagerAdapter。但是，和 FragmentPagerAdapter 不一样的是，FragmentStatePagerAdapter 只保留当前页面，当页面离开视线后，就会被消除，释放其资源；而在页面需要显示时，生成新的页面(这和 ListView 的实现一样)。这种方式的好处就是当拥有大量的页面时，不必在内存中占用大量的内存。（参考自：[FragmentPagerAdapter与FragmentStatePagerAdapter区别](https://link.jianshu.com/?t=http://www.cnblogs.com/lianghui66/p/3607091.html)）

FragmentStatePagerAdapter 的实现与 FragmentPagerAdapter 有很大区别，如果照搬上述 FragmentPagerAdapter 刷新数据的方式，你会发现没有什么问题（可以使用 FPagerAdapter11.java 测试）。

### 另一种思路

但是，我在项目中实际应用的时候（Fragment 比较复杂，里面有网络任务等）出现了 IllegalStateException，发生在 ”fragmentTransaction.remove(f);“ 时。当时找了一些文章没有解决该问题，考虑到项目中的 Fragment 里面逻辑过多，就换思路，没有在这个上面继续深究了。

如果，你也是这样使用 FragmentStatePagerAdapter 来动态改变 ViewPager 中 Fragment，并且在 remove Fragment 时遇到了 IllegalStateException。那么，你可以考虑使用下面的方式，先看代码（FSPagerAdapter .java）：

```
public class FSPagerAdapter extends FragmentStatePagerAdapter {

    private ArrayList<Fragment> mFragmentList;

    public FSPagerAdapter(FragmentManager fm, List<Integer> types) {
        super(fm);
        updateData(types);
    }

    public void updateData(List<Integer> dataList) {
        ArrayList<Fragment> fragments = new ArrayList<>();
        for (int i = 0, size = dataList.size(); i < size; i++) {
            Log.e("FPagerAdapter1", dataList.get(i).toString());
            fragments.add(FragmentTest.instance(dataList.get(i)));
        }
        setFragmentList(fragments);
    }

    private void setFragmentList(ArrayList<Fragment> fragmentList) {
        if(this.mFragmentList != null){
            mFragmentList.clear();
        }
        this.mFragmentList = fragmentList;
        notifyDataSetChanged();
    }

    @Override
    public int getCount() {
        return this.mFragmentList.size();
    }
    
    public int getItemPosition(Object object) {
        return POSITION_NONE;
    }

    @Override
    public Fragment getItem(int position) {
        return mFragmentList.get(position);
    }
}
```

对应的测试 Activity 见 **FSPagerAdapterActivity.java**。

上面的代码挺简单，稍微解释一下实现思路：\
&#x20;**1、缓存所有的 Fragment**\
&#x20;使用一个 List 将数据源对应的 Fragment 都缓存起来

**2、更新数据源，刷新 Fragment**\
&#x20;当有数据源更新的时候，从 List 中取出相应的 Fragment，然后刷新 Adapter

**3、删除数据时，删除 List 中对应的 Fragment**\
&#x20;当数据源中删除某项时，将 List 中对应的 Fragment 也删除，然后刷新 Adapter

## 小结

关于 ViewPager 数据源刷新比较麻烦的地方是从数据源中删除数据的情况，这和 ViewPager 的实现方式有关，我们在解决该问题的时候要分具体情况来采取不同的方案。

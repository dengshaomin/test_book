# Transiton过渡动画

[官方文档](https://developer.android.com/training/transitions?hl=zh-cn)

在Android 4.4 Transition 就已经引入了，但在Android 5.0(API 21)之后，Transition 被更多的应用起来。为了对Transition有一个大概的了解，我们通过：Scene Transition(场景过渡动画)、Activity过渡动画、Shared Element Transition(共享元素过渡动画)这三个方面来做一个简单的了解。

> **尽量在调用此系列API时，先判断SDK版本符合否则使用普通跳转**

## 介绍

场景过渡动画中有两个特别关键概念：（场景），Transition（过渡）。

* Scene：Scene代表一个场景。Scene保存了一个视图层级结构，包括它所有的views以及views的状态，通常由getSceneForLayout (ViewGroup sceneRoot,int layoutId,Context context)获取Scene实例。Transition框架可以实现在starting scene和ending scene之间执行动画。而且大多数情况下，我们不需要创建starting scene，因为starting scene通常由当前UI状态决定，我们只需要创建ending scene。
* Transition：Transiton则是用来设置过渡动画效果用的。而且系统给提供了一些非常使用的Transtion动画效果，如下表所示:

| 系统Transition         | 解释                                                                                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ChangeBounds         | 检测View的位置边界创建移动和缩放动画(关注布局边界的变化)                                                                                                                          |
| ChangeTransform      | 检测View的scale和rotation创建缩放和旋转动画(关注scale和ratation的变化)                                                                                                      |
| ChangeClipBounds     | 检测View的剪切区域的位置边界，和ChangeBounds类似。不过ChangeBounds针对的是view而ChangeClipBounds针对的是view的剪切区域setClipBound(Rect rect) 中的rect(关注的是setClipBounds(Rect rect)rect的变化) |
| ChangeImageTransform | 检测ImageView的ScaleType，并创建相应动画(关注的是ImageView的scaleType)                                                                                                   |
| Fade                 | 根据View的visibility状态的的不同创建淡入淡动画,调整的是透明度(关注的是View的visibility的状态)                                                                                           |
| Slide                | 根据View的visibility状态的的不同创建滑动动画(关注的是View的visibility的状态)                                                                                                    |
| Explode              | 根据View的visibility状态的的不同创建分解动画(关注的是View的visibility的状态)                                                                                                    |
| AutoTransition       | 默认动画，ChangeBounds、Fade动画的集合                                                                                                                              |

要想实现一个场景过渡动画，至少需要一个transition实例和一个ending scene实例。并通过TransitionManager执行过渡动画。TransitionManager执行动画有两种方式：TransitionManager.go()、beginDelayedTransition()。下面简单来介绍下这两种开启场景动画的方式。

## **TransitionManager.go()**

先上个效果：

![](https://im4.ezgif.com/tmp/ezgif-4-f8a5fe8bd4af.gif)

总共两个Scene，先通过 `Scene.getSceneForLayout` 创建【开始场景】、【结束场景】，然后通过`TransitonManager.go`启动场景切换

### 开始场景

{% tabs %}
{% tab title="source_scene.xml" %}
```markup
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_container"
    android:layout_width="match_parent"
    android:padding="20dp"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/image0"
        android:layout_width="150dp"
        android:background="@mipmap/circle0"
        android:gravity="center"
        android:layout_height="150dp" />

    <TextView
        android:layout_alignParentRight="true"
        android:id="@+id/image1"
        android:layout_width="80dp"
        android:layout_height="80dp"
        android:background="@mipmap/circle1" />
</RelativeLayout>
```
{% endtab %}
{% endtabs %}

### 结束场景

{% tabs %}
{% tab title="target_scene.xml" %}
```markup
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_container"
    android:layout_width="match_parent"
    android:padding="20dp"
    android:layout_height="match_parent">

    <ImageView
        android:id="@+id/image0"
        android:layout_width="80dp"
        android:background="@mipmap/circle0"
        android:layout_alignParentRight="true"
        android:gravity="center"
        android:layout_height="80dp" />

    <TextView
        android:id="@+id/image1"
        android:layout_width="150dp"
        android:layout_height="150dp"
        android:background="@mipmap/circle1" />
</RelativeLayout>
```
{% endtab %}
{% endtabs %}

### Activity

{% tabs %}
{% tab title="activity_transiton_manager_go.xml" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".TransitionManagerGoActivity">

    <TextView
        android:layout_width="match_parent"
        android:id="@+id/switch_sence"
        android:background="@color/purple_200"
        android:gravity="center"
        android:text="switch"
        android:padding="10dp"
        android:layout_margin="10dp"
        android:layout_height="wrap_content"></TextView>

    <FrameLayout
        android:id="@+id/scene_root"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content">
        <include layout="@layout/source_scene" />
    </FrameLayout>
</LinearLayout>
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="TransitonManagerGoActivity.kt" %}
```kotlin
class TransitionManagerGoActivity : AppCompatActivity() {
    var  switch = false
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_transition_manager_go)

        val sceneRoot: ViewGroup = findViewById(R.id.scene_root)
        val sourceScene = Scene.getSceneForLayout(sceneRoot, R.layout.source_scene, this)
        val targetScene = Scene.getSceneForLayout(sceneRoot, R.layout.target_scene, this)
        switch_sence.setOnClickListener {
            switch = !switch
            TransitionManager.go(if(switch) targetScene else sourceScene)
            targetScene.sceneRoot.findViewById<View>(R.id.image0).setOnClickListener{
                Toast.makeText(this@TransitionManagerGoActivity, "image0", Toast.LENGTH_SHORT).show()
            }
            targetScene.sceneRoot.findViewById<View>(R.id.image1).setOnClickListener{
                Toast.makeText(this@TransitionManagerGoActivity, "image1", Toast.LENGTH_SHORT).show()
            }
        }

        image0.setOnClickListener {
            Toast.makeText(this@TransitionManagerGoActivity, "image0", Toast.LENGTH_SHORT).show()
        }
        image1.setOnClickListener {
            Toast.makeText(this@TransitionManagerGoActivity, "image1", Toast.LENGTH_SHORT).show()
        }
    }
}
```
{% endtab %}
{% endtabs %}

这两个场景布局，除了内部元素的位置、大小不一致，其他都是相同的。这里差不多也联想到内部实现的原理：rootScene是一致的，通过索引内部元素一一做匹配，然后根据元素的改变启动相应的动画。前面提到了系统的各种Transition，看上面代码中我们也没配置Transiton那是怎么实现以上效果呢 ？直接看源码：

调用go时会给一个默认的Transtion，这是一个AutoTransition

```kotlin
public static void go(@NonNull Scene scene) {
    changeScene(scene, sDefaultTransition);
}
```

 AutoTransition在初始化的时候配置了一些组合Transition，所以能达到以上视频中的效果

```kotlin
public class AutoTransition extends TransitionSet {

    /**
     * Constructs an AutoTransition object, which is a TransitionSet which
     * first fades out disappearing targets, then moves and resizes existing
     * targets, and finally fades in appearing targets.
     */
    public AutoTransition() {
        init();
    }

    public AutoTransition(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setOrdering(ORDERING_SEQUENTIAL);
        addTransition(new Fade(Fade.OUT))
                .addTransition(new ChangeBounds())
                .addTransition(new Fade(Fade.IN));
    }

}
```

###  点击事件失效的问题

在Actiivty中对image0和image1设置了点击事件之后为什么还要在调用TransitionManager.go之后再次设置点击事件呢？

```kotlin
private static void changeScene(Scene scene, Transition transition) {
        final ViewGroup sceneRoot = scene.getSceneRoot();

        if (!sPendingTransitions.contains(sceneRoot)) {
            Scene oldScene = Scene.getCurrentScene(sceneRoot);
            if (transition == null) {
                // Notify old scene that it is being exited
                if (oldScene != null) {
                    oldScene.exit();
                }
                scene.enter();
            } else {
                sPendingTransitions.add(sceneRoot);

                Transition transitionClone = transition.clone();
                transitionClone.setSceneRoot(sceneRoot);

                if (oldScene != null && oldScene.isCreatedFromLayoutResource()) {
                    transitionClone.setCanRemoveViews(true);
                }
                sceneChangeSetup(sceneRoot, transitionClone);
                scene.enter();
                sceneChangeRunTransition(sceneRoot, transitionClone);
            }
        }
    }
```

先调用了oldScene.exit 然后再调用了scene.enter()完成退出和进入的动画，进入enter函数

```kotlin
public void enter() {
        // Apply layout change, if any
        if (mLayoutId > 0 || mLayout != null) {
            // empty out parent container before adding to it
            getSceneRoot().removeAllViews();

            if (mLayoutId > 0) {
                LayoutInflater.from(mContext).inflate(mLayoutId, mSceneRoot);
            } else {
                mSceneRoot.addView(mLayout);
            }
        }

        // Notify next scene that it is entering. Subclasses may override to configure scene.
        if (mEnterAction != null) {
            mEnterAction.run();
        }

        setCurrentScene(mSceneRoot, this);
    }
```

点击事件失效的原因就是先对oldScene的布局进行了移除然后将scene的布局添加进来，所以点击事件需要重新绑定。

## TransitionManager.beginDelayedTransition()

在不涉及到布局交换的时候可用此函数进行单布局的动画，文章底部Demo会对所有系统Transition给出示例

## Activity过渡使用Transiton

![](https://im.ezgif.com/tmp/ezgif-1-cfe3af282f30.gif)



### 1：启用窗口内容转换

*   通过Theme启动

    ```
    <style name="Theme.Transition" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <item name="android:windowContentTransitions">true</item>
        <!--在前一个动画完全执行完之后再进行下一个动画，否则进入和退出动画可能会有部分重叠-->
        <item name="android:windowAllowEnterTransitionOverlap">false</item>
        <item name="android:windowAllowReturnTransitionOverlap">false</item>
    </style>
    ```
*   通过代码启动

    ```
    window.requestFeature(Window.FEATURE_CONTENT_TRANSITIONS)
    ```

###  2设置当前Activity动画 

设置退出和进入动画都为Explode，并且通过makeSceneTransitionAnimation启动Activity

```kotlin
if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP) {
    window.enterTransition = Explode()
    window.exitTransition = Explode()
    startActivity(
        Intent(this@MainActivity, TransitionActivity0::class.java),
        ActivityOptions.makeSceneTransitionAnimation(this@MainActivity).toBundle()
    )
}
```

###  3.设置目标Activity动画

```kotlin
if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP) {
            window.enterTransition = TransitionInflater.from(this).inflateTransition(R.transition.activity_transition0_enter)
            window.exitTransition = TransitionInflater.from(this).inflateTransition(R.transition.activity_transition0_out)
        }
```

 通过XML配置进入合并、退出撕裂效果的Transition，如果不单独指定退出时的动画，则使用逆向动画效果退出当前Activity

{% tabs %}
{% tab title="enter.xml" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
    <slide
        android:duration="1000"
        android:interpolator="@android:interpolator/fast_out_slow_in"
        android:slideEdge="top">
        <targets>
            <!--指定要执行动画的布局id,如果不指定则默认为根布局下的所有子布局-->
            <target android:targetId="@+id/top_image"/>
            <!--指定不需要执行动画的布局id，这里排除状态栏和导航栏-->
            <target android:excludeId="@android:id/statusBarBackground"/>
            <target android:excludeId="@android:id/navigationBarBackground"/>
        </targets>
    </slide>
    <slide
        android:duration="1000"
        android:interpolator="@android:interpolator/fast_out_slow_in"
        android:slideEdge="bottom">
        <targets>
            <target android:targetId="@+id/bottom_content"/>
        </targets>
    </slide>
</transitionSet>
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="out.xml" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android">
    <slide
        android:duration="1000"
        android:interpolator="@android:interpolator/fast_out_slow_in"
        android:slideEdge="right">
        <targets>
            <target android:targetId="@+id/bottom_content"/>
        </targets>
    </slide>
    <slide
        android:duration="1000"
        android:interpolator="@android:interpolator/fast_out_slow_in"
        android:slideEdge="left">
        <targets>
            <target android:targetId="@+id/top_image"/>
        </targets>
    </slide>
</transitionSet>
```
{% endtab %}
{% endtabs %}

 window下各种transiton的区别：

* Exit Transition:可以理解为activity进入后台的过渡动画
* Enter Transition:可以理解为创建activity并显示时的过渡动画
* Return Transition:可以理解为销毁activity时的过渡动画
* Reenter Transition:可以理解为activity从后台进入前台时的过渡动画

## SharedElementEnterTransition

![](https://im.ezgif.com/tmp/ezgif-1-28909af43b17.gif)

### 1.配置shareElements

如果是单个共享元素可以直接指定shareElement、shareElementName

```kotlin
var shareView = findViewById<View>(R.id.share_image)
var options = ActivityOptionsCompat.makeSceneTransitionAnimation(this@MainActivity,shareView,"share_image")
```

 如果是多个共享元素则可通过\`Pare\<View,String>指定

```kotlin
var shareView = findViewById<View>(R.id.share_image)
var share0 = Pair(shareView,"share_image")
var options = ActivityOptionsCompat.makeSceneTransitionAnimation(this@MainActivity,shareView,"share_image")
```

> 注意：共享元素的 ID不必相等，但是shareElementName必须一致

### 2.配置被启动的Activity

设置shareElement的transitionName，与启动Activity中Pair中的name必须保持一致

```markup
<ImageView
    android:id="@+id/top_image"
    android:transitionName="share_image"
    android:layout_width="match_parent"
    android:src="@mipmap/rect0"
    android:layout_height="300dp"></ImageView>
```

 设置共享元素和非共享元素的进入和退出动画

```kotlin
if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.LOLLIPOP) {
    //设置非shareElements之外的布局动画
    window.enterTransition =TransitionInflater.from(this).inflateTransition(R.transition.activity_transition0_enter)
    window.exitTransition = TransitionInflater.from(this).inflateTransition(R.transition.activity_transition0_out)
    //设置shareElements动画
    window.sharedElementEnterTransition =TransitionInflater.from(this).inflateTransition(R.transition.activity_share_transition_enter)
    window.sharedElementReturnTransition = TransitionInflater.from(this).inflateTransition(R.transition.activity_share_transition_enter)
}
```

{% tabs %}
{% tab title="share_enter.xml" %}
```markup
<?xml version="1.0" encoding="utf-8"?>
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000">
    <changeImageTransform>

    </changeImageTransform>
    <changeBounds></changeBounds>
</transitionSet>
```
{% endtab %}
{% endtabs %}

##  [本章Demo](https://github.com/dengshaomin/Transition)

## 相关文章

* [http://adolph.cc/15561207871979.html](http://adolph.cc/15561207871979.html)
* [https://www.jianshu.com/p/e497123652b5](https://www.jianshu.com/p/e497123652b5)
* [https://damonzh.github.io/2016/04/04/transition-framework/](https://damonzh.github.io/2016/04/04/transition-framework/)

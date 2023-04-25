# 琐碎知识点

### CopyOnWriteArrayList[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#copyonwritearraylist) <a href="#copyonwritearraylist" id="copyonwritearraylist"></a>

CopyOnWriteArrayList支持并发读写。

### RemoteCallbackList[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#remotecallbacklist) <a href="#remotecallbacklist" id="remotecallbacklist"></a>

RemoteCallbackList支持跨进程删除listener，原理是`ArrayMap<IBinder, Callback>`。IPC时Parcelable对象会序列化、反序列化，因此不会是同一个对象，但底层IBinder不会变。 它同时具有以下特点： 1. 客户端进程终止后，能够自动移除客户端注册的listener 2. RemoteCallbackList内部自动实现了线程同步功能，所以使用它来进行注册、取消注册时不需要做额外的线程同步工作。

### android:windowSoftInputMode[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#androidwindowsoftinputmode) <a href="#androidwindowsoftinputmode" id="androidwindowsoftinputmode"></a>

EditText禁止进入时弹出键盘：

```
android:windowSoftInputMode="stateHidden"
```

键盘弹出时禁止顶动View：

```
android:windowSoftInputMode="adjustPan"
```

### 常用Intent[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#intent) <a href="#intent" id="intent"></a>

跳转拨号盘

```
new Intent(Intent.ACTION_DIAL, Uri.parse("tel:12306"));
```

直接拨打电话

```
new Intent(Intent.ACTION_CALL, Uri.parse("tel:12306"));
```

发送短信

```
new Intent(Intent.ACTION_SENDTO, Uri.parse("smsto:12306"));
```

### Collections.sort[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#collectionssort) <a href="#collectionssort" id="collectionssort"></a>

Java Bean实体进行排序

```
Collections.sort(mRowsBeanList, new Number2zComparator());

private class Number2zComparator implements Comparator<T> {
    @Override
    public int compare(T o1, T o2) {
        boolean o1IsHot = o1.getHotFlag() == 1;
        boolean o2IsHot = o2.getHotFlag() == 1;
        String o1Letter = PinyinUtils.ccs2Pinyin(o1.getPlatformName());
        String o2Letter = PinyinUtils.ccs2Pinyin(o2.getPlatformName());

        if (o1IsHot && o2IsHot) {
            return o1Letter.compareToIgnoreCase(o2Letter);
        } else if (o1IsHot) {
            return -1;
        } else if (o2IsHot) {
            return 1;
        }

        return o1Letter.compareToIgnoreCase(o2Letter);
    }
}
```

o1与o2进行比较，如果o1要排在o2前面，返回-1;\
如果两者相等，返回0;\
如果o1排在o2后面，返回1。

### Gif加载[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#gif) <a href="#gif" id="gif"></a>

Gif加载\
在尝试Glide加载(不卡，但是图片错乱了)、帧动画实现(超级耗内存，30帧，每帧8k原图，耗内存大约20M)、android-gif-drawable库(Android O上非常卡顿，耗内存大约10M)加载失败之后，找到了一种新奇的思路：每隔一段时间调用`setBackgroundResource`，内存消耗基本不计。

### 代码设置drawable[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#drawable) <a href="#drawable" id="drawable"></a>

`setCompoundDrawablesWithIntrinsicBounds`

### response.body().string()只能调用一次[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#responsebodystring) <a href="#responsebodystring" id="responsebodystring"></a>

OkHttp访问网络成功的回调中，`Response response`的`response.body().string()`只能调用一次，否则

```
E/AndroidRuntime: FATAL EXCEPTION: OkHttp Dispatcher
                  Process: yorek.demoandtest, PID: 24671
                  java.lang.IllegalStateException: closed
```

因为这是一个流，使用过后就被关闭了：

```
  public final String string() throws IOException {
    BufferedSource source = source();
    try {
      Charset charset = Util.bomAwareCharset(source, charset());
      return source.readString(charset);
    } finally {
      Util.closeQuietly(source);
    }
  }
```

### Glide加载圆形图片[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#glide) <a href="#glide" id="glide"></a>

```
Glide.with(mContext)
    .load(item.platformLogo)
    .bitmapTransform(CropCircleTransformation(mContext))
    .into(iv_logo)
```

其中，`.bitmapTransform(CropCircleTransformation(mContext))`是其中的重点， `CropCircleTransformation`使用了`jp.wasabeef:glide-transformations:2.0.2`这个lib

```
compile 'jp.wasabeef:glide-transformations:2.0.2'
```

Glide v4已经内置了圆形、圆角等常用的Trasform了。上面是Glide v3时的做法。

### 好人好信Tab3下拉冲突问题[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#tab3) <a href="#tab3" id="tab3"></a>

`SwipeRefreshLayout`与`CollapsingToolbarLayout`和充满`RecyclerView`的`ViewPager`联用出现的滑动冲突问题

解决办法是自定义`SwipeRefreshLayout`，在`AppBarLayout`元素没有到顶时允许child向上滑，到顶后不允许滑动，这样就触发了下拉刷新。

```
public class CreditFragmentSwipeRefreshLayout extends SwipeRefreshLayout {
    private AppBarLayout targetView;

    public CreditFragmentSwipeRefreshLayout(Context context) {
        super(context);
    }

    public CreditFragmentSwipeRefreshLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onFinishInflate() {
        super.onFinishInflate();
        targetView = (AppBarLayout) findViewById(R.id.abl);
    }

    @Override
    public boolean canChildScrollUp() {
        if (targetView != null) {
            return targetView.getTop() != 0;
        } else {
            return super.canChildScrollUp();
        }
    }
}
```

### Android中字体的加载[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#android) <a href="#android" id="android"></a>

在`app/src/main/assets`下面放置字体文件，然后使用下面的代码加载\


```
class KTypeface {
    companion object {
        val DIN_PRO_MIDIUM =
            Typeface.createFromAsset(MyApp.getInstance().assets, "DINPro-Medium.otf")
        val ROBOTO_MIDIUM =
            Typeface.createFromAsset(MyApp.getInstance().assets, "Roboto-Medium.ttf")
    }
}
```

这么使用：\


```
tvMyScore.typeface = KTypeface.DIN_PRO_MIDIUM
```

Android中字体最好加载一次之后缓存起来，因为每次加载需要消耗时间。

### 判断应用通知权限是否打开[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_1) <a href="#_1" id="_1"></a>

```
/**
 * 只能检查KITKAT及以上的系统，以下会返回true
 */
public static boolean isNotificationEnabled(Context context) {
    NotificationManagerCompat notificationManagerCompat = NotificationManagerCompat.from(context);
    return notificationManagerCompat.areNotificationsEnabled();
}
```

### ViewPager实现画廊效果[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#viewpager) <a href="#viewpager" id="viewpager"></a>

实现原理很简单

1. `ViewPager`的父布局需要设置`clipChildren = false`，同时最好设置背景色(pageMargin的部分需要背景色填充)
2. `ViewPager`也需要设置`clipChildren = false`
3. `ViewPager`宽度为`MATCH_PARENT`，但需要为其设置`leftMargin`和`rightMargin`，这样才有画廊的效果
4. 两张卡片之间的间距由`ViewPager`的`pageMargin`控制

`ViewPager`里面的子元素距离边框最好不要有`padding`以及`margin`，这样不利于控制效果

### 全面屏Splash以及广告页适配指北[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#splash) <a href="#splash" id="splash"></a>

**`SplashActivity`适配**

1. 新建`drawable-xxhdpi-2016x1080`文件夹，里面放入为全面屏准备的开屏页(一般是1080x2160)
2. 在`SplashActivity`的主题中加入`<item name="android:windowBackground">@drawable/img_splash</item>`(所谓程序秒开主要指这个)

上述直接放图片的方式不太适用了，因为手机高宽比越来越大，这样总会被拉长的。如果图片底部是纯色的话，可以使用layer-list包裹一下图片，其他位置用纯色就好了。

**广告页适配(我家广告页就是与`SplashActivity` style一样的第二屏)**\
这个位置麻烦的地方就是需要显示从网络下载的图片，而且有多种尺寸，所以为了在全面屏上显示更好，我们需要加载一张1080x2160大小的图片然后显示。\
所以，我们会先判断是不是全面屏，如果是就下载1080x2160大小的图片，否则一律下载1080x1920大小的图片。

```
// 下载图片的判断逻辑
String screenType = "1080x1920";
Point point = HRScreenUtils.Companion.getScreenSize(this);
if (HRScreenUtils.Companion.isFullScreen(point.x, point.y)) {
    screenType = "1080x2160";
}
Call<FestivalImgRes> call = HttpHelper.getApiService().getFestivalHead("android", screenType);

// ------------------------------
// HRScreenUtils.kt
class HRScreenUtils {
    companion object {
        fun isFullScreen(width: Int, height: Int) : Boolean = (1.0f * height / width) >= 1.86f

        fun getScreenSize(context: Context): Point {
            val point = Point()
            val windowManager = context.getSystemService(Context.WINDOW_SERVICE) as WindowManager
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
                windowManager.defaultDisplay.getRealSize(point)
            } else {
                windowManager.defaultDisplay.getSize(point)
            }

            return point
        }
    }
}
```

同样，这里需要考虑一下手机高宽比越来越大的情况。

### 震动反馈[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_2) <a href="#_2" id="_2"></a>

不调用`Vibrator`实现震动反馈的两种方式 - 在`View`的`OnLongClickListener`中返回`true` - 调用`View#performHapticFeedback(HapticFeedbackConstants.KEYBOARD_TAP);`方法

> 震动反馈能否发生还取决与系统的震动反馈设置，通过`performHapticFeedback`重载方法实现震动反馈可以忽略系统设置。

### Android 5.0 平台共享元素动画[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#android-50) <a href="#android-50" id="android-50"></a>

1. 为两个`Activity`中需要进行动画的`View`取上相同的`transitionName`，若需要同时进行多个元素的动画，每个元素的`transitionName`都要不同
2. 跳转`Activity`时，
3. 若只有一个元素，可以调用`startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this, tv_right, getString(R.string.transition_name)).toBundle())`
4.  若多个元素，如下\


    ```
    ActivityOptions activityOptions = ActivityOptions.makeSceneTransitionAnimation(getActivity(),
          new Pair<>(view.findViewById(R.id.iv_folder_icon), getContext().getString(R.string.transition_task_icon)),
          new Pair<>(view.findViewById(R.id.tv_folder_progress_value), getContext().getString(R.string.transition_task_progress_value)),
          new Pair<>(view.findViewById(R.id.pb_folder_progress), getContext().getString(R.string.transition_task_progress)),
          new Pair<>(view.findViewById(R.id.tv_folder_count), getContext().getString(R.string.transition_task_count)));
    Intent intent = new Intent(getContext(), TodoFolderActivity.class);
    getContext().startActivity(intent, activityOptions.toBundle());
    ```

> 由B返回A时，一般情况都也会默认进行共享元素动画返回，特殊情况下可以调用`finishAfterTransition()`进行共享元素动画返回

共享元素动画执行的过程也有监听方法，`SharedElementCallback`里面方法比较多，具体可以看源码注释

* Activity#setEnterSharedElementCallback(SharedElementCallback)
* Activity#setExitSharedElementCallback(SharedElementCallback)
* Fragment#setEnterSharedElementCallback(SharedElementCallback)
* Fragment#setExitSharedElementCallback(SharedElementCallback)
* SharedElementCallback
* onSharedElementStart
* onSharedElementEnd
* onRejectSharedElements
* onMapSharedElements
* onSharedElementsArrived

### 向WebView注入本地JS并调用[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#webviewjs) <a href="#webviewjs" id="webviewjs"></a>

1.定义本地JS文件(test.js)，放到assets目录下

```
'use strict';

function test() {
  $(".fake-box input").val('1');
  angular.element(document.getElementById('pwd-input')).scope().$apply('verify_code = "123458"');
}
```

2.定义webview注入本地js文件的方法

```
private void injectScriptFile(WebView view, String scriptFile) {
    InputStream input;
    try {
        input = getAssets().open(scriptFile);
        byte[] buffer = new byte[input.available()];
        input.read(buffer);
        input.close();

        // String-ify the script byte-array using BASE64 encoding !!!
        String encoded = Base64.encodeToString(buffer, Base64.NO_WRAP);
        view.loadUrl("javascript:(function() {" +
                "var parent = document.getElementsByTagName('head').item(0);" +
                "var script = document.createElement('script');" +
                "script.type = 'text/javascript';" +
                // Tell the browser to BASE64-decode the string into your script !!!
                "script.innerHTML = window.atob('" + encoded + "');" +
                "parent.appendChild(script)" +
                "})()");
    } catch (IOException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
}
```

3.注入并调用

```
injectScriptFile(webView, "test.js");                   // 注入
webView.loadUrl("javascript:setTimeout(test(), 500)");  // 调用
```

### NavigationView菜单分割线[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#navigationview) <a href="#navigationview" id="navigationview"></a>

给`NavigationView`中的菜单添加分割线，只要给每个group添加id即可。

```
<?xml version="1.0" encoding="utf-8"?>
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:showIn="navigation_view">

    <group
        android:id="@+id/group_all"
        android:checkableBehavior="single">
        <item
            android:id="@+id/nav_all"
            android:icon="@drawable/ic_menu_lightbulb"
            android:title="@string/nav_menu_all"
            android:checked="true"/>
    </group>

    <item android:title="@string/nav_menu_labels">
        <menu>
            <group android:checkableBehavior="single">
                <item
                    android:id="@+id/nav_new_label11"
                    android:icon="@drawable/ic_menu_lightbulb"
                    android:title="测试1"/>
                <item
                    android:id="@+id/nav_new_label"
                    android:icon="@drawable/ic_menu_lightbulb"
                    android:title="@string/nav_menu_create_new_label" />
            </group>
        </menu>
    </item>

    <group
        android:id="@+id/group_collection"
        android:checkableBehavior="single">
        <item
            android:id="@+id/nav_archive"
            android:icon="@drawable/ic_menu_lightbulb"
            android:title="@string/nav_menu_archive" />
        <item
            android:id="@+id/nav_trash"
            android:icon="@drawable/ic_menu_lightbulb"
            android:title="@string/nav_menu_trash" />
    </group>

    <group
        android:id="@+id/group_settings"
        android:checkableBehavior="single">
        <item
            android:id="@+id/nav_settings"
            android:icon="@drawable/ic_menu_lightbulb"
            android:title="@string/nav_menu_settings" />
        <item
            android:id="@+id/nav_help"
            android:icon="@drawable/ic_menu_lightbulb"
            android:title="@string/nav_menu_help_and_feedback" />
    </group>
</menu>
```

### CardView背景色[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#cardview) <a href="#cardview" id="cardview"></a>

给`CardView`添加背景色要使用`app:cardBackgroundColor`，不然会导致`app:cardCornerRadius`不生效

### 从attr中提取资源id[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#attrid) <a href="#attrid" id="attrid"></a>

从attr中提取资源id，可以使用`TypedValue`

```
val typedValue = TypedValue()
theme.resolveAttribute(android.R.attr.colorControlNormal, typedValue, true)
val resourceId = typedValue.resourceId
```

### 跳微信扫一扫[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_3) <a href="#_3" id="_3"></a>

```
try {
    val intent = mActivity.packageManager.getLaunchIntentForPackage("com.tencent.mm")
    intent.putExtra("LauncherUI.From.Scaner.Shortcut", true)
    startActivity(intent)
} catch (e: Exception) {
    ToastUtils.showLongToast("无法跳转到微信，请检查您是否安装了微信！")
}
```

### RadioGroup互斥[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#radiogroup) <a href="#radiogroup" id="radiogroup"></a>

RadioGroup里面RadioButton最好都设置id,不然给其中一个设置为默认选中后，其他的RadioButton不会互斥。

### 不flatDir使用aar包[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#flatdiraar) <a href="#flatdiraar" id="flatdiraar"></a>

使用aar包不用flatDir，一句代码就好

```
implementation fileTree(dir: 'libs', include: ['*.jar', '*.aar'])
```

### 自定义资源目录[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_4) <a href="#_4" id="_4"></a>

```
android {
    sourceSets {
        main {
            java.srcDirs += 'src/main/kotlin'
            res.srcDirs = [
                    'src/main/res',
                    'src/main/res_overlay'
            ]
        }
    }
}
```

### 修改gradle输出包的名字[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#gradle) <a href="#gradle" id="gradle"></a>

```
// -----------------------------------------------------------------------------------------
// build script
android {
    applicationVariants.all { variant ->
        variant.outputs.all { output ->
            def newName
            def timeNow
            if ("true" == IS_JENKINS) {
                // Jenkins编译
                newName = APP_NAME + '-' + variant.buildType.name + '-' + BUILD_TIME + '.apk'
                outputFileName = new File("../../../../..", newName)
            } else {
                // Android Studio编译
                timeNow = new Date().format("yyyyMMddHHmm")
                newName = APP_NAME + "-v" + variant.versionName + '-' + variant.buildType.name + '-' + timeNow + '.apk'
                outputFileName = newName
            }
        }
    }
}
// -----------------------------------------------------------------------------------------
```

### gradle copy函数[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#gradle-copy) <a href="#gradle-copy" id="gradle-copy"></a>

```
android.libraryVariants.all {
    it.outputs.all {
        outputFileName = "hruilib-${version}-${it.name}.aar"
    }
}

project.afterEvaluate {
    android.libraryVariants.each {
        String variantName = it.name.capitalize()
        if (variantName == 'Release') {
            def assembleTask = project.tasks.getByName("assemble${variantName}")
            assembleTask.doLast {
                copy {
                    from('build/outputs/aar/')
                    into('../app/libs/')
                    include("*-release.aar")
                }
            }
        }
    }
}
```

### ScrollView始终显示scrollbar[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#scrollviewscrollbar) <a href="#scrollviewscrollbar" id="scrollviewscrollbar"></a>

```
android:scrollbars="vertical"
android:fadeScrollbars="false"
```

实践效果：当可以滑动的时候会显示scollbar，不能滑动的时候不会显示。

### AlertDialog消息换行[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#alertdialog) <a href="#alertdialog" id="alertdialog"></a>

AlertDialog中`\n`不换行，调用`create().show()`就可以了。

### accentColor[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#accentcolor) <a href="#accentcolor" id="accentcolor"></a>

基础库中UI的style最好不要加上accentColor属性，不然上层可能TextView等各种崩溃

### zip/unzip[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#zipunzip) <a href="#zipunzip" id="zipunzip"></a>

zip -q -r \\\<zip\_file\_name> \*

unzip \\\<zip\_file\_name> -d \\\<path>

### Mac调整Launcher行列数[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#maclauncher) <a href="#maclauncher" id="maclauncher"></a>

Mac调整Launcher一屏显示多少行、多少列，比如7行10列。

```
defaults write com.apple.dock springboard-rows -int 7
defaults write com.apple.dock springboard-columns -int 10
defaults write com.apple.dock ResetLaunchPad -bool TRUE;killall Dock
```

### OkHttp下载文件[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#okhttp) <a href="#okhttp" id="okhttp"></a>

```
@Suppress("DEPRECATION")
private var progressDialog: ProgressDialog? = null
private var apkFile: File? = null
private fun doDownloadApk(versionRes: VersionRes) {
    apkFile = File(getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS), "${BuildConfig.APPLICATION_ID}_${versionRes.version}.apk")
    // check file is exists or not
    if (apkFile?.exists() == true) {
        val deleted = apkFile?.delete()
        Logger.dTag(TAG_APP, "[SplashActivity] [doDownloadApk] apkFile exists, delete=$deleted")
    }

    // check download url is illegal
    val apkUrl = versionRes.channelUrl
    if (apkUrl.isNullOrEmpty()) {
        Logger.eTag(TAG_APP, "[SplashActivity] [doDownloadApk] apkUrl isNullOrEmpty >>>>>> ")
        viewModel.checkSwitchEnable()
        return
    }

    // show progress dialog
    @Suppress("DEPRECATION")
    progressDialog = ProgressDialog(this, R.style.HddAlertDialog).apply {
        setMessage(getString(R.string.download_apk_title))
        setProgressStyle(ProgressDialog.STYLE_HORIZONTAL)
        setCancelable(false)
        setCanceledOnTouchOutside(false)
        show()
    }

    // begin download
    Schedulers.newThread().createWorker().schedule {
        try {
            Logger.dTag(TAG_APP, "[SplashActivity] [doDownloadApk] download begin")
            val request = Request.Builder()
                .url(apkUrl)
                .build()
            val response = OkHttpClient.Builder().build().newCall(request).execute()

            // check response body
            val responseBody = response.body()
            if (responseBody == null) {
                viewModel.checkSwitchEnable()
                return@schedule
            }
            val inputStream = responseBody.byteStream()
            // get file size
            val contentLength = responseBody.contentLength()
            // set progress dialog max progress (shown in the bottom-right corner)
            // percent will be calculated by system
            progressDialog?.max = contentLength.toInt()
            Logger.dTag(TAG_APP, "[SplashActivity] [doDownloadApk] contentLength=$contentLength")

            // ready write files
            val fileOutputStream = FileOutputStream(apkFile)
            val bis = BufferedInputStream(inputStream)
            val buffer = ByteArray(1024)
            var length: Int
            var downloaded = 0

            // write files
            do {
                length = bis.read(buffer)
                if (length != -1) {
                    fileOutputStream.write(buffer, 0, length)
                    // update progress
                    downloaded += length
                    val percent = (downloaded * 100F / contentLength).toInt()
                    progressDialog?.progress = downloaded
                    Logger.dTag(TAG_APP, "[SplashActivity] [onProgress] progress = $percent")
                }
            } while (length != -1)
            fileOutputStream.flush()
            fileOutputStream.close()
            bis.close()
            inputStream.close()
            Logger.dTag(TAG_APP, "[SplashActivity] [doDownloadApk] download end")

            // write files done
            progressDialog?.dismiss()
            // install apk
            if (FileUtils.isFileExists(apkFile)) {
                @Suppress("DEPRECATION")
                AppUtils.installApp(this@SplashActivity, apkFile, Constants.FILE_PROVIDER, REQUEST_INSTALL_APP)
            }
        } catch (e: Exception) {
            e.printStackTrace()
            runOnUiThread {
                YLToastUtils.showToast(
                    getString(R.string.download_apk_error_try_again, e.message),
                    duration = Toast.LENGTH_LONG
                )
                viewModel.checkSwitchEnable()
            }
        }
    }
}
```

### monkey测试：[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#monkey) <a href="#monkey" id="monkey"></a>

```
adb shell monkey -p <package-name> --throttle 100 --pct-touch 50 --pct-motion 50 -v -v <count> > <file_path>
```

### App物料尺寸[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#app) <a href="#app" id="app"></a>

app icon的尺寸：

logo (3:4:6:8:12:16)

| name    | scale | 普通  | adaptive icon |
| ------- | ----- | --- | ------------- |
| ldip    | 0.75  | 108 | 81            |
| mdpi    | 1     | 144 | 108           |
| hdpi    | 1.5   | 192 | 162           |
| xhdpi   | 2     | 256 | 216           |
| xxhdpi  | 3     | 384 | 324           |
| xxxhdpi | 4     | 512 | 432           |

adaptive icon直接采用Android Studio中Image Asset工具进行生成即可。

两者可以共存：

1. 在mipmap-\*dpi中放入普通的icon，取名为ic\_launcher
2. 使用Image Asset工具生成adaptive icon，在Legacy选项卡中去除不需要的icon。这样会生成会在mipmap-\*dpi目录下生成一套ic\_launcher\_foreground和一套ic\_launcher\_background，同时会在mipmap-anydpi-v26中生成一个ic\_launcher.xml文件，里面会引用ic\_launcher\_foreground和ic\_launcher\_background资源。

引导页 - 2160×1080（适配全面屏） - 1920x1080 - 720x1280

屏幕快照 - 480x800 (vivo) - 1080x1920

### StatusBar和NavigationBar完全透明[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#statusbarnavigationbar) <a href="#statusbarnavigationbar" id="statusbarnavigationbar"></a>

代码方式：

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
    window.statusBarColor = Color.TRANSPARENT
    window.navigationBarColor = Color.TRANSPARENT
}
window.setFlags(
    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS
)
StatusBarUtil.setTransparent(this, false)
```

From [StackOverflow](https://stackoverflow.com/a/31596735/7440866)

### 查看签名证书信息[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_5) <a href="#_5" id="_5"></a>

```
keytool -v -list -keystore <keystore>
```

然后输入密码即可。

### Dynamic Permission[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#dynamic-permission) <a href="#dynamic-permission" id="dynamic-permission"></a>

[Request App Permissions](https://developer.android.com/training/permissions/requesting)

Android在6.0及后对于危险权限需要动态申请，目前(_2019-06-24_)动态权限有以下10组共计26个权限，表格如下：

[Permission groups](https://developer.android.com/guide/topics/permissions/overview.html#perm-groups)Dangerous permissions and permission groups.

| Permission Group | Permissions                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------- |
| CALENDAR         | <p>READ_CALENDAR<br>WRITE_CALENDAR</p>                                                                        |
| CALL\_LOG        | <p>READ_CALL_LOG<br>WRITE_CALL_LOG<br>PROCESS_OUTGOING_CALLS</p>                                              |
| CAMERA           | CAMERA                                                                                                        |
| CONTACTS         | <p>READ_CONTACTS<br>WRITE_CONTACTS<br>GET_ACCOUNTS</p>                                                        |
| LOCATION         | <p>ACCESS_FINE_LOCATION<br>ACCESS_COARSE_LOCATION</p>                                                         |
| MICROPHONE       | RECORD\_AUDIO                                                                                                 |
| PHONE            | <p>READ_PHONE_STATE<br>READ_PHONE_NUMBERS<br>CALL_PHONE<br>ANSWER_PHONE_CALLS<br>ADD_VOICEMAIL<br>USE_SIP</p> |
| SENSORS          | BODY\_SENSORS                                                                                                 |
| SMS              | <p>SEND_SMS<br>RECEIVE_SMS<br>READ_SMS<br>RECEIVE_WAP_PUSH<br>RECEIVE_MMS</p>                                 |
| STORAGE          | <p>READ_EXTERNAL_STORAGE<br>WRITE_EXTERNAL_STORAGE</p>                                                        |

**注**：您的应用仍需要明确请求其需要的每项权限，即使用户已向应用授予该权限组中的其他权限。此外，权限分组在将来的 Android 版本中可能会发生变化。您的代码不应依赖特定权限属于或不属于相同组这种假设。

运行时权限的处理涉及到以下四个方法：

* `ContextCompat.checkSelfPermission`\
  检查权限是否已经授予
* `ActivityCompat.shouldShowRequestPermissionRationale` 在用户已经拒绝某项权限时，用来向用户解释为什么需要权限。\
  如果用户没有申请过、或者第二次或以上拒绝时选择了`Don't ask again`选项，此方法会返回false\
  如果用户拒绝了权限请求，此方法会返回true
* `ActivityCompat.requestPermissions`\
  申请权限
* `onRequestPermissionsResult`\
  申请权限的回调

动态权限申请流程代码片段如下：

```
private fun permission() {
    val thisActivity = this

    // Here, thisActivity is the current activity
    if (ContextCompat.checkSelfPermission(thisActivity,
            Manifest.permission.READ_CONTACTS)
        != PackageManager.PERMISSION_GRANTED) {

        // Permission is not granted
        // Should we show an explanation?
        if (ActivityCompat.shouldShowRequestPermissionRationale(thisActivity,
                Manifest.permission.READ_CONTACTS)) {
            // Show an explanation to the user *asynchronously* -- don't block
            // this thread waiting for the user's response! After the user
            // sees the explanation, try again to request the permission.

            // @@@ add by author
            showRequestPermissionRationale()
            // @@@
        } else {
            // No explanation needed, we can request the permission.
            ActivityCompat.requestPermissions(thisActivity,
                arrayOf(Manifest.permission.READ_CONTACTS),
                MY_PERMISSIONS_REQUEST_READ_CONTACTS)

            // MY_PERMISSIONS_REQUEST_READ_CONTACTS is an
            // app-defined int constant. The callback method gets the
            // result of the request.
        }
    } else {
        // Permission has already been granted
    }
}

override fun onRequestPermissionsResult(requestCode: Int,
                                        permissions: Array<String>, grantResults: IntArray) {
    when (requestCode) {
        MY_PERMISSIONS_REQUEST_READ_CONTACTS -> {

            // If request is cancelled, the result arrays are empty.
            if ((grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED)) {
                // permission was granted, yay! Do the
                // contacts-related task you need to do.
            } else {
                // permission denied, boo! Disable the
                // functionality that depends on this permission.
            }
            return
        }

        // Add other 'when' lines to check for other
        // permissions this app might request.
        else -> {
            // Ignore all other requests.
        }
    }
}

private fun showRequestPermissionRationale() {
    AlertDialog.Builder(this)
        .setTitle("RequestPermissionRationale")
        .setMessage("This is the message")
        .setPositiveButton(android.R.string.ok) { _, _ ->
            ActivityCompat.requestPermissions(this,
                arrayOf(Manifest.permission.READ_CONTACTS),
                MY_PERMISSIONS_REQUEST_READ_CONTACTS)
        }.setNegativeButton(android.R.string.cancel, null)
        .create()
        .show()
}

companion object {
    private const val MY_PERMISSIONS_REQUEST_READ_CONTACTS = 0x1111
}
```

[![动态权限流程图](https://blog.yorek.xyz/assets/images/android/dynamic-permission-flow.png)](https://blog.yorek.xyz/assets/images/android/dynamic-permission-flow.png)

> BTW，在权限请求回调的权限拒绝的分支中，此时我们再次调用`ActivityCompat.shouldShowRequestPermissionRationale`方法，如果返回true表示用户拒绝了权限；如果返回了false，表示用户拒绝了权限且勾选了不再提醒，此时我们弹框提醒用户。\
> 可以参考PermissionDispatcher中生成的辅助代码——[MainActivity.kt对应的辅助代码](https://blog.yorek.xyz/android/3rd-library/permissiondispatcher/#kt)

### Google Chrome强制禁用darkmode[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#google-chromedarkmode) <a href="#google-chromedarkmode" id="google-chromedarkmode"></a>

```
defaults write com.google.Chrome NSRequiresAquaSystemAppearance -bool YES
```

### ANR的判定[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#anr) <a href="#anr" id="anr"></a>

1. Activity超过5秒无响应
2. BroadcastReceiver超过10秒无响应
3. Service超过20秒无响应

### RecyclerView是否到顶部[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#recyclerview) <a href="#recyclerview" id="recyclerview"></a>

`RecyclerView.canScrollVertically(-1)`

* `true` 表示还没有到顶部
* `false` 表示到顶部了

### 判断程序是否运行在主进程[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#\_6) <a href="#_6" id="_6"></a>

私有多进程下，pid相等的进程可能是主进程或者子进程。获取当前进程id对应的进程，判断进程名是否是包名即可。

```
private fun isMainProcess(context: Context) : Boolean {
    val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as? ActivityManager

    activityManager?.runningAppProcesses?.find {
        it.pid == Process.myPid()
    }?.let {
        return it.processName == context.packageName
    }

    return false
}
```

### Drawable2Bitmap[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#drawable2bitmap) <a href="#drawable2bitmap" id="drawable2bitmap"></a>

通过Canvas将Drawable画到空白的Bitmap上。

```
private fun getBitmapFromDrawableResourceId(context: Context, @DrawableRes drawableId: Int): Bitmap {
    val drawable = ContextCompat.getDrawable(context, drawableId)!!
    val bitmap = Bitmap.createBitmap(drawable.intrinsicWidth, drawable.intrinsicHeight, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(bitmap)
    drawable.setBounds(0, 0, drawable.intrinsicWidth, drawable.intrinsicHeight)
    drawable.draw(canvas)

    return bitmap
}
```

### Gravity的妙用[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#gravity) <a href="#gravity" id="gravity"></a>

在一个小红点需求中，由于想封装的更完整一点，所以设计了一个包裹普通View的小红点FrameLayout，该FrameLayout监听对应的小红点事件，从而显示或隐藏小红点。这样业务上的View一般情况无需处理小红点的相关逻辑。\
所以在设计BadgeFrameLayout时，需要事先设置好小红点需要显示的位置，这里参考了gravity的一些实现：

BadgeFrameLayout使用下面属性来指定小红点的位置：

```
<declare-styleable name="BadgeFrameLayout">
    <!-- badge的方位 -->
    <attr name="badgeGravity" format="flags">
        <flag name="top" value="0x30" />
        <flag name="bottom" value="0x50" />
        <flag name="left" value="0x03" />
        <flag name="right" value="0x05" />
        <flag name="center_vertical" value="0x10" />
        <flag name="center_horizontal" value="0x01" />
        <flag name="center" value="0x11" />
    </attr>
    <!-- badge在X-Y方向上的偏移量，follow gravity -->
    <attr name="badgeXAdj" format="dimension" />
    <attr name="badgeYAdj" format="dimension" />
    ...
</declare-styleable>
```

在代码中这样来确定小红点的rect：

```
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec)

    // 容器的rect
    mBadgeContainer.set(0, 0, measuredWidth, measuredHeight)
    // mBadgeRadius为小红点的半径，mBadgeRect就是计算出来的小红点的rect
    Gravity.apply(mGravity,
        mBadgeRadius shl 1,
        mBadgeRadius shl 1,
        mBadgeContainer,
        mBadgeXAdj,
        mBadgeYAdj,
        mBadgeRect
    )
}
```

`Gravity.apply`方法可以避免绘制超过边界，不需要我们自己做一些麻烦的运算，还是相当方便的。方法解释见： [https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Gravity.java;l=186?q=Gravity.apply](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/view/Gravity.java;l=186?q=Gravity.apply)

### 9patch[¶](https://blog.yorek.xyz/android/other/%E7%90%90%E7%A2%8E%E7%9F%A5%E8%AF%86%E7%82%B9/#9patch) <a href="#9patch" id="9patch"></a>

长期不用还是会忘记，具体来说可以参考[Google官网](https://developer.android.google.cn/studio/write/draw9patch?hl=zh\_cn)

左边上边的线条决定哪些区域可以被缩放，右边下边的线条决定哪个区域显示内容。

最后更新: 2020年2月13日\

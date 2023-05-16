---
description: 转：https://juejin.im/post/5dd274515188254c6443815e
---

# 插件化、模块化、组件化、热修复、增量更新、Gradle

### 1.对热修复和插件化的理解

[blog.csdn.net/github\_3713…](https://blog.csdn.net/github\_37130188/article/details/89762543)

```
Android 类加载器
PathClassLoader.java
DexClassLoader.java
BaseDexClassLoader.java
DexPathList.java
复制代码
```

```
（1）PathClassLoader：只能加载已经安装到Android系统中的apk文件（/data/app目录），是Android默认使用的类加载器。

（2）DexClassLoader：可以加载任意目录下的dex/jar/apk/zip文件，比PathClassLoader更灵活，是实现热修复的重点。
复制代码
```

```
// PathClassLoader
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
 
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
 
 
// DexClassLoaderpublic 
class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
}
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/19/16e7f9a64f81a28b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

BaseDexClassLoader

```
dexPath：要加载的程序文件（一般是dex文件，也可以是jar/apk/zip文件）所在目录。
optimizedDirectory：dex文件的输出目录（因为在加载jar/apk/zip等压缩格式的程序文件时会解压出其中的dex文件，该目录就是专门用于存放这些被解压出来的dex文件的）。
libraryPath：加载程序文件时需要用到的库路径。
parent：父加载器
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/19/16e7ffed57448064?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
阿里系：DeXposed、andfix：从底层二进制入手（c语言）。阿里andFix hook 方法在native的具体字段。
       art虚拟机上是一个叫ArtMethod的结构体。通过修改该结构体上有bug的字段来达到修复bug方法的目的，
       但这个artMethod是根据安卓原生的结构写死的，国内很多第三方厂家会改写ArtMethod结构，导致替换失效。
腾讯系：tinker：从java加载机制入手。qq的dex插装就类似上面分析的那种。通过将修复的dex文件插入到app的dexFileList的前面，达到更新bug的效果，但是不能及时生效，需要重启。
        但虚拟机在安装期间会为类打上CLASS_ISPREVERIFIED标志，是为了提高性能的，我们强制防止类被打上标志是否会有些影响性能
美团robust：是在编译器为每个方法插入了一段逻辑代码，并为每个类创建了一个ChangeQuickRedirect静态成员变量，当它不为空会转入新的代码逻辑达到修复bug的目的。
            优点是兼容性高,但是会增加应用体积
复制代码
```

```
PathClassLoader和DexClassLoader都继承自BaseDexClassLoader

1、Android使用PathClassLoader作为其类加载器，只能去加载已经安装到Android系统中的apk文件；

2、DexClassLoader可以从.jar和.apk类型的文件内部加载classes.dex文件就好了。热修复也用到这个类。

（1）动态改变BaseDexClassLoader对象间接引用的dexElements；
（2）在app打包的时候，阻止相关类去打上CLASS_ISPREVERIFIED标志。
（3）我们使用 hook 思想代理 startActivity 这个方法，使用占坑的方式。
复制代码
```

```
1. startActivity 的时候最终会走到 AMS 的 startActivity 方法
2. 系统会检查一堆的信息验证这个 Activity 是否合法。
3. 然后会回调 ActivityThread 的 Handler 里的 handleLaunchActivity
4. 在这里走到了 performLaunchActivity 方法去创建 Activity 并回调一系列生命周期的方法
5. 创建 Activity 的时候会创建一个 LoaderApk对象，然后使用这个对象的 getClassLoader 来创建 Activity
6. 我们查看 getClassLoader() 方法发现返回的是 PathClassLoader，然后他继承自 BaseDexClassLoader
7. 然后我们查看 BaseDexClassLoader 发现他创建时创建了一个 DexPathList 类型的 pathList对象，然后在 findClass 时调用了 pathList.findClass 的方法
8. 然后我们查看 DexPathList类 中的 findClass 发现他内部维护了一个 Element[] dexElements的dex 数组，findClass 时是从数组中遍历查找的
复制代码
```

### 2.插件化原理分析

[cloud.tencent.com/developer/a…](https://cloud.tencent.com/developer/article/1071815)

![](https://user-gold-cdn.xitu.io/2019/11/19/16e82db92f6945fa?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
DexClassLoader和PathClassLoader，它们都继承于BaseDexClassLoader。
DexClassloader多传了一个optimizedDirectory
复制代码
```

DexPathList

![](https://user-gold-cdn.xitu.io/2019/11/19/16e82ff2e9f684eb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

多DexClassLoader

![](https://user-gold-cdn.xitu.io/2019/11/19/16e83006fdeca2d9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
每个插件单独一个DexClassLoader，相对隔离，RePlugin采用该方案
复制代码
```

单DexClassLoader

```
将插件的DexClassLoader中的pathList合并到主工程的DexClassLoader中。方便插件与宿主(插件)之间的调用，Small采用该方案
复制代码
```

```
插件调用主工程
主工程的ClassLoader作为插件ClassLoader的父加载器

主工程调用插件
若使用多ClassLoader机制，通过插件的ClassLoader先加载类，再通过反射调用
若使用单ClassLoader机制，直接通过类名去访问插件中的类，弊端是库的版本可能不一致，需要规范
复制代码
```

资源加载

```
//创建AssetManager对象 
AssetManager assets = new AssetManager();
 //将apk路径添加到AssetManager中
  if (assets.addAssetPath(resDir) == 0){              
    return null;  
}
 //创建Resource对象

r = new Resources(assets, metrics, getConfiguration(), compInfo);

插件apk的路径加入到AssetManager中
通过反射去创建，并且部分Rom对创建的Resource类进行了修改，所以需要考虑不同Rom的兼容性。
复制代码
```

资源路径的处理

![](https://user-gold-cdn.xitu.io/2019/11/19/16e830a8e16ebe57?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Context的处理

```
// 第一步：创建Resource

if (Constants.COMBINE_RESOURCES) {
    //插件和主工程资源合并时需要hook住主工程的资源
    Resources resources = ResourcesManager.createResources(context, apk.getAbsolutePath());
    ResourcesManager.hookResources(context, resources);  
    return resources;
} else {  
    //插件资源独立，该resource只能访问插件自己的资源
    Resources hostResources = context.getResources();
    AssetManager assetManager = createAssetManager(context, apk);  
    return new Resources(assetManager, hostResources.getDisplayMetrics(), hostResources.getConfiguration());
}
复制代码
```

```
//第二步：hook主工程的Resource

//对于合并式的资源访问方式，需要替换主工程的Resource，下面是具体替换的代码。

public static void hookResources(Context base, Resources resources) { 
   try {
        ReflectUtil.setField(base.getClass(), base, "mResources", resources);
        Object loadedApk = ReflectUtil.getPackageInfo(base);
        ReflectUtil.setField(loadedApk.getClass(), loadedApk, "mResources", resources);

        Object activityThread = ReflectUtil.getActivityThread(base);
        Object resManager = ReflectUtil.getField(activityThread.getClass(), activityThread, "mResourcesManager");       
        if (Build.VERSION.SDK_INT < 24) {
            Map<Object, WeakReference<Resources>> map = (Map<Object, WeakReference<Resources>>) ReflectUtil.getField(resManager.getClass(), resManager, "mActiveResources");
            Object key = map.keySet().iterator().next();
            map.put(key, new WeakReference<>(resources));
        } else {                
            // still hook Android N Resources, even though it's unnecessary, then nobody will be strange.
            Map map = (Map) ReflectUtil.getFieldNoException(resManager.getClass(), resManager, "mResourceImpls");
            Object key = map.keySet().iterator().next();
            Object resourcesImpl = ReflectUtil.getFieldNoException(Resources.class, resources, "mResourcesImpl");
            map.put(key, new WeakReference<>(resourcesImpl));
        }
    } catch (Exception e) {
        e.printStackTrace();
    }
复制代码
```

替换了主工程context中LoadedApk的mResource对象

将新的Resource添加到主工程ActivityThread的mResourceManager中，并且根据Android版本做了不同处理

```
//第三步：关联resource和Activity

Activity activity = mBase.newActivity(plugin.getClassLoader(), targetClassName, intent);
activity.setIntent(intent);
//设置Activity的mResources属性，Activity中访问资源时都通过mResources

ReflectUtil.setField(ContextThemeWrapper.class, activity, "mResources", plugin.getResources());
复制代码
```

资源冲突

资源id是由8位16进制数表示，表示为0xPPTTNNNN， 由三部分组成：PackageId+TypeId+EntryId

```
修改aapt源码，编译期修改PP段。
修改resources.arsc文件，该文件列出了资源id到具体资源路径的映射。
复制代码
```

[blog.csdn.net/jiangwei091…](https://blog.csdn.net/jiangwei0910410003/article/details/50820219)

```
// Main.cpp
result = handleCommand(&bundle);
case kCommandPackage: return doPackage(bundle);

// Command.cpp
int doPackage(Bundle* bundle) {
    if (bundle->getResourceSourceDirs().size() || bundle->getAndroidManifestFile()) {
        err = buildResources(bundle, assets, builder);
        if (err != 0) {
            goto bail;
        }
    }
}

Resource.cpp
buildResources

ResourceTable.cpp

switch(mPackageType) {
    case App:
    case AppFeature:
        packageId = 0x7f;
        break;
    case System:
        packageId = 0x01;
        break;
    case SharedLibrary:
        packageId = 0x00;
        break;    
}

复制代码
```

```
首先找到入口类：Main.cpp：main函数，解析参数，然后调用handleCommand函数处理参数对应的逻辑，我们看到了有一个函数doPackage。

然后就搜索到了Command.cpp：在他内部的doPackage函数中进行编译工具的一个函数：buildResources函数，在全局搜索，发现了Resource.cpp：发现这里就是处理编译工作，构建ResourceTable的逻辑，在ResourceTable.cpp中，也是获取PackageId的地方，下面我们就来看看如何修改呢？

其实最好的方法是，能够修改aapt源码，添加一个参数，把我们想要编译的PackageId作为输入值，传进来最好了，那就是Bundle类型，他是从Main.cpp中的main函数传递到了最后的buildResources函数中，那么我们就可以把这个参数用Bundle进行携带。
复制代码
```

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="784"%20height="330">\</svg>)

[juejin.im/entry/68449…](https://juejin.im/entry/6844903727711649800) [www.jianshu.com/p/8d691b6bf…](https://www.jianshu.com/p/8d691b6bf8b4)

————————————————————————————————————————————————

[cloud.tencent.com/developer/a…](https://cloud.tencent.com/developer/article/1029254)

在整个过程中，需要修改到R文件、resources.arsc和二进制的xml文件

四大组件支持

ProxyActivity代理

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="1013"%20height="389">\</svg>)

```
代理方式的关键总结起来有下面两点：

ProxyActivity中需要重写getResouces，getAssets，getClassLoader方法返回插件的相应对象。生命周期函数以及和用户交互相关函数，如onResume，onStop，onBackPressedon，KeyUponWindow，FocusChanged等需要转发给插件。
PluginActivity中所有调用context的相关的方法，如setContentView，getLayoutInflater，getSystemService等都需要调用ProxyActivity的相应方法。

该方式有几个明显缺点：

插件中的Activity必须继承PluginActivity，开发侵入性强。
如果想支持Activity的singleTask，singleInstance等launchMode时，需要自己管理Activity栈，实现起来很繁琐。
插件中需要小心处理Context，容易出错。
如果想把之前的模块改造成插件需要很多额外的工作。
复制代码
```

预埋StubActivity，hook系统启动Activity的过程

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="1168"%20height="423">\</svg>)

```
VirtualAPK通过替换了系统的Instrumentation，hook了Activity的启动和创建，省去了手动管理插件Activity生命周期的繁琐，让插件Activity像正常的Activity一样被系统管理，并且插件Activity在开发时和常规一样，即能独立运行又能作为插件被主工程调用。

其他插件框架在处理Activity时思想大都差不多，无非是这两种方式之一或者两者的结合。在hook时，不同的框架可能会选择不同的hook点。如360的RePlugin框架选择hook了系统的ClassLoader，即构造Activity2的ClassLoader，在判断出待启动的Activity是插件中的时，会调用插件的ClassLoader构造相应对象。另外RePlugin为了系统稳定性，选择了尽量少的hook，因此它并没有选择hook系统的startActivity方法来替换intent，而是通过重写Activity的startActivity，因此其插件Activity是需要继承一个类似PluginActivity的基类的。不过RePlugin提供了一个Gradle插件将插件中的Activity的基类换成了PluginActivity，用户在开发插件Activity时也是没有感知的。
复制代码
```

[www.jianshu.com/p/ac96420fc…](https://www.jianshu.com/p/ac96420fc82c)

[sanjay-f.github.io/2016/04/17/…](https://sanjay-f.github.io/2016/04/17/%E6%BA%90%E7%A0%81%E6%8E%A2%E7%B4%A2%E7%B3%BB%E5%88%9731---%E6%8F%92%E4%BB%B6%E5%8C%96%E5%9F%BA%E7%A1%80%E4%B9%8BService%E7%BB%84%E4%BB%B6%E7%AE%A1%E7%90%86/)

[www.jianshu.com/p/d43e1fb42…](https://www.jianshu.com/p/d43e1fb426f3)

```
Service插件化总结

初始化时通过ActivityManagerProxy Hook住了IActivityManager。
服务启动时通过ActivityManagerProxy拦截，判断是否为远程服务，如果为远程服务，启动RemoteService，如果为同进程服务则启动LocalService。
如果为LocalService，则通过DexClassLoader加载目标Service，然后反射调用attach方法绑定Context，然后执行Service的onCreate、onStartCommand方法
如果为RemoteService，则先加载插件的远程Service，后续跟LocalService一致。
复制代码
```

### 3.模块化实现（好处，原因）

[www.cnblogs.com/Jackie-zhan…](https://www.cnblogs.com/Jackie-zhang/p/9875581.html)

```
1、模块间解耦，复用。
（原因：对业务进行模块化拆分后，为了使各业务模块间解耦，因此各个都是独立的模块，它们之间是没有依赖关系。
每个模块负责的功能不同，业务逻辑不同，模块间业务解耦。模块功能比较单一，可在多个项目中使用。）

2、可单独编译某个模块，提升开发效率。
（原因：每个模块实际上也是一个完整的项目，可以进行单独编译，调试）

3、可以多团队并行开发，测试。
原因：每个团队负责不同的模块，提升开发，测试效率。
复制代码
```

组件化与模块化

组件化是指以重用化为目的，将一个系统拆分为一个个单独的组件

```
避免重复造轮子，节省开发维护成本；
降低项目复杂性，提升开发效率；
多个团队公用同一个组件，在一定层度上确保了技术方案的统一性。
复制代码
```

模块化业务分层：由下到上

```
基础组件层：
底层使用的库和封装的一些工具库（libs），比如okhttp,rxjava,rxandroid,glide等
业务组件层：
与业务相关，封装第三方sdk，比如封装后的支付，即时通行等
业务模块层：
按照业务划分模块，比如说IM模块，资讯模块等
复制代码
```

Library Module开发问题

```
在把代码抽取到各个单独的Library Module中，会遇到各种问题。
最常见的就是R文件问题，Android开发中，各个资源文件都是放在res目录中，在编译过程中，会生成R.java文件。
R文件中包含有各个资源文件对应的id，这个id是静态常量，但是在Library Module中，这个id不是静态常量，那么在开发时候就要避开这样的问题。

举个常见的例子，同一个方法处理多个view的点击事件，有时候会使用switch(view.getId())这样的方式，
然后用case R.id.btnLogin这样进行判断，这时候就会出现问题，因为id不是经常常量，那么这种方式就用不了。
复制代码
```

### 4.热修复、插件化

[www.jianshu.com/p/704cac3eb…](https://www.jianshu.com/p/704cac3eb13d)

```
宿主： 就是当前运行的APP
插件： 相对于插件化技术来说，就是要加载运行的apk类文件
补丁： 相对于热修复技术来说，就是要加载运行的.patch,.dex,*.apk等一系列包含dex修复内容的文件。
复制代码
```

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="543"%20height="384">\</svg>)

QQ 空间超级补丁方案

Tinker

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="669"%20height="392">\</svg>)

HotFix

![](data:image/svg+xml;utf8,\<?xml%20version="1.0"?>\<svg%20xmlns="http://www.w3.org/2000/svg"%20version="1.1"%20width="1142"%20height="605">\</svg>)

```
当然就热修复的实现，各个大厂还有各自的实现，比如饿了吗的Amigo,美团的Robust,实现及优缺点各有差异，但总的来说就是两大类

ClassLoader 加载方案
Native层替换方案
或者是参考Android Studio Instant Run 的思路实现代码整体的增量更新。但这样势必会带来性能的影响。
复制代码
```

Sophix

[www.jianshu.com/p/4d30ce3e5…](https://www.jianshu.com/p/4d30ce3e55dd)

```
底层替换方案
原理：在已经加载的类中直接替换掉原有方法，是在原有类的结构基础上进行修改的。在hook方法入口ArtMethod时，通过构造一个新的ArtMethod实现替换方法入口的跳转。
应用：能即时生效，Andfix采用此方案。
缺点：底层替换稳定性不好，适用范围存在限制，通过改造代码绕过限制既不优雅也不方便，并且还没提供资源及so的修复。
类加载方案
原理：让app重新启动后让ClassLoader去加载新的类。如果不重启，原来的类还在虚拟机中无法重复加载。

优点：修复范围广，限制少。

应用：腾讯系包括QQ空间，手QFix，Tinker采用此方案。
QQ空间会侵入打包流程。
QFix需要获取底层虚拟机的函数，不稳定。
Tinker是完整的全量dex加载。
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/20/16e850668846ac0e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
Tinker与Sophix方案不同之处
Tinker采用dex merge生成全量DEX方案。反编译为smali，然后新apk跟基线apk进行差异对比，最后得到补丁包。
Dalvik下Sophix和Tinker相同，在Art下，Sophix不需要做dex merge，因为Art下本质上虚拟机已经支持多dex的加载，要做的仅仅是把补丁dex作为主dex(classes.dex)加载而已：
将补丁dex命名为classes.dex，原apk中的dex依次命名为classes(2, 3, 4...).dex就好了，然后一起打包为一个压缩文件。然后DexFile.loadDex得到DexFile对象，最后把该DexFile对象整个替换旧的dexElements数组就好了。

资源修复方案
基本参考InstantRun的实现：构造一个包含所有新资源的新的AssetManager。并在所有之前引用到原来的AssetManager通过反射替换掉。
Sophix不修改AssetManager的引用，构造的补丁包中只包含有新增或有修改变动的资源，在原AssetManager中addAssetPath这个包就可以了。资源包不需要在运行时合成完整包。

so库修复方案
本质是对native方法的修复和替换。类似类修复反射注入方式，将补丁so库的路径插入到nativeLibraryDirectories数据最前面。
复制代码
```

Method Hook

[www.jianshu.com/p/7dcb32f8a…](https://www.jianshu.com/p/7dcb32f8a0ce) [pqpo.me/2017/07/07/…](https://pqpo.me/2017/07/07/hotfix-method-hook/)

### 5.项目组件化的理解

[juejin.im/post/684490…](https://juejin.im/post/6844903649102004231)

```
总结
组件化相较于单一工程，在组件模式下可以提高编译速度，方便单元测试，提高开发效率。
开发人员分工更加明确，基本上做到互不干扰。
业务组件的架构也可以自由选择，不影响同伴之间的协作。
降低维护成本，代码结构更加清晰。
复制代码
```

### 6.描述清点击 Android Studio 的 build 按钮后发生了什么

[blog.csdn.net/u011026779/…](https://blog.csdn.net/u011026779/article/details/69947666) [blog.csdn.net/github\_3713…](https://blog.csdn.net/github\_37130188/article/details/89762696)

```
apply plugin : 'com.android.application'
apply plugin : 'com.android.library'

编译五阶段

1.准备依赖包 Preparation of dependecies
2.合并资源并处理清单 Merging resources and proccesssing Manifest
3.编译 Compiling
4.后期处理 Postprocessing
5.包装和出版 Packaging and publishing
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/20/16e87a39572814ef?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
简单构建流程：
1. Android编译器（5.0之前是Dalvik，之后是ART）将项目的源代码（包括一些第三方库、jar包和aar包）转换成DEX文件，将其他资源转换成已编译资源。

2. APK打包器将DEX文件和已编译资源在使用秘钥签署后打包。

3. 在生成最终 APK 之前，打包器会使用zipalign 等工具对应用进行优化，减少其在设备上运行时的内存占用。

构建流程结束后获得测试或发布用的apk。
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/20/16e87a64ed3e3bd7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

```
图中的矩形表示用到或者生成的文件，椭圆表示工具。
1. 通过aapt打包res资源文件，生成R.java、resources.arsc和res文件
2. 处理.aidl文件，生成对应的Java接口文件
3. 通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件
4. 通过dex命令，将.class文件和第三方库中的.class文件处理生成classes.dex
5. 通过apkbuilder工具，将aapt生成的resources.arsc和res文件、assets文件和classes.dex一起打包生成apk
6. 通过Jarsigner工具，对上面的apk进行debug或release签名
7. 通过zipalign工具，将签名后的apk进行对齐处理。
这样就得到了一个可以安装运行的Android程序。
复制代码
```

![](https://user-gold-cdn.xitu.io/2019/11/20/16e87a72c1d38b6c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 7.彻底搞懂Gradle、Gradle Wrapper与Android Plugin for Gradle的区别和联系

[zhuanlan.zhihu.com/p/32714369](https://zhuanlan.zhihu.com/p/32714369) [blog.csdn.net/LVXIANGAN/a…](https://blog.csdn.net/LVXIANGAN/article/details/84869770)

```
`Offline work`时可能出现"No cached version of com.android.tools.build:gradle:xxx available for offline mode"问题
复制代码
```

```
Gradle：   gradle-wrapper.properties中的distributionUrl=https/://services.gradle.org/distributions/gradle-2.10-all.zip
Gradle插件：build.gradle中依赖的classpath 'com.android.tools.build:gradle:2.1.2'
复制代码
```

```
Gradle：
一个构建系统，构建项目的工具，用来编译Android app，能够简化你的编译、打包、测试过程。

Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化建构工具。它使用一种基于Groovy的特定领域语言来声明项目设置，而不是传统的XML。当前其支持的语言限于Java、Groovy和Scala

Gradle插件：

我们在AS中用到的Gradle被叫做Android Plugin for Gradle，它本质就是一个AS的插件，它一边调用 Gradle本身的代码和批处理工具来构建项目，一边调用Android SDK的编译、打包功能。
Gradle插件跟 Android SDK BuildTool有关联，因为它还承接着AS里的编译相关的功能，在项目的 local.properties 文件里写明 Android SDK 路径、在build.gradle 里写明 buildToolsVersion 的原因。
复制代码
```

| 插件版本          | Gradle 版本    |
| ------------- | ------------ |
| 1.0.0 - 1.1.3 | 2.2.1 - 2.3  |
| 1.2.0 - 1.3.1 | 2.2.1 - 2.9  |
| 1.5.0         | 2.2.1 - 2.13 |
| 2.0.0 - 2.1.2 | 2.10 - 2.13  |
| 2.1.3 - 2.2.3 | 2.14.1+      |
| 2.3.0+        | 3.3+         |
| 3.0.0+        | 4.1+         |
| 3.1.0+        | 4.4+         |
| 3.2.0 - 3.2.1 | 4.6+         |
| 3.3.0 - 3.3.2 | 4.10.1+      |
| 3.4.0+        | 5.1.1+       |

Done\
作者：爱雨浮龙\
链接：https://juejin.im/post/6844904022017572877\
来源：掘金\
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

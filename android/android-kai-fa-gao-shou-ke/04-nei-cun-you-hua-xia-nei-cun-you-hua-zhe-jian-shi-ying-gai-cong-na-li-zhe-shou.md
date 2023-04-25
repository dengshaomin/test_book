# 04 | 内存优化（下）：内存优化这件事，应该从哪里着手？

**内存优化探讨**

那要进行内存优化，应该从哪里着手呢？我通常会从设备分级、Bitmap 优化和内存泄漏这三个方面入手

1. 设备分级\
   **内存优化首先需要根据设备环境来综合考虑**，专栏上一期我提到过很多同学陷入的一个误区：“内存占用越少越好”。其实我们可以让高端设备使用更多的内存，做到针对设备性能的好坏使用不同的内存分配和回收策略。\
   当然这需要有一个良好的架构设计支撑，在架构设计时需要做到以下几点。
   * 设备分级。使用类似 [device-year-class](https://blog.yorek.xyz/android/paid/master/memory\_2/\(https://github.com/facebook/device-year-class\)) 的策略对设备分级，对于低端机用户可以关闭复杂的动画，或者是某些功能；使用 565 格式的图片，使用更小的缓存内存等。在现实环境下，不是每个用户的设备都跟我们的测试机一样高端，在开发过程我们要学会思考功能要不要对低端机开启、在系统资源吃紧的时候能不能做降级。
   * 缓存管理。我们需要有一套统一的缓存管理机制，可以适当地使用内存；当“系统有难”时，也要义不容辞地归还。我们可以使用 OnTrimMemory 回调，根据不同的状态决定释放多少内存。对于大项目来说，可能存在几十上百个模块，统一缓存管理可以更好地监控每个模块的缓存大小。
   * 进程模型。一个空的进程也会占用 10MB 的内存，而有些应用启动就有十几个进程，甚至有些应用已经从双进程保活升级到四进程保活，所以减少应用启动的进程数、减少常驻进程、有节操的保活，对低端机内存优化非常重要。
   * 安装包大小。安装包中的代码、资源、图片以及 so 库的体积，跟它们占用的内存有很大的关系。一个 80MB 的应用很难在 512MB 内存的手机上流畅运行。这种情况我们需要考虑针对低端机用户推出 4MB 的轻量版本，例如 Facebook Lite、今日头条极速版都是这个思路。\
     安装包中的代码、图片、资源以及 so 库的大小跟内存究竟有哪些关系？你可以参考下面的这个表格。\
     ![安装包中代码、图片、资源、so库的大小与内存的关系](https://blog.yorek.xyz/assets/images/android/memory\_in\_apk.png)\
     安装包中代码、图片、资源、so库的大小与内存的关系
2.  Bitmap优化\
    Bitmap 内存一般占应用总内存很大一部分，所以做内存优化永远无法避开图片内存这个“永恒主题”。\
    即使把所有的 Bitmap 都放到 Native 内存，并不代表图片内存问题就完全解决了，这样做只是提升了系统内存利用率，减少了 GC 带来的一些问题而已。

    * 统一图片库\
      图片内存优化的前提是收拢图片的调用，这样我们可以做整体的控制策略。例如低端机使用 565 格式、更加严格的缩放算法，可以使用 Glide、Fresco 或者采取自研都可以。而且需要进一步将所有 Bitmap.createBitmap、BitmapFactory 相关的接口也一并收拢。
    * 统一监控\
      在统一图片库后就非常容易监控 Bitmap 的使用情况了，这里主要有三点需要注意。
      * 大图片监控。我们需要注意某张图片内存占用是否过大，例如长宽远远大于 View 甚至是屏幕的长宽。在开发过程中，如果检测到不合规的图片使用，应该立即弹出对话框提示图片所在的 Activity 和堆栈，让开发同学更快发现并解决问题。在灰度和线上环境下可以将异常信息上报到后台，我们可以计算有多少比例的图片会超过屏幕的大小，也就是图片的 **“超宽率”**。
      * 重复图片监控。重复图片指的是 Bitmap 的像素数据完全一致，但是有多个不同的对象存在。这个监控不需要太多的样本量，一般只在内部使用。下图是一个简单的例子，你可以看到两张图片的内容完全一样，通过解决这张重复图片可以节省 1MB 内存。
      * 图片总内存。通过收拢图片使用，我们还可以统计应用所有图片占用的内存，这样在线上就可以按不同的系统、屏幕分辨率等维度去分析图片内存的占用情况。**在 OOM 崩溃的时候，也可以把图片占用的总内存、Top N 图片的内存都写到崩溃日志中，帮助我们排查问题。**

    讲完设备分级和 Bitmap 优化，我们发现架构和监控需要两手抓，一个好的架构可以减少甚至避免我们犯错，而一个好的监控可以帮助我们及时发现问题。
3. 内存泄漏
   * Java内存泄漏
   * OOM监控
   * Native内存监控
   * GC监控

总的来说，内存优化应该看以下方面：

* 设备分级：缓存管理、进程模型、安装包大小
* Bitmap优化，Native内存，统一图片库、统一监控
* 内存泄漏：Java内存泄漏、OOM监控、Native内存监控、GC监控

### 课后作业[¶](https://blog.yorek.xyz/android/paid/master/memory\_2/#\_1) <a href="#_1" id="_1"></a>

使用HAHA库快速判断内存中是否存在重复的图片，且将这些重复图片的PNG、堆栈等信息输出。

该作业可以参考微信开源的Matrix中的部分[DuplicatedBitmapAnalyzer.java](https://github.com/Tencent/matrix/blob/master/matrix/matrix-android/matrix-resource-canary/matrix-resource-canary-analyzer/src/main/java/com/tencent/matrix/resource/analyzer/DuplicatedBitmapAnalyzer.java)，效果更好。

自己的作业如下，需要借助`leakcanary-analayzer-1.6.2.jar`以及`leakcanary-watcher-1.6.2.jar`解析堆栈信息。

实践起来还是有一些问题：

* 对比之下，堆栈打印不准确
* 自己作业对比微信的，找出来的元素更多，应该是误报

build.gradle

```
apply plugin: 'java'
apply plugin: 'kotlin'

version 1.0

dependencies {
    implementation fileTree(include: ['*.jar'], dir: 'libs')
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation 'com.squareup.haha:haha:2.0.4'
    implementation rootProject.ext.dependencies.gson
}

jar {
    manifest {
        attributes 'Main-Class': 'com.ximalaya.ting.kid.bitmap.Main'
        attributes 'Manifest-Version': version
    }

    from {
        exclude 'META-INF/MANIFEST.MF'
        exclude 'META-INF/*.SF'
        exclude 'META-INF/*.DSA'
        exclude 'META-INF/*.RSA'
        configurations.runtimeClasspath.collect {
            it.isDirectory() ? it : zipTree(it)
        }
    }
}

// copy the jar to work directory
task buildAlloctrackJar(type: Copy, dependsOn: [build, jar]) {
    group = "buildTool"
    from('build/libs') {
        include '*.jar'
        exclude '*-javadoc.jar'
        exclude '*-sources.jar'
    }
    into(rootProject.file("tools"))
}
```

Main.javaDuplicateBitmapChecker.ktLogWrapper.ktLeakTraceParser.ktDigestUtil.javaAnalyzerResult.ktBitmapExtractor.java

最后更新: 2020年6月8日\

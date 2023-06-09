# Android Studio build过程

Android Studio点击build按钮后，Android Studio就会编译整个项目并将apk安装到手机上，请详细描述下这个过程的背后到底发生了什么？

点击build按钮后，AS会根据Build Variants中Module的Build Variant的类型对对应的Module执行`gradlew :${Module}:assemble${variant}`\
以一个典型的单Module的工程为例，此处就是`gradlew :app:assembleDebug`

### [The build process](https://developer.android.com/studio/build#build-process)[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#the-build-process) <a href="#the-build-process" id="the-build-process"></a>

下面是典型Android应用的构建过程：

![典型Android App模块的构建过程](https://blog.yorek.xyz/assets/images/android/build-process.png)

该过程一般分为四步：编译 -> 打包 -> 签名 -> 对齐

1. 编译器将源代码转换为DEX（Dalvik Executable）文件，其中包括在Android设备上运行的字节码；其余的作为打包好的资源
2. APK Packager把DEX文件以及打包好的资源再次打包成一个单一的APK。在安装或发布APK之前，APK必须要被签名
3. APK Packager会使用debug签名文件或release签名文件对APK进行签名
4. 在生成最终的APK之前，APK Packager会使用[zipalign](https://developer.android.com/studio/command-line/zipalign.html)工具进行优化，此工具会使我们的App在运行时使用更少的内存

![更详细的Android App模块的构建过程](https://blog.yorek.xyz/assets/images/android/android\_apk\_build\_process.png)

右图是更详细的构建过程图（_该部分内容没有在官网上找到出处，据说是老版本官网的内容_）

可以很明显的看到，这里面有七步：

1. 应用资源（res文件、assets文件、AndroidManifest.xml以及android.jar）通过 **aapt** 生成`R.java`文件以及打包好的资源文件
2. AIDL文件通过 **aidl** 生成对应的Java文件
3. 源码文件、`R.java`文件以及AIDL生成的Java文件通过 **javac** 编译成`.class`文件
4. 第3步生成的`.class`文件以及第三方库中的`.class`文件通过 **dx** 处理生成`classes.dex`文件
5. 打包好的资源文件、上一步生成的`classes.dex`文件、第三方库中的资源文件以及`.so`文件等其他资源通过 **apkbuilder** 生成未签名的`.apk`文件
6. 调用 **jarsigner** 对上面未签名`.apk`进行签名
7. 调用 **zipalign** 对签名后的`.apk`进行对齐处理

Warning

注意：`apkbuilder`是一个调用了`tools/lib/sdklib.jar`里面`com.android.sdklib.build.` `ApkBuilderMain`的脚本，在某次sdk更新之后脚本被删除了，但是调用还在。\
关于`jarsigner`与脚本`apksigner`，这两着的差别在于V1签名和V2签名；`apksigner`支持V2签名。[Android-APK签名工具-jarsigner和apksigner](https://www.jianshu.com/p/53078d03c9bf)

编译工具路径可以参考：[Android SDK Build Tools](https://developer.android.com/studio/command-line#tools-build)\
编译流程可以参考：[Android逆向分析(2) APK的打包与安装](http://blog.zhaiyifan.cn/2016/02/13/android-reverse-2/)

点击查看[更详细的构建流程](https://blog.yorek.xyz/assets/images/android\_build\_process\_detail.png)

### ProGuard & R8[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#proguard-r8) <a href="#proguard-r8" id="proguard-r8"></a>

在App构建的过程中，生成`.dex`文件之前，还会经过ProGuard或者R8的处理。这一步可以将App和Library中没有使用的代码、资源文件进行移除（又称Tree Shaking）；同时对于需要使用的代码，会使用短名称混淆这些类、字段和方法。

> 本节参考文献： [Shrink, obfuscate, and optimize your app - Android Developers](https://developer.android.com/studio/build/shrink-code)\
> 截止2019-09-02，英语语言介绍的是R8编译器，中文文档还没有更新，所以还是ProGuard编译器。

要尽可能减小 APK 文件，我们应该启用\*压缩\*来移除 release build 中未使用的代码和资源。在启用压缩时，我们还可以从 _混淆_ （缩短类和成员的名称）和 _优化_ （采用更积极的策略来进一步缩小应用程序的大小）中获益。

当我们的项目使用Android Gradle plugin 3.4.0或以上时，插件不再使用ProGuard编辑器来执行编译期代码优化。取而代之的是R8编译器，其将处理以下编译期任务：

* **代码压缩（Code shrinking or tree-shaking）** 检测并安全地从应用程序及其依赖库中删除未使用的类、字段、方法和属性（这使其成为处理[64k引用限制](https://developer.android.com/studio/build/multidex.html)的有用工具）。例如，如果我们只使用了依赖库的少量API，则压缩可以识别我们应用中未使用的库代码，并从我们的应用中只删除这部分代码。
* **资源压缩（Resource shrinking）** 从打包的应用程序中删除未使用的资源，包括应用程序依赖库中未使用的资源。它与代码压缩一起使用，这样一旦删除了未使用的代码，也可以安全地删除不再引用的任何资源。
* **混淆（Obfuscation）** 缩短类和类成员的名称，从而减少DEX文件大小。
* **优化（Optimization）** 检查并重写代码，以进一步减小应用程序DEX文件的大小。例如，如果R8检测到从不使用给定if-else语句的 else 分支，则R8将删除 else 分支的代码。

在构建应用程序的 release 版本时，默认情况下，R8会自动执行上述编译时任务。但是，我们可以通过ProGuard规则文件禁用某些任务或自定义R8的行为。事实上，**R8适用于所有现有的ProGuard规则文件**，因此更新Android Gradle插件来使用R8，这不应该要求我们更改现有规则。

#### 启用压缩、混淆以及优化[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_1) <a href="#_1" id="_1"></a>

由于这些编译时优化会增加编译时间，且可能因为没有充分准备规则而会引入Bug。所以，开启这些编译优化任务最好的时间就是在版本发布前的最终版本。在项目级别的`build.gradle`文件中添加如下代码就可以开启压缩、混淆和优化了：

```
android {
    buildTypes {
        release {
            // Enables code shrinking, obfuscation, and optimization for only
            // your project's release build type.
            minifyEnabled true

            // Enables resource shrinking, which is performed by the
            // Android Gradle plugin.
            shrinkResources true

            // Includes the default ProGuard rules files that are packaged with
            // the Android Gradle plugin. To learn more, go to the section about
            // R8 configuration files.
            proguardFiles getDefaultProguardFile(
                    'proguard-android-optimize.txt'),
                    'proguard-rules.pro'
        }
    }
    ...
}
```

**R8配置文件¶**

下面是R8使用的ProGuard配置文件的来源：the sources of ProGuard rules files that R8 uses

| Source                               | Location                                                                                                                                            | Description                                                                                                                                                                                                                                    |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Android Studio                       | **\<module-dir>/proguard-rules.pro**                                                                                                                |                                                                                                                                                                                                                                                |
| Android Gradle Plugin                | Generated by the Android Gradle plugin at compile time.                                                                                             | The Android Gradle plugin generates **proguard-android-optimize.txt**, which includes rules that are useful to most Android projects and enables [@Keep\* annotations](https://developer.android.com/reference/androidx/annotation/Keep.html). |
| Library dependencies                 | <p>AAR libraries: <strong>&#x3C;library-dir>/proguard.txt</strong><br><br>JAR libraries: <strong>&#x3C;library-dir>/META-INF/proguard/</strong></p> | 如果aar库中已经包含了ProGuard文件，R8会自动使用这些规则。然而，我们需要注意，由于ProGuard规则是相加的，因此aar库中的规则也会影响到app中其他部分。                                                                                                                                                         |
| Android Asset Package Tool 2 (AAPT2) | After building your project with **minifyEnabled true: \<module-dir>/build/intermediates/proguard-rules/debug/aapt\_rules.txt**                     | AAPT2 generates keep rules based on references to classes in your app’s manifest, layouts, and other app resources. For example, AAPT2 includes a keep rule for each Activity that you register in your app’s manifest as an entry point.      |
| Custom configuration files           | By default, when you create a new module using Android Studio, the IDE creates **\<module-dir>/proguard-rules.pro** for you to add your own rules.  | You can [include additional configurations](https://developer.android.com/studio/build/shrink-code#add-configuration), and R8 applies them at compile-time.                                                                                    |

当我们将`minifyEnabled`属性设置为`true`时，R8编译器会将上面表格中所有的来源结合到一起。这在排除R8故障的时候非常重要，因为其他编译期的依赖，例如三方库，可能会引发我们不知道的R8行为的改变。

要在构建项目时输出R8应用的所有规则的完整报告，请在模块的`proguard-rules.pro`文件中包含以下内容：

```
// You can specify any path and filename.
-printconfiguration ~/tmp/full-r8-config.txt
```

#### 额外配置[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_2) <a href="#_2" id="_2"></a>

我们对不同的编译变量添加不同的ProGuard文件，在对应的`productFlavor`代码块中。下面的例子给`flavor2`添加了`flavor2-rules.pro`，因此，`flavor2`现在使用3个ProGuard规则了，因为`release`代码块中的rules也会被使用：

```
android {
    ...
    buildTypes {
        release {
            minifyEnabled true
            proguardFiles getDefaultProguardFile(
              'proguard-android-optimize.txt'),
              // List additional ProGuard rules for the given build type here. By default,
              // Android Studio creates and includes an empty rules file for you (located
              // at the root directory of each module).
              'proguard-rules.pro'
        }
    }
    flavorDimensions "version"
    productFlavors {
        flavor1 {
          ...
        }
        flavor2 {
            proguardFile 'flavor2-rules.pro'
        }
    }
}
```

**代码压缩¶**

要缩小应用程序的代码，R8首先根据组合的配置文件集确定应用程序代码中的所有入口点（_entry point_）。这些入口点包括Android平台可用于打开应用程序的 Activity 或 Service 的所有类。从每个入口点开始，R8检查应用程序的代码，以构建应用程序可能在运行时访问的所有方法，成员变量和其他类的图。未连接到该图的代码被视为不能到达的（_unreachable_），可能会从应用中删除。

下图显示了一个具有运行时依赖库的应用程序。在检查应用程序代码时，R8确定可以从`MainActivity.class`入口点抵达方法`foo()`、`faz()`和`bar()`。但是，我们的应用程序在运行时从不使用类`OkayApi.class`或其方法`baz()`，因此R8在压缩应用程序时会删除该代码。

![At compile-time, R8 builds a graph based on your project's combined keep rules to determine unreachable code.](https://blog.yorek.xyz/assets/images/android/tree-shaking.png)At compile-time, R8 builds a graph based on your project's combined keep rules to determine unreachable code.

R8通过项目的R8配置文件中的`-keep`规则确定入口点。也就是说，keep 规则指定R8在压缩应用程序时不应丢弃的类，R8将这些类视为应用程序的可能入口点。Android Gradle插件和AAPT2会自动生成大多数应用项目所需的保留规则，例如activities、views和services。但是，如果我们需要用其他keep规则来自定义此默认行为，这也是支持的。

#### 自定义需要keep的代码[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#keep) <a href="#keep" id="keep"></a>

对于大多数情况，默认的ProGuard规则文件（`proguard-android-optimize.txt`）足以让R8删除未使用的代码。但是，某些情况很难让R8正确分析，并且可能会删除我们的应用实际需要的代码。可能错误删除代码的一些示例包括：

* 当应用调用的方法来自 Java 原生接口 (JNI) 时
* 当应用在运行时（例如使用反射）操作代码时

要修复错误并强制R8保留某些代码，请在ProGuard规则文件中添加`-keep`行。例如：

```
-keep public class MyClass
```

或者，我们可以将`@Keep`注释添加到要保留的代码中。在类上添加`@Keep`会使整个类保持原样。在方法或字段上添加它将保持方法/字段（及其名称）以及类名完整。请注意，此注释仅在使用AndroidX注释库时以及包含随Android Gradle插件打包的ProGuard规则文件时才可用。

在使用`-keep`选项时，有很多注意事项需要我们注意，更多关于这方面的内容可以阅读[ProGuard Manual](https://www.guardsquare.com/en/products/proguard/manual/usage)。[TroubleShooting](https://www.guardsquare.com/en/products/proguard/manual/troubleshooting)这一小节列出了我们可能遇到的常见问题。

**压缩资源¶**

资源压缩只与代码压缩协同工作。在代码压缩器删除所有未使用的代码之后，资源压缩器可以识别应用仍在使用的资源。这在我们添加包含资源的代码库时体现得尤为明显 ------ 我们必须移除未使用的库代码，使库资源变为未引用资源，才能通过资源压缩器将它们移除。

在`build.gradle`文件中（在用于代码压缩的 `minifyEnabled` 旁边），将`shrinkResources`属性设置为`true`，就可以开启资源压缩了。

```
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
}
```

#### 自定义需要keep的资源[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#keep\_1) <a href="#keep_1" id="keep_1"></a>

如果我们有一些资源想要keep或者discard，我们可以在`/res/raw/keep.xml`文件中进行声明。`tools:keep`和`tools:discard`属性都接收以逗号(“,”)分隔的资源名字的列表，资源名字可以使用星号(“\*”)作为一个通配符。

比如：

```
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/l_used*_c,@layout/l_used_a,@layout/l_used_b*"
    tools:discard="@layout/unused2" />
```

指定要舍弃的资源可能看似愚蠢，因为我们本可将它们删除，但在使用构建变体时，这样做可能很有用。例如，如果我们明知给定资源表面上会在代码中使用（并因此不会被压缩器移除），但实际不会用于给定构建变体，就可以将所有资源放入公用项目目录，然后为每个构建变体创建一个不同的 keep.xml 文件。构建工具也可能无法根据需要正确识别资源，这是因为编译器会添加内联资源 ID，而资源分析器可能不知道真正引用的资源和恰巧具有相同值的代码中的整数值之间的差别。

#### 启用严格引用检查[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_5) <a href="#_5" id="_5"></a>

通常，资源压缩器可以准确地确定是否使用了资源。但是，如果我们的代码调用`Resources.getIdentifier()`（或者如果我们的任何库执行此操作 ------ AppCompat库执行此操作），则表示我们的代码将根据动态生成的字符串查找资源名称。当我们执行这一调用时，默认情况下资源压缩器会采取防御性行为，将所有具有匹配名称格式的资源标记为可能已使用，无法移除。

举个例子，下面的代码会导致所有`img_`前缀的资源被标记为已使用。

```
val name = String.format("img_%1d", angle + 1)
val res = resources.getIdentifier(name, "drawable", packageName)
```

资源压缩器还会浏览代码以及各种 `res/raw/` 资源中的所有字符串常量，寻找格式类似于 `file:///android_res/drawable//ic_plus_anim_016.png` 的资源网址。如果它找到与其类似的字符串，或找到其他看似可用来构建与其类似的网址的字符串，则不会将它们移除。

这些是默认启用的安全压缩模式的示例。但是，我们可以关闭这种“有备无患”处理，并指定资源压缩器只保留其确定已使用的资源。为此，请在`keep.xml`文件中将`shrinkMode`设置为`strict`，如下所示：

```
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="strict" />
```

如果我们确实启用了严格压缩模式，并且代码也引用了包含动态生成字符串的资源，如上所示，那么我们必须使用通过`tools:keep`属性手动keep这些资源。

#### 移除无用的[替代资源](https://developer.android.com/guide/topics/resources/providing-resources.html#AlternativeResources)[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_6) <a href="#_6" id="_6"></a>

下面的代码片段展示了怎么样将语言资源限定为仅支持英语和法语：

```
android {
    defaultConfig {
        ...
        resConfigs "en", "fr"
    }
}
```

同理，我们也可以利用 APK 拆分为不同设备构建不同的 APK，自定义在 APK 中包括的屏幕密度或 ABI 资源。

#### 合并重复资源[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_7) <a href="#_7" id="_7"></a>

默认情况下，Gradle 还会合并同名资源，例如可能位于不同资源文件夹中的同名drawables。这一行为不受 `shrinkResources` 属性控制，也无法停用，因为在有多个资源匹配代码查询的名称时，有必要利用这一行为来避免错误。

只有在两个或更多个文件具有完全相同的资源名称、类型和限定符时，才会进行资源合并。Gradle 会在重复项中选择其视为最佳选择的文件（根据下述优先顺序），并只将这一个资源传递给 AAPT，以供在 APK 文件中分发。

Gradle 会在下列位置寻找重复资源：

* 与 main source set 关联的主资源，一般位于 `src/main/res/` 中。
* 变量overlay，来自构建类型和构建flavors。
* 库项目依赖项。

Gradle 会按以下级联优先顺序合并重复资源：

依赖项 → 主资源 → 构建flavor → 构建类型

例如，如果某个重复资源同时出现在主资源和构建flavor中，Gradle 会选择构建flavor中的重复资源。

如果完全相同的资源出现在同一源集中，Gradle 无法合并它们，并且会发出资源合并错误。如果我们在 `build.gradle` 文件的 `sourceSet` 属性中定义了多个源集，则可能会发生这种情况，例如，如果 `src/main/res/` 和 `src/main/res2/` 包含完全相同的资源，就可能会发生这种情况。

**代码混淆¶**

混淆的目的是通过缩短应用程序的类，方法和字段的名称来减少应用程序的大小。\
虽然混淆不会从我们的应用程序中删除代码，但在具有索引许多类、方法和字段的DEX文件的应用程序中可以看到显著的减小。

此外，如果我们的代码依赖于应用程序的方法和类的可预测命名 ------ 例如，在使用反射时，我们应该将这些签名视为入口点并为它们指定保留规则。那些保留规则告诉R8不仅要将该代码保留在应用程序的最终DEX中，还要保留其原始命名。

**代码优化¶**

为了进一步缩小我们的应用程序，R8会在更深层次上检查我们的代码，以删除更多未使用的代码，或者在可能的情况下重写代码以使其更简洁。下面是几个优化的例子：

* 如果R8检测到从不使用给定if-else语句的 else 分支，则R8将删除 else 分支的代码
* 如果我们的代码仅在一个地方调用方法，R8可能会删除该方法并在单个调用位置内联它。
* 如果R8确定一个类只有一个唯一的子类，并且该类本身未实例化（例如，一个抽象基类仅由一个具体的实现类使用），那么R8可以组合这两个类并从app中删除一个类。
* 要了解更多信息，请阅读Jake Wharton撰写的[R8优化博客文章](https://jakewharton.com/blog/)。

R8不允许我们禁用或启用零碎的优化，或修改优化的行为。实际上，R8忽略了任何试图修改默认优化的ProGuard规则，例如`-optimizations`和`-optimizepasses`。此限制很重要，因为随着R8的不断改进，维护标准的优化行为有助于Android Studio团队轻松排除故障并解决我们可能遇到的任何问题。

#### 启用更激进的优化[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#\_10) <a href="#_10" id="_10"></a>

R8包含一组默认情况下未启用的其他优化。我们可以通过在项目的`gradle.properties`文件中包含以下内容来启用这些其他优化：

```
android.enableR8.fullMode=true
```

由于额外的优化使R8的行为与ProGuard不同，因此它们可能要求我们包含其他ProGuard规则以避免运行时问题。例如，假设我们的代码通过Java Reflection API引用了一个类。默认情况下，R8假定我们打算在运行时检查和操作该类的对象 ------ 即使我们的代码实际上没有 ------ 因此它会自动保留该类及其静态初始化程序。但是，当使用“完整模式”时，R8不会做出这种假设，如果R8断言你的代码在运行时从不使用该类，它会从你应用程序的最终DEX中删除该类。也就是说，如果要保留类及其静态初始化程序，则需要在规则文件中包含keep规则才能执行此操作。

### gradlew命令[¶](https://blog.yorek.xyz/android/paid/zsxq/week22-android-studio-build/#gradlew) <a href="#gradlew" id="gradlew"></a>

可以通过如下命令获取编译log：

```
./gradlew aR --info > ~/Downloads/log.txt
```

从中可以提取`assembleRelease`的子任务（日志有经过美化）：

```
Tasks to be executed: [
  task ':app:preBuild', 
  task ':app:extractProguardFiles', 
  task ':app:preReleaseBuild', 
  task ':app:compileReleaseAidl', 
  task ':app:compileReleaseRenderscript', 
  task ':app:checkReleaseManifest', 
  task ':app:generateReleaseBuildConfig', 
  task ':app:prepareLintJar', 
  task ':app:generateReleaseSources', 
  task ':app:dataBindingExportBuildInfoRelease', 
  task ':app:dataBindingMergeDependencyArtifactsRelease', 
  task ':app:generateReleaseResValues', 
  task ':app:generateReleaseResources', 
  task ':app:mergeReleaseResources', 
  task ':app:transformDataBindingBaseClassLogWithDataBindingMergeGenClassesForRelease', 
  task ':app:dataBindingGenBaseClassesRelease', 
  task ':app:dataBindingExportFeaturePackageIdsRelease', 
  task ':app:mainApkListPersistenceRelease', 
  task ':app:createReleaseCompatibleScreenManifests', 
  task ':app:processReleaseManifest', 
  task ':app:processReleaseResources', 
  task ':app:kaptGenerateStubsReleaseKotlin', 
  task ':app:kaptReleaseKotlin', 
  task ':app:compileReleaseKotlin', 
  task ':app:javaPreCompileRelease', 
  task ':app:compileReleaseJavaWithJavac', 
  task ':app:compileReleaseNdk', 
  task ':app:compileReleaseSources', 
  task ':app:lintVitalRelease', 
  task ':app:mergeReleaseShaders', 
  task ':app:compileReleaseShaders', 
  task ':app:generateReleaseAssets', 
  task ':app:mergeReleaseAssets', 
  task ':app:validateSigningRelease', 
  task ':app:signingConfigWriterRelease', 
  task ':app:processReleaseJavaRes', 
  task ':app:transformResourcesWithMergeJavaResForRelease', 
  task ':app:transformClassesAndResourcesWithProguardForRelease', 
  task ':app:transformClassesWithDexBuilderForRelease', 
  task ':app:transformClassesWithMultidexlistForRelease', 
  task ':app:transformDexArchiveWithDexMergerForRelease', 
  task ':app:transformClassesAndDexWithShrinkResForRelease', 
  task ':app:mergeReleaseJniLibFolders', 
  task ':app:transformNativeLibsWithMergeJniLibsForRelease', 
  task ':app:transformNativeLibsWithStripDebugSymbolForRelease', 
  task ':app:packageRelease', 
  task ':app:assembleRelease'
]
```

最后更新: 2020年3月13日\

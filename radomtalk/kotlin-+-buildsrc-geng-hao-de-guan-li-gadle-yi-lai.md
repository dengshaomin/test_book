# Kotlin + buildSrc：更好的管理Gadle依赖

为了充分利用Android Plugin for Gradle 3.0+的优点，将Android项目拆分成多个module的做法越来越常见。然而，随着module数量的增多，我们很快就会遇到依赖管理的混乱问题。

**管理Gradle依赖的三种不同方法：**

1. 手动管理
2. 使用Google推荐的“ext”
3. **Kotlin + buildSrc**

### 1) 手动管理

这是一种大多数人在采用的管理依赖的方法，但每次升级依赖库时都需要做大量的手动更改。

**module\_a/build.gradle**

```
implementation "com.android.support:support-annotations:27.0.2"
implementation "com.android.support:appcompat-v7:27.0.2"
implementation "com.squareup.retrofit2:retrofit:2.3.0"
implementation "com.squareup.retrofit2:adapter-rxjava2:2.3.0"
implementation "io.reactivex.rxjava2:rxjava:2.1.9"
复制代码
```

**module\_b/build.gradle**

```
implementation "com.android.support:support-annotations:27.0.2"
implementation "com.android.support:appcompat-v7:27.0.2"
implementation "com.squareup.retrofit2:retrofit:2.3.0"
implementation "com.squareup.retrofit2:adapter-rxjava2:2.3.0"
implementation "io.reactivex.rxjava2:rxjava:2.1.9"
复制代码
```

这里存在许多重复的配置，而且当你的项目有很多module时很难管理依赖库的版本更新。

### Google推荐：使用gradle的extra属性

Google在[Android官方文档](https://developer.android.com/studio/build/gradle-tips#configure-project-wide-properties)中推荐这种管理依赖的方法。许多项目例如ButterKnife、Picasso等都在使用这种方法。

此方法非常适用于更新support library的版本，因为每个support library都具有相同的版本号，你只需要在一个地方更改它就行了。 Retrofit等其它第三方库也是如此。

**Root-level build.gradle**

```
ext {
  versions = [
    support_lib: "27.0.2",
    retrofit: "2.3.0",
    rxjava: "2.1.9"
  ]
  libs = [
    support_annotations: "com.android.support:support-annotations:${versions.support_lib}",
    support_appcompat_v7: "com.android.support:appcompat-v7:${versions.support_lib}",
    retrofit :"com.squareup.retrofit2:retrofit:${versions.retrofit}",
    retrofit_rxjava_adapter: "com.squareup.retrofit2:adapter-rxjava2:${versions.retrofit}",
    rxjava: "io.reactivex.rxjava2:rxjava:${versions.rxjava}"
  ]
}
复制代码
```

**module\_a/build.gradle**

```
implementation libs.support_annotations
implementation libs.support_appcompat_v7
implementation libs.retrofit
implementation libs.retrofit_rxjava_adapter
implementation libs.rxjava
复制代码
```

**module\_b/build.gradle**

```
implementation libs.support_annotations
implementation libs.support_appcompat_v7
implementation libs.retrofit
implementation libs.retrofit_rxjava_adapter
implementation libs.rxjava
复制代码
```

这种方法是手动管理的一大进步，但是缺少IDE的支持，更准确的说是在更新依赖库的时候IDE不能自动补全。

### Kotlin + buildSrc == Android Studio Autocomplete 😎 🎉

![](<../.gitbook/assets/image (243).png>)

您需要在您的项目里创建一个**buildSrc**模块，然后编写**kotlin**代码来管理依赖库，使得IDE支持自动补全。

#### [Gradle文档](https://docs.gradle.org/current/userguide/organizing\_build\_logic.html#sec:build\_sources)中有这样一段话:

> 当你运行Gradle时，它会检查项目中是否存在一个名为`buildSrc`的目录。然后Gradle会自动编译并测试这段代码，并将其放入构建脚本的类路径中。您不需要提供任何进一步的操作提示。

#### 你只需要在buildSrc module中新建两个文件:

1. build.gradle.kts
2. 编写Kotlin代码的文件 (本文中是指`Dependencies.kt`)

![](<../.gitbook/assets/image (218).png>)

#### buildSrc/build.gradle.kts:

```
plugins {
    `kotlin-dsl`
}
复制代码
```

#### buildSrc/src/main/java/Dependencies.kt

```
object Versions {
    val support_lib = "27.0.2"
    val retrofit = "2.3.0"
    val rxjava = "2.1.9"
}

object Libs {
 val support_annotations = "com.android.support:support-annotations:${Versions.support_lib}"
 val support_appcompat_v7 = "com.android.support:appcompat-v7:${Versions.support_lib}"
 val retrofit = "com.squareup.retrofit2:retrofit:${Versions.retrofit}"
 val retrofit_rxjava_adapter = "com.squareup.retrofit2:adapter-rxjava2:${Versions.retrofit}"
 val rxjava = "io.reactivex.rxjava2:rxjava:${Versions.rxjava}"
}
复制代码
```

经过上面两个步骤后，执行一次Gradle Sync任务，现在我们可以在Android Studio中访问Dependencies.kt中任何值了。

看起来结果与“ext”非常相似，但是它支持自动补全和单击跳转。

**module\_a/build.gradle**

```
implementation Libs.support_annotations
implementation Libs.support_appcompat_v7
implementation Libs.retrofit
implementation Libs.retrofit_rxjava_adapter
implementation Libs.rxjava
复制代码
```

**module\_a/build.gradle**

```
implementation Libs.support_annotations
implementation Libs.support_appcompat_v7
implementation Libs.retrofit
implementation Libs.retrofit_rxjava_adapter
implementation Libs.rxjava
复制代码
```

### 结束语

我强烈推荐您使用“Kotlin + buildSrc”的方法。它支持自动补全和单击跳转，使得您无需在文件之间手动来回切换，方便你更好的管理Gradle依赖。

### 动手实践：

**新建的module名称必须为buildSrc**

一开始我按照作者原文的描述，在Android Studio里右键单击项目，New 出一个名为buildSrc的Android Library，试了好几遍都提示“**Gradle sync failed: Plugin with id 'com.android.library' not found**”的错误。

后来我参考[这里](https://zeroturnaround.com/rebellabs/using-buildsrc-for-custom-logic-in-gradle-builds/)的做法，手动创建了buildSrc这个模块。步骤如下：

1. 在项目**根目录**下新建一个名为**buildSrc**的文件夹(与项目里的app文件夹同级)。
2. 在buildSrc文件夹里创建名为**build.gradle.kts**的文件，文件内容参考之前的描述。
3. 在buildSrc文件夹里创建**src/main/java**文件夹，如下图所示。并在该文件夹下创建Dependencies.kt文件，文件内容参考之前的描述。

![](https://user-gold-cdn.xitu.io/2018/6/4/163ca425907ed5a5?imageView2/0/w/1280/h/960/format/webp/ignore-error/1) 4. build一遍你的项目，然后重启你的Android Studio，项目里就会多出一个名为buildSrc的module。\

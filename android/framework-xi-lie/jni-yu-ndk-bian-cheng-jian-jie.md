# JNI与NDK编程简介

### 1 JNI与NDK简介[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#1-jnindk) <a href="#1-jnindk" id="1-jnindk"></a>

JNI指Java Native Interface，它可以使Java调用C/C++代码。

NDK是Android提供的一个工具集合，通过NDK可以在Android中更加方便的通过JNI来访问本地代码。NDK还提供了交叉编译器，开发人员只需要简单的修改mk文件就可以生成特定CPU平台的动态库。NDK的好处有如下几点：

1. **提高代码安全性**。由于so库反编译比较困难，因此NDK提高了Android程序的安全性。
2. **可以很方便的使用目前已拥有的C/C++开源库。**
3. **便于平台间的移植。** 通过C/C++实现的动态库可以很方便的在其他平台上使用。
4. **提高程序在某些特定情况下的执行效率**，但不能明显提升Android程序的性能。

### 2 JNI的开发流程[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#2-jni) <a href="#2-jni" id="2-jni"></a>

JNI的开发流程有如下几步：

1. 在Java中声明native方法
2. 编译Java源文件得到class文件，然后通过javah命令导出JNI的头文件
3. 实现JNI方法
4. 编译so库并在Java中调用

#### 2.1 在Java中声明native方法[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#21-javanative) <a href="#21-javanative" id="21-javanative"></a>

创建一个类，声明两个native方法:get和set(String)。

```
package com.example;

import java.lang.System;

public class JniTest {

  public static void main(String[] args) {
    JniTest jniTest = new JniTest();
    System.out.println(jniTest.get());
    jniTest.set("hello world");
  }

  public native String get();
  public native String set(String str);
}
```

#### 2.2 编译Java源文件得到class文件，然后通过javah命令导出JNI的头文件[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#22-javaclassjavahjni) <a href="#22-javaclassjavahjni" id="22-javaclassjavahjni"></a>

```
javac com/example/JniTest.java
javah com.example.JniTest
```

![jni编译Java源文件](https://blog.yorek.xyz/assets/images/android/jni%E7%BC%96%E8%AF%91Java%E6%BA%90%E6%96%87%E4%BB%B6.png)

输入上面命令后会在当前目录生成`com_example_JniTest.h`文件，它是javah命令生成的。内容如下所示：

```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_example_JniTest */

#ifndef _Included_com_example_JniTest
#define _Included_com_example_JniTest
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_example_JniTest
 * Method:    get
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_get
  (JNIEnv *, jobject);

/*
 * Class:     com_example_JniTest
 * Method:    set
 * Signature: (Ljava/lang/String;)Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_set
  (JNIEnv *, jobject, jstring);

#ifdef __cplusplus
}
#endif
#endif
```

从上面我可以看出，函数的命名格式遵循如下规则：`Java_包名_类名_方法名`。比如Demo中的`get`方法，到这里就变成了`JNIEXPORT jstring JNICALL Java_com_example_JniTest_get(JNIEnv *, jobject);`方法。这里面都是JNI标准定义的类型或者宏：

* JNIEnv\*：表示一个指向JNI环境的指针，可以通过它访问JNI提供的借口方法
* jobject：表示Java对象中的this
* JNIEXPORT和JNICALL：JNI中所指定的宏，可以在`jni.h`头文件中查找到。

#### 2.3 实现JNI方法[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#23-jni) <a href="#23-jni" id="23-jni"></a>

JNI方法是指Java中声明的native方法，这里可以选用C++或者C来实现，它们的实现过程是类似的。\
下面分别选用两者来实现JNI方法。首先，在工程的主目录下创建一个子目录，名称随意，这里选择jni作为子目录的名称，然后将之前通过`javah`生成的头文件`com_example_JniTest.h`复制到jni目录下，接着创建`test.cpp`和`test.c`两个文件，它们的实现如下。

```
// test.cpp
#include "com_example_JniTest.h"
#include <stdio.h>

/*
 * Class:     com_example_JniTest
 * Method:    get
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_get(JNIEnv *env, jobject thiz) {
  printf("invoke get in c++\n");
  return env->NewStringUTF("Hello from JNI !");
}

/*
 * Class:     com_example_JniTest
 * Method:    set
 * Signature: (Ljava/lang/String;)Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_set(JNIEnv *env, jobject thiz, jstring string) {
  printf("invoke set from c++\n");
  char* str = (char*)env->GetStringUTFChars(string, NULL);
  printf("%s\n", str);
  env->ReleaseStringUTFChars(string, str);
}
```

```
// test.c
#include "com_example_JniTest.h"
#include <stdio.h>

/*
 * Class:     com_example_JniTest
 * Method:    get
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_get(JNIEnv *env, jobject thiz) {
  printf("invoke get in c\n");
  return (*env)->NewStringUTF(env, "Hello from JNI !");
}

/*
 * Class:     com_example_JniTest
 * Method:    set
 * Signature: (Ljava/lang/String;)Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_example_JniTest_set(JNIEnv *env, jobject thiz, jstring string) {
  printf("invoke set from c\n");
  char* str = (char*)(*env)->GetStringUTFChars(env, string, NULL);
  printf("%s\n", str);
  (*env)->ReleaseStringUTFChars(env, string, str);
}
```

#### 2.4 编译so库并在Java中调用[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#24-sojava) <a href="#24-sojava" id="24-sojava"></a>

编译命令与操作系统相关，此处作者用的是OSX系统。

编译的命令如下： 在jni目录下

```
gcc -I /System/Library/Frameworks/JavaVM.framework/Headers/ -c test.c
gcc -dynamiclib -o libjni-test.jnilib test.o
```

最后会在jni目录下生成`libjni-test.jnilib`文件。

然后在我们的Java文件中加载动态库:

```
package com.example;

import java.lang.System;

public class JniTest {

  static {
    System.loadLibrary("jni-test");
  }

  public static void main(String[] args) {
    JniTest jniTest = new JniTest();
    System.out.println(jniTest.get());
    jniTest.set("hello world");
  }

  public native String get();
  public native String set(String str);
}
```

最后重新编译Java文件并通过java命令执行：

```
cd ..
javac com/example/JniTest.java
java -Djava.library.path=jni com.example.JniTest
```

![jni执行结果.png](https://blog.yorek.xyz/assets/images/android/jni%E6%89%A7%E8%A1%8C%E7%BB%93%E6%9E%9C.png)

### 3 NDK的开发流程[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#3-ndk) <a href="#3-ndk" id="3-ndk"></a>

NDK开发的基本框架官方已经很方便的支持了，查看[Getting Started with the NDK](https://developer.android.com/studio/projects/add-native-code.html#create-sources)

#### 3.1 NDK使用实例[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#31-ndk) <a href="#31-ndk" id="31-ndk"></a>

1. 准备一个简单的Android项目
2. 创建jni文件夹，如下所示：\
   选中app module -> 右键New -> Folder -> JNI Folder\
   [![创建jni文件夹](https://blog.yorek.xyz/assets/images/android/ndk\_create\_jni\_dir.png)](https://blog.yorek.xyz/assets/images/android/ndk\_create\_jni\_dir.png)
3.  准备一个JNI Java层文件\
    NativeTest.java

    ```
    package com.example.nativedemo;

    public class NativeTest {
        public static native String getHelloWorld();
    }
    ```
4. 编译一下，这时候会为NativeTest生成class文件，目录在`app/build/intermediates/javac/debug/classes/com/example/nativedemo/NativeTest.class`
5.  在项目根目录下执行下面命令，生成对应的jni头文件到jni目录下\


    ```
    javah -d app/src/main/jni -cp app/build/intermediates/javac/debug/classes com.example.nativedemo.NativeTest
    ```

    生成的头文件`com_example_nativedemo_NativeTest.h`如下：

    ```
    /* DO NOT EDIT THIS FILE - it is machine generated */
    #include <jni.h>
    /* Header for class com_example_nativedemo_NativeTest */

    #ifndef _Included_com_example_nativedemo_NativeTest
    #define _Included_com_example_nativedemo_NativeTest
    #ifdef __cplusplus
    extern "C" {
    #endif
    /*
    * Class:     com_example_nativedemo_NativeTest
    * Method:    getHelloWorld
    * Signature: ()Ljava/lang/String;
    */
    JNIEXPORT jstring JNICALL Java_com_example_nativedemo_NativeTest_getHelloWorld
      (JNIEnv *, jclass);

    #ifdef __cplusplus
    }
    #endif
    #endif
    ```
6.  在jni目录下实现该头文件：\


    ```
    // com_example_nativedemo_NativeTest.cpp
    #include <jni.h>
    #include "com_example_nativedemo_NativeTest.h"


    JNIEXPORT jstring JNICALL Java_com_example_nativedemo_NativeTest_getHelloWorld
      (JNIEnv *env, jclass jz)
    {
      return env->NewStringUTF("Hello, World from JNI!");
    }
    ```
7.  在jni目录下放置`Application.mk`和`Android.mk`两个文件\


    ```
     # Application.mk
     #APP_ABI := armeabi armeabi-v7a x86 mips mips-r2 mips-r2-sf
     APP_ABI := armeabi-v7a arm64-v8a x86 x86_64
     #APP_ABI := armeabi-v7a arm64-v8a
     APP_PLATFORM := android-14
     #APP_OPTIM := debug


     # Android.mk
     LOCAL_PATH := $(call my-dir)
     include $(CLEAR_VARS)

     LOCAL_MODULE := nativetest

     LOCAL_SRC_FILES := \
           com_example_nativedemo_NativeTest.cpp \

     include $(BUILD_SHARED_LIBRARY)
    ```
8. 在jni目录下执行`ndk-build`命令，执行完毕后就会生成指定的so库
9.  在build.gradle中配置so的目录：

    ```
     android {
         ...
         sourceSets {
             main() {
                 jniLibs.srcDirs = ['src/main/libs']
                 jni.srcDirs = [] //屏蔽掉默认的jni编译生成过程
             }
         }
     }
    ```
10. 最后在NativeTest.java中load一下就可以使用了：

    ```
     public class NativeTest {

         static {
             System.loadLibrary("nativetest");
         }

         public static native String getHelloWorld();
     }
    ```

### 4 JNI的数据类型和类型签名[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#4-jni) <a href="#4-jni" id="4-jni"></a>

JNI的数据包含两大种：基本类型和引用类型，基本类型主要有`jboolean`、`jchar`、`jint`等，它们和Java中的数据类型的对应关系如下：

| JNI类型    | Java类型  | 描述       |
| -------- | ------- | -------- |
| jboolean | boolean | 无符号8位整型  |
| jbyte    | byte    | 有符号8位整型  |
| jchar    | char    | 无符号16位整型 |
| jshort   | short   | 有符号16位整型 |
| jint     | int     | 32位整型    |
| jlong    | long    | 64位整型    |
| jfloat   | float   | 32位浮点型   |
| jdouble  | double  | 64位浮点型   |
| void     | void    | 无类型      |

JNI中的引用类型主要有类、对象和数组，它们和Java中的引用类型的对应关系如表所示：

| JNI类型         | Java类型     | 描述        |
| ------------- | ---------- | --------- |
| jobject       | Object     | Object类型  |
| jclass        | Class      | Class类型   |
| jstring       | String     | 字符串       |
| jobjectArray  | Object\[]  | 对象数组      |
| jbooleanArray | boolean\[] | boolean数组 |
| jbyteArray    | byte\[]    | byte数组    |
| jcharArray    | char\[]    | char数组    |
| jshortArray   | short\[]   | short数组   |
| jintArray     | int\[]     | int数组     |
| jlongArray    | long\[]    | long数组    |
| jfloatArray   | float\[]   | float数组   |
| jdoubleArray  | double\[]  | double数组  |
| jthrowable    | Throwable  | Throwable |

JNI的类型签名标识了一个特定的Java类型，这个类型可以是类和方法，也可以是数据类型。类的签名比较简单，它采用`L+包名+类名+;`的形式，只需要将其中的"."替换为"/"即可，比如`java.lang.String`的签名为`Ljava/lang/System;`，末尾的";"也是签名的一部分。

基本数据类型的签名采用一系列大写字母来表示：

| Java类型  | 签名 |
| ------- | -- |
| boolean | Z  |
| byte    | B  |
| char    | C  |
| short   | S  |
| int     | I  |
| long    | J  |
| float   | F  |
| Double  | D  |
| void    | V  |

从上表可以看出，基本数据类型的签名都是有规律的，一般为首字母的大写，但是boolean除外，因为B已经被byte占用了；而long的签名不是L的原因是类的签名是用L表示的。

对象和数组的签名稍微复杂点。\
对于对象来说，它的签名就是对象所属的类的签名，比如String对象，它的签名为`Ljava/lang/String;`。\
对于数组来说，它的签名为`[+类型签名`，比如int数组，其签名就是`[I`。\
对于多位数组来说，它的签名为`n个[+类型签名`，比如`int[][]`的签名就是`[[I`。

方法的签名为`(+参数类型签名+)+返回值类型签名`。比如\
`boolean func1(int a, double b, int[] c)` → `(ID[I)Z`\
`boolean func1(int a, String b, int[] c)` → `(ILjava/lang/String;[I)Z`

### 5 JNI调用Java方法的流程[¶](https://blog.yorek.xyz/android/framework/JNI%E4%B8%8ENDK/#5-jnijava) <a href="#5-jnijava" id="5-jnijava"></a>

JNI调用Java方法的流程是先通过类名找到类，然后在根据方法名找到方法id，最后就可以调用这个方法了。如果调用Java中非静态方法，那么需要构造出来的对象后才能调用它。

**1.调用非静态方法**\


```
JNIEXPORT void JNICALL Java_cn_itcast_ndkcallback_DataProvider_callmethod1
  (JNIEnv * env, jobject obj){
    //1 . 找到java代码的 class文件
    jclass dpclazz = (*env)->FindClass(env, "cn/itcast/ndkcallback/DataProvider");
    if(dpclazz == 0){
        LOGI("find class error");
        return;
    }
    LOGI("find class ");

    //2 寻找class里面的方法
    jmethodID method1 = (*env)->GetMethodID(env, dpclazz, "helloFromJava", "()V");
    if(method1 == 0){
        LOGI("find method1 error");
        return;
    }
    LOGI("find method1 ");
    //3 .调用这个方法
    (*env)->CallVoidMethod(env, obj, method1);
}
```

注意在根据方法名查找id时需要传入方法的签名。

**2.调用静态方法**\


```
JNIEXPORT void JNICALL Java_cn_itcast_ndkcallback_DataProvider_callmethod4
  (JNIEnv * env, jobject obj){
      //1 . 找到java代码的 class文件
      jclass dpclazz = (*env)->FindClass(env, "cn/itcast/ndkcallback/DataProvider");
      if(dpclazz == 0){
          LOGI("find class error");
          return;
      }
      LOGI("find class ");

      //2 寻找class里面的方法
      jmethodID method4 = (*env)->GetStaticMethodID(env, dpclazz, "printStaticStr", "(Ljava/lang/String;)V");
      if(method4 == 0){
          LOGI("find method4 error");
          return;
      }
      LOGI("find method4 ");

      //3.调用一个静态的java方法
      (*env)->CallStaticVoidMethod(env, dpclazz, method4, (*env)->NewStringUTF(env, "static haha in c"));
}
```

最后更新: 2020年1月21日\
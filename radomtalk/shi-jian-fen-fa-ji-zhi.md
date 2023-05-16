---
description: https://www.jianshu.com/p/38015afcdb58
---

# 事件分发机制

目录

![](<../.gitbook/assets/image (101).png>)

## 1. 基础认知

#### 1.1 事件分发的”事件“是指什么？

**答：点击事件（`Touch`事件）**。具体介绍如下：

![](<../.gitbook/assets/image (98).png>)

此处需要特别说明：事件列，即指从手指接触屏幕至手指离开屏幕这个过程产生的一系列事件。一般情况下，事件列都是以DOWN事件开始、UP事件结束，中间有无数的MOVE事件。

![](<../.gitbook/assets/image (61).png>)

#### 1.2 事件分发的本质

**答：将点击事件（MotionEvent）传递到某个具体的`View` & 处理的整个过程**

> 即 事件传递的过程 = 分发过程。

#### 1.3 事件在哪些对象之间进行传递？

**答：Activity、ViewGroup、View**。`Android`的`UI`界面由`Activity`、`ViewGroup`、`View` 及其派生类组成\


![](<../.gitbook/assets/image (34).png>)

![](<../.gitbook/assets/image (18).png>)

#### 1.4 事件分发的顺序

即 事件传递的顺序：`Activity` -> `ViewGroup` -> `View`

> 即：1个点击事件发生后，事件先传到`Activity`、再传到`ViewGroup`、最终再传到 `View`

![](<../.gitbook/assets/image (22).png>)

#### 1.5 事件分发过程由哪些方法协作完成？

**答：dispatchTouchEvent() 、onInterceptTouchEvent()和onTouchEvent()**

![](<../.gitbook/assets/image (200).png>)

> 下文会对这3个方法进行详细介绍

#### 1.6 总结

![](<../.gitbook/assets/image (82).png>)

* 至此，相信大家已经对 `Android`的事件分发有了感性的认知
* 下面，我将详细介绍`Android`事件分发机制

***

## 2. 事件分发机制流程概述

`Android`事件分发流程 = **Activity -> ViewGroup -> View**

> 即：1个点击事件发生后，事件先传到`Activity`、再传到`ViewGroup`、最终再传到 `View`

即要想充分理解Android分发机制，本质上是要理解：

1. `Activity`对点击事件的分发机制
2. `ViewGroup`对点击事件的分发机制
3. `View`对点击事件的分发机制

下面，我将通过源码，全面解析 **事件分发机制**

> 即按顺序讲解：`Activity`事件分发机制、`ViewGroup`事件分发机制、`View`事件分发机制

***

## 3. 事件分发机制流程详细分析

主要包括：`Activity`事件分发机制、`ViewGroup`事件分发机制、`View`事件分发机制

### 流程1：Activity的事件分发机制

Android事件分发机制首先会将点击事件传递到Activity中，具体是执行dispatchTouchEvent()进行事件分发。

#### 源码分析

```csharp
/**
  * 源码分析：Activity.dispatchTouchEvent（）
  */ 
  public boolean dispatchTouchEvent(MotionEvent ev) {

    // 仅贴出核心代码

    // ->>分析1
    if (getWindow().superDispatchTouchEvent(ev)) {

        return true;
        // 若getWindow().superDispatchTouchEvent(ev)的返回true
        // 则Activity.dispatchTouchEvent（）就返回true，则方法结束。即 ：该点击事件停止往下传递 & 事件传递过程结束
        // 否则：继续往下调用Activity.onTouchEvent

    }
    // ->>分析3
    return onTouchEvent(ev);
  }

/**
  * 分析1：getWindow().superDispatchTouchEvent(ev)
  * 说明：
  *     a. getWindow() = 获取Window类的对象
  *     b. Window类是抽象类，其唯一实现类 = PhoneWindow类
  *     c. Window类的superDispatchTouchEvent() = 1个抽象方法，由子类PhoneWindow类实现
  */
  @Override
  public boolean superDispatchTouchEvent(MotionEvent event) {

      return mDecor.superDispatchTouchEvent(event);
      // mDecor = 顶层View（DecorView）的实例对象
      // ->> 分析2
  }

/**
  * 分析2：mDecor.superDispatchTouchEvent(event)
  * 定义：属于顶层View（DecorView）
  * 说明：
  *     a. DecorView类是PhoneWindow类的一个内部类
  *     b. DecorView继承自FrameLayout，是所有界面的父类
  *     c. FrameLayout是ViewGroup的子类，故DecorView的间接父类 = ViewGroup
  */
  public boolean superDispatchTouchEvent(MotionEvent event) {

      return super.dispatchTouchEvent(event);
      // 调用父类的方法 = ViewGroup的dispatchTouchEvent()
      // 即将事件传递到ViewGroup去处理，详细请看后续章节分析的ViewGroup的事件分发机制

  }
  // 回到最初的分析2入口处

/**
  * 分析3：Activity.onTouchEvent()
  * 调用场景：当一个点击事件未被Activity下任何一个View接收/处理时，就会调用该方法
  */
  public boolean onTouchEvent(MotionEvent event) {

        // ->> 分析5
        if (mWindow.shouldCloseOnTouch(this, event)) {
            finish();
            return true;
        }
        
        return false;
        // 即 只有在点击事件在Window边界外才会返回true，一般情况都返回false，分析完毕
    }

/**
  * 分析4：mWindow.shouldCloseOnTouch(this, event)
  * 作用：主要是对于处理边界外点击事件的判断：是否是DOWN事件，event的坐标是否在边界内等
  */
  public boolean shouldCloseOnTouch(Context context, MotionEvent event) {

  if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
          && isOutOfBounds(context, event) && peekDecorView() != null) {

        // 返回true：说明事件在边界外，即 消费事件
        return true;
    }

    // 返回false：在边界内，即未消费（默认）
    return false;
  } 
```

#### 源码总结

当一个点击事件发生时，从`Activity`的事件分发开始（`Activity.dispatchTouchEvent()`），流程总结如下：

![](<../.gitbook/assets/image (81).png>)

#### 核心方法总结

主要包括：dispatchTouchEvent()、onTouchEvent() 总结如下

![](<../.gitbook/assets/image (67).png>)

那么，`ViewGroup`的`dispatchTouchEvent()`什么时候返回`true` / `false`？请继续往下看**ViewGroup事件的分发机制**

### 流程2： ViewGroup的事件分发机制

从上面Activity的事件分发机制可知，在Activity.dispatchTouchEvent()实现了将事件从Activity->ViewGroup的传递，ViewGroup的事件分发机制从dispatchTouchEvent()开始。

#### 源码分析

```java
/**
  * 源码分析：ViewGroup.dispatchTouchEvent（）
  */ 
  public boolean dispatchTouchEvent(MotionEvent ev) { 

  // 仅贴出关键代码
  ... 

  if (disallowIntercept || !onInterceptTouchEvent(ev)) {  
  // 分析1：ViewGroup每次事件分发时，都需调用onInterceptTouchEvent()询问是否拦截事件
    // 判断值1-disallowIntercept：是否禁用事件拦截的功能(默认是false)，可通过调用requestDisallowInterceptTouchEvent()修改
    // 判断值2-!onInterceptTouchEvent(ev) ：对onInterceptTouchEvent()返回值取反
        // a. 若在onInterceptTouchEvent()中返回false，即不拦截事件，从而进入到条件判断的内部
        // b. 若在onInterceptTouchEvent()中返回true，即拦截事件，从而跳出了该条件判断
        // c. 关于onInterceptTouchEvent() ->>分析1

  // 分析2
    // 1. 通过for循环，遍历当前ViewGroup下的所有子View
    for (int i = count - 1; i >= 0; i--) {  
        final View child = children[i];  
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE  
                || child.getAnimation() != null) {  
            child.getHitRect(frame);  

            // 2. 判断当前遍历的View是不是正在点击的View，从而找到当前被点击的View
            if (frame.contains(scrolledXInt, scrolledYInt)) {  
                final float xc = scrolledXFloat - child.mLeft;  
                final float yc = scrolledYFloat - child.mTop;  
                ev.setLocation(xc, yc);  
                child.mPrivateFlags &= ~CANCEL_NEXT_UP_EVENT;  

                // 3. 条件判断的内部调用了该View的dispatchTouchEvent()
                // 即 实现了点击事件从ViewGroup到子View的传递（具体请看下面章节介绍的View事件分发机制）
                if (child.dispatchTouchEvent(ev))  { 

                // 调用子View的dispatchTouchEvent后是有返回值的
                // 若该控件可点击，那么点击时dispatchTouchEvent的返回值必定是true，因此会导致条件判断成立
                // 于是给ViewGroup的dispatchTouchEvent()直接返回了true，即直接跳出
                // 即该子View把ViewGroup的点击事件消费掉了

                mMotionTarget = child;  
                return true; 
                      }  
                  }  
              }  
          }  
      }  
    }  

  ...

  return super.dispatchTouchEvent(ev);
  // 若无任何View接收事件(如点击空白处)/ViewGroup本身拦截了事件(复写了onInterceptTouchEvent()返回true)
  // 会调用ViewGroup父类的dispatchTouchEvent()，即View.dispatchTouchEvent()
  // 因此会执行ViewGroup的onTouch() -> onTouchEvent() -> performClick（） -> onClick()，即自己处理该事件，事件不会往下传递
  // 具体请参考View事件分发机制中的View.dispatchTouchEvent()

  ... 

}

/**
  * 分析1：ViewGroup.onInterceptTouchEvent()
  * 作用：是否拦截事件
  * 说明：
  *     a. 返回false：不拦截（默认）
  *     b. 返回true：拦截，即事件停止往下传递（需手动复写onInterceptTouchEvent()其返回true）
  */
  public boolean onInterceptTouchEvent(MotionEvent ev) {  
    
    // 默认不拦截
    return false;

  } 
  // 回到调用原处
```

#### 源码总结

`Android`事件分发传递到Acitivity后，总是先传递到`ViewGroup`、再传递到`View`。流程总结如下：(假设已经经过了Acitivity事件分发传递并传递到ViewGroup)

![](<../.gitbook/assets/image (16).png>)

#### 核心方法总结

主要包括：dispatchTouchEvent()、onTouchEvent() 、onInterceptTouchEvent()总结如下

![](<../.gitbook/assets/image (96).png>)

#### 实例分析

**1. 布局说明**

![](<../.gitbook/assets/image (93).png>)

**2. 测试代码**

布局文件：_activity\_main.xml_

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/my_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:focusableInTouchMode="true"
    android:orientation="vertical">

    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="按钮1" />

    <Button
        android:id="@+id/button2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="按钮2" />

</LinearLayout>
```

核心代码：_MainActivity.java_

```java
public class MainActivity extends AppCompatActivity {

  Button button1,button2;
  ViewGroup myLayout;

  @Override
  public void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.activity_main);

      button1 = (Button)findViewById(R.id.button1);
      button2 = (Button)findViewById(R.id.button2);
      myLayout = (LinearLayout)findViewById(R.id.my_layout);

      // 1.为ViewGroup布局设置监听事件
      myLayout.setOnClickListener(new View.OnClickListener() {
          @Override
          public void onClick(View v) {
              Log.d("TAG", "点击了ViewGroup");
          }
      });

      // 2. 为按钮1设置监听事件
      button1.setOnClickListener(new View.OnClickListener() {
          @Override
          public void onClick(View v) {
              Log.d("TAG", "点击了button1");
          }
      });

      // 3. 为按钮2设置监听事件
      button2.setOnClickListener(new View.OnClickListener() {
          @Override
          public void onClick(View v) {
              Log.d("TAG", "点击了button2");
          }
      });
   }
}
```

**3. 测试结果**

```cpp
// 点击按钮1，输出如下
点击了button1

// 点击按钮2，输出如下
点击了button2

// 点击空白处，输出如下
点击了ViewGroup
```

**4. 结果分析**

* 点击Button时，因为ViewGroup默认不拦截，所以事件会传递到子View Button，于是执行Button.onClick()。
* 此时ViewGroup. dispatchTouchEvent()会直接返回true，所以ViewGroup自身不会处理该事件，于是ViewGroupLayout的dispatchTouchEvent()不会执行，所以注册的onTouch()不会执行，即onTouchEvent() -> performClick() -> onClick()整个链路都不会执行，所以最后不会执行ViewGroup设置的onClick()里。
* 点击空白区域时，ViewGroup. dispatchTouchEvent()里遍历所有子View希望找到被点击子View时找不到，所以ViewGroup自身会处理该事件，于是执行onTouchEvent() -> performClick() -> onClick()，最终执行ViewGroupLayout的设置的onClick()

***

### 流程3：View的事件分发机制

从上面`ViewGroup`事件分发机制知道，`View`事件分发机制从`dispatchTouchEvent()`开始

#### 源码分析

```csharp
/**
  * 源码分析：View.dispatchTouchEvent（）
  */
  public boolean dispatchTouchEvent(MotionEvent event) {  

       
        if ( (mViewFlags & ENABLED_MASK) == ENABLED && 
              mOnTouchListener != null &&  
              mOnTouchListener.onTouch(this, event)) {  
            return true;  
        } 

        return onTouchEvent(event);  
  }
  // 说明：只有以下3个条件都为真，dispatchTouchEvent()才返回true；否则执行onTouchEvent()
  //   1. (mViewFlags & ENABLED_MASK) == ENABLED
  //   2. mOnTouchListener != null
  //   3. mOnTouchListener.onTouch(this, event)
  // 下面对这3个条件逐个分析

/**
  * 条件1：(mViewFlags & ENABLED_MASK) == ENABLED
  * 说明：
  *    1. 该条件是判断当前点击的控件是否enable
  *    2. 由于很多View默认enable，故该条件恒定为true（除非手动设置为false）
  */

/**
  * 条件2：mOnTouchListener != null
  * 说明：
  *   1. mOnTouchListener变量在View.setOnTouchListener()里赋值
  *   2. 即只要给控件注册了Touch事件，mOnTouchListener就一定被赋值（即不为空）
  */
  public void setOnTouchListener(OnTouchListener l) { 

    mOnTouchListener = l;  

} 

/**
  * 条件3：mOnTouchListener.onTouch(this, event)
  * 说明：
  *   1. 即回调控件注册Touch事件时的onTouch()；
  *   2. 需手动复写设置，具体如下（以按钮Button为例）
  */
  button.setOnTouchListener(new OnTouchListener() {  
      @Override  
      public boolean onTouch(View v, MotionEvent event) {  
   
        return false;  
        // 若在onTouch()返回true，就会让上述三个条件全部成立，从而使得View.dispatchTouchEvent（）直接返回true，事件分发结束
        // 若在onTouch()返回false，就会使得上述三个条件不全部成立，从而使得View.dispatchTouchEvent（）中跳出If，执行onTouchEvent(event)
        // onTouchEvent()源码分析 -> 分析1
      }  
  });

/**
  * 分析1：onTouchEvent()
  */
  public boolean onTouchEvent(MotionEvent event) {  

    ... // 仅展示关键代码

    // 若该控件可点击，则进入switch判断中
    if (((viewFlags & CLICKABLE) == CLICKABLE || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {  

        // 根据当前事件类型进行判断处理
        switch (event.getAction()) { 

            // a. 事件类型=抬起View（主要分析）
            case MotionEvent.ACTION_UP:  
                    performClick(); 
                    // ->>分析2
                    break;  

            // b. 事件类型=按下View
            case MotionEvent.ACTION_DOWN:  
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());  
                break;  

            // c. 事件类型=结束事件
            case MotionEvent.ACTION_CANCEL:  
                refreshDrawableState();  
                removeTapCallback();  
                break;

            // d. 事件类型=滑动View
            case MotionEvent.ACTION_MOVE:  
                final int x = (int) event.getX();  
                final int y = (int) event.getY();  

                int slop = mTouchSlop;  
                if ((x < 0 - slop) || (x >= getWidth() + slop) ||  
                        (y < 0 - slop) || (y >= getHeight() + slop)) {  
                    removeTapCallback();  
                    if ((mPrivateFlags & PRESSED) != 0) {  
                        removeLongPressCallback();  
                        mPrivateFlags &= ~PRESSED;  
                        refreshDrawableState();  
                    }  
                }  
                break;  
        }  

        // 若该控件可点击，就一定返回true
        return true;  
    }  
  // 若该控件不可点击，就一定返回false
  return false;  
}

/**
  * 分析2：performClick（）
  */  
  public boolean performClick() {  

      if (mOnClickListener != null) {
          // 只要通过setOnClickListener()为控件View注册1个点击事件
          // 那么就会给mOnClickListener变量赋值（即不为空）
          // 则会往下回调onClick() & performClick()返回true
          playSoundEffect(SoundEffectConstants.CLICK);  
          mOnClickListener.onClick(this);  
          return true;  
      }  
      return false;  
  }  
```

#### 源码总结

![](<../.gitbook/assets/image (23).png>)

这里需要特别注意的是，`onTouch（）`的执行 先于 `onClick（）`

#### 核心方法总结

主要包括：dispatchTouchEvent()、onTouchEvent()

![](<../.gitbook/assets/image (79).png>)

#### 实例分析

在本示例中，将分析两种情况：

1. 注册Touch事件监听 且 在onTouch()返回false
2. 注册Touch事件监听 且 在onTouch()返回true

**分析1：注册Touch事件监听 且 在onTouch()返回false**

**代码示例**

```csharp
// 1. 注册Touch事件监听setOnTouchListener 且 在onTouch()返回false
button.setOnTouchListener(new View.OnTouchListener() {

      @Override
      public boolean onTouch(View v, MotionEvent event) {
          System.out.println("执行了onTouch(), 动作是:" + event.getAction());
          return false;
      }
});

// 2. 注册点击事件OnClickListener()
button.setOnClickListener(new View.OnClickListener() {

      @Override
      public void onClick(View v) {
          System.out.println("执行了onClick()");
      }
});

```

**测试结果**

```css
执行了onTouch(), 动作是:0
执行了onTouch(), 动作是:1
执行了onClick()
```

**测试结果说明**

* 点击按钮会产生两个类型的事件-按下View与抬起View，所以会回调两次onTouch()；
* 因为onTouch()返回了false，所以事件无被消费，会继续往下传递，即调用View.onTouchEvent()；
* 调用View.onTouchEvent()时，对于抬起View事件，在调用performClick()时，因为设置了点击事件，所以会回调onClick()。

**分析2：注册Touch事件监听 且 在onTouch()返回true**

**代码示例**

```csharp
// 1. 注册Touch事件监听setOnTouchListener 且 在onTouch()返回false
button.setOnTouchListener(new View.OnTouchListener() {

        @Override
        public boolean onTouch(View v, MotionEvent event) {
            System.out.println("执行了onTouch(), 动作是:" + event.getAction());
            return true;
        }
    });

// 2. 注册点击事件OnClickListener()
button.setOnClickListener(new View.OnClickListener() {

        @Override
        public void onClick(View v) {
            System.out.println("执行了onClick()");
        }
        
  });

```

**测试结果**

```css
执行了onTouch(), 动作是:0
执行了onTouch(), 动作是:1
```

**测试结果说明**

* 点击按钮会产生两个类型的事件-按下View与抬起View，所以会回调两次onTouch()；
* 因为onTouch()返回了true，所以事件被消费，不会继续往下传递，View.dispatchTouchEvent()直接返回true；
* 所以最终不会调用View.onTouchEvent()，也不会调用onClick()。

**至此，关于Android事件分发机制的内容已经讲解完毕，即Activity、ViewGroup、View的事件分发机制。**。

> 即：`Activity`、`ViewGroup`、`View` 的事件分发机制

## 4. 总结

在本章节中，将采用大量的图表从各个角度对Android事件分发机制进行总结。主要包括：

* 工作流程总结
* 业务流程总结
* 以分发对象为核心的总结
* 以方法为核心的总结

### 4.1 工作流程-总结

Android事件分发流程 = Activity -> ViewGroup -> View，即：1个点击事件发生后，事件先传到Activity、再传到ViewGroup、最终再传到

### 4.2 以分发对象为核心-总结

分发对象主要包括：Activity、ViewGroup、View。

### 4.3 以方法为核心-总结

事件分发的方法主要包括：dispatchTouchEvent()、onInterceptTouchEvent()和onTouchEvent()。

![](broken-reference)

这里需要特别注意的是：

* 注意点1：左侧虚线代表具备相关性及逐层返回；
* 注意点2：各层dispatchTouchEvent() 返回true的情况保持一致（图中虚线）\
  原因是：上层dispatchTouchEvent() 的返回true情况 取决于 下层dispatchTouchEvent() 是否返回ture，如Activity.dispatchTouchEvent() 返回true的情况 = ViewGroup.dispatchTouchEvent() 返回true
* 注意点3：各层dispatchTouchEvent() 与 onTouchEvent()的返回情况保持一致\
  原因：最下层View的dispatchTouchEvent()的返回值 取决于 View.onTouchEvent()的返回值；结合注意点1，逐层往上返回，从而保持一致。

下面，将针对该3个方法，分别针对默认执行逻辑、返回true、返回false的三种情况进行流程图示意。

#### 方法1：dispatchTouchEvent()

默认执行逻辑、返回true、返回false 这三种情况的返回逻辑分别如下所示。

![](https://upload-images.jianshu.io/upload\_images/944365-4e61ea3891b889cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)![](https://upload-images.jianshu.io/upload\_images/944365-22faf6c84f4bfea3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)![](https://upload-images.jianshu.io/upload\_images/944365-a3c89322222557dd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

#### 方法2：onInterceptTouchEvent()

默认执行逻辑、返回true、返回false 这三种情况的返回逻辑分别如下所示。

![](https://upload-images.jianshu.io/upload\_images/944365-39faef74cbc29f1d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

这里需要特别注意的是：`Activity`、`View`都无该方法，仅`ViewGroup`特有。

#### 方法3：onTouchEvent()

默认执行逻辑、返回true、返回false 这三种情况的返回逻辑分别如下所示。

\
![](https://upload-images.jianshu.io/upload\_images/944365-8aeb13d9d87df0a2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)![](https://upload-images.jianshu.io/upload\_images/944365-38aa6dd6fdf88ecd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

#### 三者关系

下面，我用一段伪代码来阐述上述3个方法的关系 & 事件传递规则

```java
// 点击事件产生后
// 步骤1：调用dispatchTouchEvent()
public boolean dispatchTouchEvent(MotionEvent ev) {

    boolean consume = false; //代表 是否会消费事件

    // 步骤2：判断是否拦截事件
    if (onInterceptTouchEvent(ev)) {
      // a. 若拦截，则将该事件交给当前View进行处理
      // 即调用onTouchEvent()去处理点击事件
      consume = onTouchEvent (ev) ;

    } else {

      // b. 若不拦截，则将该事件传递到下层
      // 即 下层元素的dispatchTouchEvent()就会被调用，重复上述过程
      // 直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    // 步骤3：最终返回通知 该事件是否被消费（接收 & 处理）
    return consume;

}
```

***

## 5. 常见事件分发场景

下面，我将通过实例说明**常见的事件传递情况 & 流程**

#### 5.1 背景描述

* 讨论的布局如下：

![](https://upload-images.jianshu.io/upload\_images/944365-e0f526dd1b5731be.png?imageMogr2/auto-orient/strip|imageView2/2/w/438/format/webp)示意图

*   情景

    1. 用户先触摸到屏幕上`View C`上的某个点（图中黄区）

    > `Action_DOWN`事件在此处产生

    1. 用户移动手指
    2. 最后离开屏幕

#### 5.2 一般的事件传递情况

一般的事件传递场景有：

* 默认情况
* 处理事件
* 拦截`DOWN`事件
* 拦截后续事件（`MOVE`、`UP`）

#### 场景1：默认

* 即不对控件里的方法（`dispatchTouchEvent()`、`onTouchEvent()`、`onInterceptTouchEvent()`）进行重写 或 更改返回值
* 那么调用的是这3个方法的默认实现：调用下层的方法 & 逐层返回
*   事件传递情况：（呈`U`型）

    1. 从上往下调用dispatchTouchEvent()

    > Activity A ->> ViewGroup B ->> View C

    1. 从下往上调用onTouchEvent()

    > View C ->> ViewGroup B ->> Activity A

![](https://upload-images.jianshu.io/upload\_images/944365-161a6e6fc8723248.png?imageMogr2/auto-orient/strip|imageView2/2/w/1140/format/webp)示意图

> 注：虽然`ViewGroup B`的`onInterceptTouchEvent`（）对`DOWN`事件返回了`false`，但后续的事件`（MOVE、UP）`依然会传递给它的`onInterceptTouchEvent()`\
> 这一点与`onTouchEvent（）`的行为是不一样的：不再传递 & 接收该事件列的其他事件

#### 场景2：处理事件

设`View C`希望处理该点击事件，即：设置`View C`为可点击的`（Clickable）` 或 复写其`onTouchEvent（）`返回`true`

> 最常见的：设置`Button`按钮来响应点击事件

事件传递情况：（如下图）

* `DOWN`事件被传递给C的`onTouchEvent`方法，该方法返回`true`，表示处理该事件
* 因为`View C`正在处理该事件，那么`DOWN`事件将不再往上传递给ViewGroup B 和 `Activity A`的`onTouchEvent()`；
* 该事件列的其他事件`（Move、Up）`也将传递给`View C`的`onTouchEvent()`

![](https://upload-images.jianshu.io/upload\_images/944365-77e933eb44682777.png?imageMogr2/auto-orient/strip|imageView2/2/w/1140/format/webp)示意图

> 会逐层往`dispatchTouchEvent()` 返回，最终事件分发结束

#### 场景3：拦截DOWN事件

假设`ViewGroup B`希望处理该点击事件，即`ViewGroup B`复写了`onInterceptTouchEvent()`返回`true`、`onTouchEvent()`返回`true`\
事件传递情况：（如下图）

* `DOWN`事件被传递给`ViewGroup B`的`onInterceptTouchEvent()`，该方法返回`true`，表示拦截该事件，即自己处理该事件（事件不再往下传递）
* 调用自身的`onTouchEvent()`处理事件（`DOWN`事件将不再往上传递给`Activity A`的`onTouchEvent()`）
* 该事件列的其他事件`（Move、Up）`将直接传递给`ViewGroup B`的`onTouchEvent()`

> 注：
>
> 1. 该事件列的其他事件`（Move、Up）`将不会再传递给`ViewGroup B`的`onInterceptTouchEvent`（）；因：该方法一旦返回一次`true`，就再也不会被调用
> 2. 逐层往`dispatchTouchEvent()` 返回，最终事件分发结束

![](https://upload-images.jianshu.io/upload\_images/944365-a5e7cfed2cba02c3.png?imageMogr2/auto-orient/strip|imageView2/2/w/1140/format/webp)示意图

#### 场景4：拦截DOWN的后续事件

**结论**

* 若 `ViewGroup` 拦截了一个半路的事件（如`MOVE`），该事件将会被系统变成一个`CANCEL`事件 & 传递给之前处理该事件的子`View`；
* 该事件不会再传递给`ViewGroup` 的`onTouchEvent()`
* 只有再到来的事件才会传递到`ViewGroup`的`onTouchEvent()`

**场景描述**\
`ViewGroup B` 无拦截`DOWN`事件（还是`View C`来处理`DOWN`事件），但它拦截了接下来的`MOVE`事件

> 即 `DOWN`事件传递到`View C`的`onTouchEvent（）`，返回了`true`

**实例讲解**

* 在后续到来的MOVE事件，`ViewGroup B` 的`onInterceptTouchEvent（）`返回`true`拦截该`MOVE`事件，但该事件并没有传递给`ViewGroup B` ；这个`MOVE`事件将会被系统变成一个`CANCEL`事件传递给`View C`的`onTouchEvent（）`
* 后续又来了一个`MOVE`事件，该`MOVE`事件才会直接传递给`ViewGroup B` 的`onTouchEvent()`

> 后续事件将直接传递给`ViewGroup B` 的`onTouchEvent()`处理，而不会再传递给`ViewGroup B` 的`onInterceptTouchEvent（）`，因该方法一旦返回一次true，就再也不会被调用了。

* `View C`再也不会收到该事件列产生的后续事件

![](https://upload-images.jianshu.io/upload\_images/944365-1599f532038686cd.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)示意图

至此，关于`Android`常见的事件传递情况 & 流程已经讲解完毕。

***

## 6. 特殊说明

#### 6.1 Touch事件的后续事件（MOVE、UP）层级传递

* 若给控件注册了`Touch`事件，每次点击都会触发一系列`action`事件（ACTION\_DOWN，ACTION\_MOVE，ACTION\_UP等）
* 当`dispatchTouchEvent（）`事件分发时，只有前一个事件（如ACTION\_DOWN）返回true，才会收到后一个事件（ACTION\_MOVE和ACTION\_UP）

> 即如果在执行ACTION\_DOWN时返回false，后面一系列的ACTION\_MOVE、ACTION\_UP事件都不会执行

从上面对事件分发机制分析知：

* dispatchTouchEvent()、 onTouchEvent() 消费事件、终结事件传递（返回true）
* 而onInterceptTouchEvent 并不能消费事件，它相当于是一个分叉口起到分流导流的作用，对后续的ACTION\_MOVE和ACTION\_UP事件接收起到非常大的作用

> 请记住：接收了ACTION\_DOWN事件的函数不一定能收到后续事件（ACTION\_MOVE、ACTION\_UP）

**这里给出ACTION\_MOVE和ACTION\_UP事件的传递结论**：

* 结论1\
  若对象（Activity、ViewGroup、View）的dispatchTouchEvent()分发事件后消费了事件（返回true），那么收到ACTION\_DOWN的函数也能收到ACTION\_MOVE和ACTION\_UP

> 黑线：ACTION\_DOWN事件传递方向\
> 红线：ACTION\_MOVE 、 ACTION\_UP事件传递方向

![](https://upload-images.jianshu.io/upload\_images/944365-93d0b1496e9e6ca4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)流程讲解

* 结论2\
  若对象（Activity、ViewGroup、View）的onTouchEvent()处理了事件（返回true），那么ACTION\_MOVE、ACTION\_UP的事件从上往下传到该`View`后就不再往下传递，而是直接传给自己的`onTouchEvent()`& 结束本次事件传递过程。

> 黑线：ACTION\_DOWN事件传递方向\
> 红线：ACTION\_MOVE、ACTION\_UP事件传递方向

![](https://upload-images.jianshu.io/upload\_images/944365-9d639a0b9ebf7b4a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)流程讲解

#### 6.2 onTouch()和onTouchEvent()的区别

* 该2个方法都是在`View.dispatchTouchEvent（）`中调用
* 但`onTouch（）`优先于`onTouchEvent`执行；若手动复写在`onTouch（）`中返回`true`（即 将事件消费掉），将不会再执行`onTouchEvent（）`

> 注：若1个控件不可点击（即非`enable`），那么给它注册`onTouch`事件将永远得不到执行，具体原因看如下代码

```csharp
// &&为短路与，即如果前面条件为false，将不再往下执行
//  故：onTouch（）能够得到执行需2个前提条件：
     // 1. mOnTouchListener的值不能为空
     // 2. 当前点击的控件必须是enable的
mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED &&  
            mOnTouchListener.onTouch(this, event)

// 对于该类控件，若需监听它的touch事件，就必须通过在该控件中重写onTouchEvent（）来实现
```



*

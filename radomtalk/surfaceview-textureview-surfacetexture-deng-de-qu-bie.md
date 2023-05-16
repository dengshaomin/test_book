# SurfaceView, TextureView, SurfaceTexture等的区别

## 1、SurfaceView

### **1.1、概述**

SurfaceView继承自类View，因此它本质上是一个View。但与普通View不同的是，它有自己的Surface，在WMS中有对应的WindowState，在SurfaceFlinger中有Layer。

调用者可以通过`lockCanvas`获得了一块类型为Canvas的画布之后，就可以调用Canvas类所提供的绘图函数来绘制任意的UI了，例如，调用Canvas类的成员函数`drawLine`、`drawRect`和`drawCircle`可以分别用来画直线、矩形和圆。

调用者在画布上绘制完成所需要的UI之后，通过调用SurfaceHolder类的成员函数`unlockCanvasAndPost`就可以将这块画布的图形绘冲区的UI数据提交给SurfaceFlinger服务来处理了，以便SurfaceFlinger服务可以在合适的时候将该图形缓冲区合成到屏幕上去显示，这样就可以将对应的SurfaceView的UI展现出来了。

### **1.2、双缓冲机制**

SurfaceView在更新视图时用到了两张Canvas，一张frontCanvas和一张backCanvas，每次实际显示的是frontCanvas，backCanvas存储的是上一次更改前的视图，当使用lockCanvas（）获取画布时，得到的实际上是backCanvas而不是正在显示的frontCanvas，之后你在获取到的backCanvas上绘制新视图，再unlockCanvasAndPost（canvas）此视图，那么上传的这张canvas将替换原来的frontCanvas作为新的frontCanvas，原来的frontCanvas将切换到后台作为backCanvas。例如，如果你已经先后两次绘制了视图A和B，那么你再调用lockCanvas（）获取视图，获得的将是A而不是正在显示的B，之后你将重绘的C视图上传，那么C将取代B作为新的frontCanvas显示在SurfaceView上，原来的B则转换为backCanvas。

### **1.3、SurfaceView优点与缺点**

* 优点： 使用双缓冲机制，可以在一个独立的线程中进行绘制，不会影响主线程，播放视频时画面更流畅
* 缺点：Surface不在View hierachy中，它的显示也不受View的属性控制，SurfaceView 不能嵌套使用。在7.0版本之前不能进行平移，缩放等变换，也不能放在其它ViewGroup中，在7.0版本之后可以进行平移，缩放等变换。

## 2、TextureView

### **2.1、概述**

在4.0(API level 14)中引入，与SurfaceView一样继承View，它可以将内容流直接投影到View中，TextureView重载了draw()方法，其中主要SurfaceTexture中收到的图像数据作为纹理更新到对应的HardwareLayer中。

和SurfaceView不同，它不会在WMS中单独创建窗口，而是作为View hierachy中的一个普通View，因此可以和其它普通View一样进行移动，旋转，缩放，动画等变化。值得注意的是TextureView必须在硬件加速的窗口中。它显示的内容流数据可以来自App进程或是远端进程。

### **2.2、TextureView优点与缺点**

* 优点：支持移动、旋转、缩放等动画，支持截图
* 缺点：必须在硬件加速的窗口中使用，占用内存比SurfaceView高，在5.0以前在主线程渲染，5.0以后有单独的渲染线程。

### **2.3、TextureView与SurfaceView对比**

| \\    | SurfaceView | TextureView |
| ----- | ----------- | ----------- |
| 内存    | 低           | 高           |
| 绘制    | 及时          | 1\~3帧的延迟    |
| 耗电    | 低           | 高           |
| 动画与截图 | 不支持         | 支持          |

* TextureView总是使用GL合成，而SurfaceView可以使用硬件overlay后端，可以占用更少的内存带宽，消耗更少的CPU（耗电）；
* TextureView的内部缓冲队列导致比SurfaceView使用更多的内存；

## 3、SurfaceTexture

### **3.1、概述**

SurfaceTexture 类是在 Android 3.0 中引入的。当你创建了一个 SurfaceTexture，你就创建了你的应用作为消费者的 BufferQueue。当一个新的缓冲区由生产者入队列时，你的应用将通过回调 (`onFrameAvailable()`) 被通知。你的应用调用 `updateTexImage()`，这将释放之前持有的缓冲区，并从队列中获取新的缓冲区，执行一些 EGL 调用以使缓冲区可作为一个外部 texture 由 GLES 使用。

### **3.2、SurfaceTexture与SurfaceView对比**

SurfaceTexture和SurfaceView不同的是，它对图像流的处理并不直接显示，而是转为OpenGL外部纹理，因此可用于图像流数据的二次处理（如Camera滤镜，桌面特效等）。比如Camera的预览数据，变成纹理后可以交给GLSurfaceView直接显示，也可以通过SurfaceTexture交给TextureView作为View heirachy中的一个硬件加速层来显示。

## 4、GLSurfaceView

GLSurfaceView从Android 1.5(API level 3)开始加入。在SurfaceView的基础上，和SurfaceView不同的是，它加入了EGL的管理，并自带了渲染线程。另外它定义了用户需要实现的Render接口，只需要将实现了渲染函数的Renderer的实现类设置给GLSurfaceView即可。

GLSurfaceView也可以作为相机的预览，但是需要创建自己的SurfaceTexture并调用OpenGl API绘制出来。GLSurfaceView 本身自带EGL的管理，并有渲染线程，这对于一些需要多个EGLSurface的场景将不适用。

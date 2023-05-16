# 使用动态代理Hook

Hook View的Click事件

```kotlin

  private fun hookView(v: View) {
    //获取view class
    val viewClass = Class.forName("android.view.View")
    //获取view 方法
    val method = viewClass.getDeclaredMethod("getListenerInfo")
    //私有设置成可读
    method.isAccessible = true
    //获取view真正的对象
    val realMethod = method.invoke(v)
    
    //获取listenerinfo class
    val listenerInfoClass = Class.forName("android.view.View$" + "ListenerInfo")
    //获取class中的mOnClickListener变量
    val onclickField = listenerInfoClass.getField("mOnClickListener")
    //获取view中真正的click对象
    val clickObject = onclickField.get(realMethod)
    //初始化代理
    val clickProxy = Proxy.newProxyInstance(
        this::class.java.classLoader, arrayOf<Class<*>>(OnClickListener::class.java),
        object : InvocationHandler {
          override fun invoke(
            proxy: Any,
            method: Method,
            args: Array<out Any>
          ): Any? {
//            return cat.callOnClick()  //call other view method
            return method.invoke(clickObject,v) //continue invoke view onclick
          }
        })
    //给变量设置动态代理
    onclickField.set(realMethod,clickProxy)
  }
```

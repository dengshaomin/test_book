---
description: https://juejin.im/post/6844903930468499469#heading-5
---

# 安卓推送集成（极光+华为+小米+OPPO+VIVO）

**目前使用：极光+华为+小米+OPPO+VIVO**\
&#x20;阶段性总结 基本上推送都按照文档来走，都不复杂，比较简单，在一开始设计好模式之后，基本上一家推送半天就可以集成成功，加上测试最多一天。\
&#x20;基本模式\
&#x20;`初始化` 需要初始化的放在app中，多进程在主进程中进行初始化\
&#x20;`判断注册`判断手机型号，只进行一种推送的注册\
&#x20;`注册` 就是设置alias，或者注册regId，提交给我方服务端\
&#x20;`解除注册`推送出登陆的时候使用，就是移除alias，或者移除regId，或者解除注册 `开启推送`调用我方服务端接口，设置个人推送标志开启\
&#x20;`关闭推送`调用我方服务端接口，设置个人推送标志关闭\
&#x20;`重试机制`在某一项设置失败之后，进行一点时间后的重试 比如10秒后重试\
&#x20;`数据传值`一般来说都是可以在打开页面进行获取到数据，不行的话也有广播接受者进行注册，调用自定义的接口进行数据回调

### 一、设置点击跳转到应用页面，并且传递数据

_**注意：这里是在点击的时候获取到的数据，如果需要在接收的的时候就的得到数据还是要在接收回到中获取**_ 建议所有的推送通知栏点击意图都设置成打开应用内界面，这样可以把所有的逻辑业务都放在一个Activity中进行处理，数据使用意图uri进行传递，由于不同的推送获取到的intent意图是不一样的，所有的数据都在PushActionActivity中获取

```
 * 华为：intent://{host}/{path}#Intent;scheme={scheme};launchFlags=0x10000000;end
 * 极光：{scheme}://{host}/{path}
 * 小米：intent:#Intent;component={包名}/{意图页面完整路径};end
 
type = 1 url = www.baidu.com 

华为: 
传参：需要把参数放到自定义动作里面 
//生成自定义动作
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("pioneer://push/notify"))
intent.putExtra("type", 1)
intent.putExtra("url", "www.baidu.com")
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
val uri = intent.toUri(Intent.URI_INTENT_SCHEME)
LogUtils.iTag("Push", "华为推送自定义动作 uri = $uri")

注意!!!
< 类型.key=value;类型.key=value > 
S.url=www.baidu.com;i.type=1

intent://push/notify#Intent;scheme=pioneer;launchFlags=0x10000000;S.url=www.baidu.com;i.type=1;end

获取数据 
val hwExtra = intent.extras
hwExtra?.keySet()?.forEach {
        builder.append("\nkey = $it").append("\t\tvalue = ${hwExtra[it]}")
}

--------------------------------------------------------------------

小米：
传参：把参数放到自定义键值对里面 不放在intent里面

intent:#Intent;component={包名}/{意图页面全路径};end

获取数据 
val miPushMessage = intent.getSerializableExtra(PushMessageHelper.KEY_MESSAGE) as MiPushMessage
val miExtra = miPushMessage.extra
miExtra.keys.forEach {
        builder.append("\nkey = $it").append("\t\tvalue = ${miExtra[it]}")
}

--------------------------------------------------------------------
极光：
传参：
不知道为什么在极光后台添加是可以实现的,但是在我们的服务端添加就不可以了,所以放弃这种方案 
！！！通过接收极光的通知栏点击意图来进行listener的回调 数据也从广播中获取

不要使用
不要使用
不要使用
1、可以直接使用api进行添加附加数据，这样需要在自定义的广播中获取额外数据
2、直接把数据放在点击意图中 从intent.data中获取数据
pioneer://push/notify?type=1&url=www.baidu.com

获取数据
val jpExtra = intent.data
jpExtra?.queryParameterNames?.forEach {
        builder.append("\nkey = $it").append("\t\tvalue = ${jpExtra.getQueryParameter(it)}")
}
复制代码
```

### 二、极光推送

[极光推送 客户端集成文档](https://docs.jiguang.cn/jpush/client/Android/android\_guide/) 很简单，基本没什么复杂的东西，半天搞定 如果手机中其他应用也集成了极光，那么他们之间会相互调启动连接极光服务器

1. **配置SDK**

* 添加jcenter集成

```
// 根节点的build.gradle
buildscript {
    repositories {
        jcenter()
    }
    ......
}

allprojects {
    repositories {
        jcenter()
    }
}

//推送模块的build.gradle
android {
    defaultConfig {
        applicationId "com.xxx.xxx" //JPush 上注册的包名.
        ndk {
            abiFilters 'armeabi', 'armeabi-v7a', 'arm64-v8a'
            // 还可以添加 'x86', 'x86_64', 'mips', 'mips64'
        }
        manifestPlaceholders = [
            JPUSH_PKGNAME : applicationId,
            JPUSH_APPKEY : "你的 Appkey",
            JPUSH_CHANNEL : "developer-default"]
    }
}

dependencies {
    implementation 'cn.jiguang.sdk:jpush:3.3.4'  
    implementation 'cn.jiguang.sdk:jcore:2.1.2' 
}
复制代码
```

1. **创建自定义3个文件** 具体的回调方法查看文档，都有详细介绍，更具需求进行逻辑处理

```
class JpushService : JCommonService()

// 接收极光设置alias和tags的反馈回调
class JpushMessageReceiver : JPushMessageReceiver(){
    //在设置alias失败后 code!=0 延时重试
    //在设置tags失败后 code!=0 延时重试
}

// 接收极光的推送的消息回调
class JpushReceiver : BroadcastReceiver(){
    //如果使用了点击通知栏意图 那么这里就打印一下日志就好
    //其他按照需求来进行逻辑操作
}
复制代码
```

1. **配置清单文件**

```
		<service
			android:name=".services.PushService"
			android:enabled="true"
			android:exported="false"
			android:process=":pushcore">
			<intent-filter>
				<action android:name="cn.jiguang.user.service.action" />
			</intent-filter>
		</service>
		
		<receiver
			android:name=".receiver.JPushMessageReceiver"
			android:enabled="true"
			android:exported="false">
			<intent-filter>
				<action android:name="cn.jpush.android.intent.RECEIVE_MESSAGE" />
				<category android:name="${JPUSH_PKGNAME}" />
			</intent-filter>
		</receiver>
		
		<receiver
			android:name=".receiver.JPushNotifyReceiver"
			android:enabled="true"
			android:exported="false">
			<intent-filter>
				<!--Required  用户注册SDK的intent-->
				<action android:name="cn.jpush.android.intent.REGISTRATION" />
				<!--Required  用户接收SDK消息的intent-->
				<action android:name="cn.jpush.android.intent.MESSAGE_RECEIVED" />
				<!--Required  用户接收SDK通知栏信息的intent-->
				<action android:name="cn.jpush.android.intent.NOTIFICATION_RECEIVED" />
				<!--Required  用户打开自定义通知栏的intent-->
				<action android:name="cn.jpush.android.intent.NOTIFICATION_OPENED" />
				<!-- 接收网络变化 连接/断开  -->
				<action android:name="cn.jpush.android.intent.CONNECTION" />
				<category android:name="${JPUSH_PKGNAME}" />
			</intent-filter>
		</receiver>
	
复制代码
```

1. **开始使用** 多进程 请在主进程中进行初始化

```
//JPush
JPushInterface.setDebugMode(BuildConfig.DEBUG)
JPushInterface.init(this)
复制代码
```

在合适的地方进行设置alias和tags 注意： setAlias是重置alias setTags是重置tags addTags是添加一个tags

```
// sequence 只是作为一次设置的标记 每一次设置可以使用不同的数字来进行标记 1，2，3
// 在JpushMessageReceiver中会返回这次设置的sequence作为唯一性判断
setAlias(Context context, int sequence, String alias)

// 重置tags 
setTags(Context context, int sequence, Set<String> tags)

// 添加tags
addTags(Context context, int sequence, Set<String> tags)
复制代码
```

### 三、华为推送

[华为推送开发文档 - 开发准备](https://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush\_agent.html?page=hmssdk\_huaweipush\_devprepare\_agent) 这个不难，我看了很多博客，他们都说华为推送很坑，不过我是一个上午就集成好了，测试通过，没问题的

1. **第一步**

* **注册认证成为华为开发者**
* **配置应用签名 SHA256** 把签名文件放在app下，然后在Android Studio底部Terminal中输入 `keytool -list -v -keystore ***.keystore` 根据提示输入密码，就可以得到SHA256了
* **创建应用并且开通推送服务**
* **集成SDK** 1、AndroidStudio使用Maven集成华为推送SDK 2、下载HMS SDK Agent套件，解压运行GetHMSAgent\_cn.bat(Mac运行GetHMSAgent\_cn.sh 我测试不能成功，可能我菜，最后让同事windows帮忙运行的) 运行后，根据提示进行集成代码，最后生成的代码在同文件夹下的copysrc中，将com文件夹直接拖到Push模块的java文件夹下合并，这样AS的com文件夹下有两个文件夹了，其中一个是huawei，不用改代码包名
* **配置清单文件** 更具文档来进行配置，对于步骤8的onEvent回调建议不要使用，会在后续删除，使用点击意图跳转本地页面进行设置
* **混淆配置，AndResGuard混淆配置**

1. **第二步**

* 初始化

```
        HMSAgent.init(app)
复制代码
```

* 连接服务器，注册token 1、注意连接服务器需要在打开应用的第一个页面就开始连接 2、后台推送是根据用户token来的，需要提交到自己的后台，所以视情况而定，token的回调通过广播接收,接收后提交给自己的服务器后台

```
//连接华为推送服务器
HMSAgent.connect(activity) { code ->
        LogUtils.iTag("Push", "华为推送", "连接华为服务器状态 $code")
}

//获取华为用户Token
HMSAgent.Push.getToken { code ->
        LogUtils.iTag("Push", "华为推送", "获取Token状态 $code")
}

//注册广播接收
class HuaWeiPushReceiver : PushReceiver() {

    override fun onToken(context: Context?, token: String?) {
        super.onToken(context, token)
        LogUtils.iTag("Push", "华为推送", "Token注册成功")
        if (token != null) {
            PushManager.get().getPushListener()?.onHuaWeiToken(token)
        }
    }

    override fun onPushState(context: Context?, pushState: Boolean) {
        super.onPushState(context, pushState)
        LogUtils.iTag("Push", "华为推送", "连接服务器状态 $pushState")
    }

}
复制代码
```

### 四、小米推送

[小米推送文档](https://dev.mi.com/console/doc/detail?pId=41) 很简单，基本没什么复杂的东西，半天搞定

1. **第一步**

* **成为小米开发者，创建推送应用，获取appid appkey**
* **下载sdk `*.jar` 放入lib**
* **创建广播接收者继承PushMessageReceiver** 由于推送的意图是打开应用内页面（1像素的透明页面），内部逻辑都在里面，所以只需要回调`onCommandResult`对于内部的处理查看文档，建议在注册失败，设置alias失败，10秒后进行重试，删除alias失败不进行重试，具体还是要看业务逻辑
* **配置清单文件**

1. **第二步**

* 初始化

```
        //由于应用有多个进程 只在主进程中进行调用
        //内部日志打印
        Logger.setLogger(appContext, logger)
        
        //注册推送
        MiPushClient.registerPush(appContext,
                BuildConfig.XiaoMiPushAppId,
                BuildConfig.XiaoMiPushAppKey)
复制代码
```

* 注册成功之后再 设置alias

```
        //在合适的地方设置Alias
        MiPushClient.setAlias(appContext, alias, null)
        
        //在合适的地方取消alias
        MiPushClient.unsetAlias(appContext, alias, null)
复制代码
```

### 五、OPPO推送

[OPPO PUSH SDK接口文档](https://open.oppomobile.com/wiki/doc#id=10196) 直接根据文档走，几乎没有难度\
&#x20;注意点：

1. 只支持OPPO系统级别的推送，打开指定页面进行处理，和其他相同
2. 使用前注意判断`PushManager.isSupportPush(context)`是否支持推送
3. 服务端传值，在打开界面进行数据处理
4. 同上，是否发送推送都由服务端决定，调用自家接口设置推送开关
5. 注册或者alias操作失败后，进行重试

* 进行注册 解除注册

```
/**
     * 注册OPush推送服务
     * @param applicatoinContext必须传入当前app的applicationcontet
     * @param appKey 在开发者网站上注册时生成的，与appID相对应，用于验证
* appID是否合法
     * @param pushCallback SDK操作的回调，如不需要使用所有回调接口，可传    	 * 入PushAdapter，选择需要的回调接口来重写
     */
    void register(Context applicatoinContext, String appKey, String a          ppSecret, PushCallback pushCallback);
    
        /**
     * 解注册OPush推送服务
     */
    void unRegister();
复制代码
```

* 设置alias

```
    /**
     * 为指定用户设置aliases
     *
     * @param aliases 为指定用户设置别名
     */
    @Deprecated
    void setAliases(List<String> aliases);
复制代码
```

### 六、Vivo推送

[VIVO PUSH SDK接口文档](https://dev.vivo.com.cn/documentCenter/doc/233) 直接更具文档走，几乎没有难度 可以通过api `checkManifest()` 来检查AndroidManifest是否正确声明组件和权限\
&#x20;注意点：！！！推送服务SDK支持的最低安卓版本为Android 6.0系统！！！

1. 判断是否是Vivo手机(RoomUtils.isVivo()) 判断是否支持Vivo推送（api方法 isSupport()）都成立使用Vivo 不成立使用极光
2. PushSdk2.3.4 开始支持将【自定义键值对】传递到打开的应用页面中，直接在打开页面进行数据处理，也可以在 `OpenClientPushMessageReceiver` 的 `onNotificationMessageClicked` 中获取数据进行接口回调（这个需要自己实现，查看文档）
3. 同上，是否发送推送都由服务端决定，调用自家接口设置推送开关

```
//判断是否支持
PushClient.getInstance(appContext).isSupport()
//判断清单文件是否正确配置
PushClient.getInstance(appContext).checkManifest()
//初始化
PushClient.getInstance(appContext).initialize()
//设置alias
PushClient.getInstance(appContext).bindAlias(alias)
//解除alias
PushClient.getInstance(appContext).unBindAlias(alias)
复制代码
```

### 七、角标设置

角标适配工具 [ShortcutBadger - Gayhub](https://github.com/leolin310148/ShortcutBadger)

1. 对于使用一般可设置 可以直接使用上方的工具类进行设置，角标数量可以自己进行设置或者清空 特殊： OPPO：需要询问客服对于接入OPPO角标需要什么条件 VIVO：需要询问客服对于接入VIVO角标需要什么条件，有的说只有微信、QQ这些和系统应用能用，有的说上架VIVO商城，达到A级标准就可以，具体还是问客服
2. 对于华为 在获取到通知的时候，通过华为推送的后台api进行设置， [华为推送java服务端配置文档](https://developer.huawei.com/consumer/cn/service/hms/catalog/huaweipush\_agent.html?page=hmssdk\_huaweipush\_api\_reference\_agent\_s2) 为了让华为角标更适配，使用华为推送Api进行设置 在页面下方`3.参数`中找到`通知栏消息附加PUSH角标` 配合文档查看\
   &#x20;action - param - intent 就是我们上面生成的intentUri ext - badegAddNum 角标数量 ext - badgeClass 启动页路径 com.push.demo.ui.SplashActivity 之后在合适的地方使用工具类进行设置角标数量或者清空
3. 对于小米 由于小米推送是通过反射来进行设置，但是使用系统通知怎么设置角标，拿不到Notification 同时注册两个小米和极光推送，收到消息的时候关闭小米推送通知和极光推送通知，然后自定义一个本地的Notification，设置角标数量，但是杀死App之后只走小米通道的，按照小米系统角标来的，最后是说服了产品，就按照系统的通知来，来一个系统加1，不用代码进行控制

```
public static void setBadgeNumber(Notification notification, int number) {
        try {
            Field field = notification.getClass().getDeclaredField("extraNotification");
            Object extraNotification = field.get(notification);
            Method method = extraNotification.getClass().getDeclaredMethod("setMessageCount", int.class);
            method.invoke(extraNotification, number);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

\
作者：MeMo44856\
链接：https://juejin.im/post/6844903930468499469\
来源：掘金\
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

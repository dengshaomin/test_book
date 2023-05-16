---
description: https://www.jianshu.com/p/678e2322fd41
---

# TaskStackBuilder

官方文档：[https://developer.android.google.cn/training/notify-user/navigation](https://developer.android.google.cn/training/notify-user/navigation)

场景：当应用处于后台时，默认情况下，从通知启动一个Activity，按返回键会回到主屏幕。但遇到这样的需求，按返回键时仍然留在当前应用。类似于微信、QQ等点击通知栏，显示Chat页，点击返回会回到主Activity。

## 一

在MainActivity点击按钮开启一个服务，并将Activity退出。服务中子线程睡眠3秒后，模拟弹出通知。点击通知栏，进入消息列表页后。点击返回按钮时，可见直接回到了桌面。并没有回到自己主页面的。![](https://upload-images.jianshu.io/upload\_images/93730-62f49dd3036eb0c5.gif?imageMogr2/auto-orient/strip|imageView2/2/w/327/format/webp)默认形式

代码片段：

```
private void showNotification() {
    NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);

    Intent msgIntent = new Intent();
    msgIntent.setClass(this, MessageActivity.class);

    PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, msgIntent, PendingIntent.FLAG_UPDATE_CURRENT);

    // create and send notificaiton
    NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this)
            .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
            .setSmallIcon(getApplicationInfo().icon)
            .setWhen(System.currentTimeMillis())
            .setAutoCancel(true)//自己维护通知的消失
            .setContentTitle("我是标题")
            .setTicker("我是ticker")
            .setContentText("我是内容")
            .setContentIntent(pendingIntent);
    //将一个Notification变成悬挂式Notification
    mBuilder.setFullScreenIntent(pendingIntent, true);
    Notification notification = mBuilder.build();
    manager.notify(0, notification);
}
```

## 二

实现上述需求，采用PendingIntent.getActivities()方法\
&#x20;代码片段：

```
private void showNotification() {
    NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);

    Intent msgIntent = new Intent();
    Intent mainIntent = new Intent();
    msgIntent.setClass(this, MessageActivity.class);
    mainIntent.setClass(this, MainActivity.class);
    //注意此处的顺序
    Intent[] intents = new Intent[]{mainIntent, msgIntent};
    PendingIntent pendingIntent = PendingIntent.
            getActivities(this, 0, intents, PendingIntent.FLAG_UPDATE_CURRENT);

    // create and send notificaiton
    NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this)
            .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
            .setSmallIcon(getApplicationInfo().icon)
            .setWhen(System.currentTimeMillis())
            .setAutoCancel(true)//自己维护通知的消失
            .setContentTitle("我是标题")
            .setTicker("我是ticker")
            .setContentText("我是内容")
            .setContentIntent(pendingIntent);
    //将一个Notification变成悬挂式Notification
    mBuilder.setFullScreenIntent(pendingIntent, true);
    Notification notification = mBuilder.build();
    manager.notify(0, notification);
}
```

![](https://upload-images.jianshu.io/upload\_images/93730-c7ef60ab63799176.gif?imageMogr2/auto-orient/strip|imageView2/2/w/327/format/webp)点击返回，回到主界面

## 三

实现上述需求，采用TaskStackBuilder方法\
&#x20;1.在AndroidManifest.xml配置Activity关系

```
<activity android:name=".MessageActivity"
          android:parentActivityName=".MainActivity"/>
```

2.代码

```
private void showNotification() {
    NotificationManager manager = (NotificationManager) getSystemService(Context.NOTIFICATION_SERVICE);
    //启动通知Activity时，拉起主页面Activity
    Intent msgIntent = new Intent();
    msgIntent.setClass(this, MessageActivity.class);

    //创建返回栈
    TaskStackBuilder stackBuilder = TaskStackBuilder.create(this);
    //添加Activity到返回栈
    stackBuilder.addParentStack(MessageActivity.class);
    //添加Intent到栈顶
    stackBuilder.addNextIntent(msgIntent);

    PendingIntent pendingIntent = stackBuilder.getPendingIntent(0, PendingIntent.FLAG_UPDATE_CURRENT);

    // create and send notificaiton
    NotificationCompat.Builder mBuilder = new NotificationCompat.Builder(this)
            .setLargeIcon(BitmapFactory.decodeResource(getResources(), R.mipmap.ic_launcher))
            .setSmallIcon(getApplicationInfo().icon)
            .setWhen(System.currentTimeMillis())
            .setAutoCancel(true)//自己维护通知的消失
            .setContentTitle("我是标题")
            .setTicker("我是ticker")
            .setContentText("我是内容")
            .setContentIntent(pendingIntent);
    //将一个Notification变成悬挂式Notification
    mBuilder.setFullScreenIntent(pendingIntent, true);
    Notification notification = mBuilder.build();
    manager.notify(0, notification);
}
```

## 小结：

第二种方式适合项目在一个module中开发的情况。如果是组件化开发，通知页面MessageActivity在其他module中，则是无法引用到MainActivity的。因此采用第三种方式更合适。只需要在app的清单文件中再次配置一下Activity的关系即可。打包的时候会合并清单文件配置。

##

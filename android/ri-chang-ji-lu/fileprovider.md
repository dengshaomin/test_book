# FileProvider

在Android 7.0及以上版本上，通过以下方式调用相机会报`android.os.FileUriExposedException`错

```
imageUri = Uri.fromFile(mTempImgFile);
intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
mContext.startActivityForResult(intent, CODE_CHOOSE_FROM_CAMERA);
```

出于安全的考虑，Google推荐采用FileProvider的方式来代替。[https://developer.android.com/reference/android/support/v4/content/FileProvider.html](https://developer.android.com/reference/android/support/v4/content/FileProvider.html)

替换FileProvider只需要三步即可：

#### 1 在res/xml/中指定可用文件[¶](https://blog.yorek.xyz/android/other/FileProvider/#1-resxml) <a href="#1-resxml" id="1-resxml"></a>

```
<!-- res/xml/filepaths.xml -->
<paths>
    <files-path name="my_images" path="images/"/>
    <files-path name="my_docs" path="docs/"/>
</paths>
```

\<paths>节点必须包含有最少一个以下子节点，可以同时包含多个子节点：

| 子节点                          | 代码获取目录                                    | 对应目录                                        |
| ---------------------------- | ----------------------------------------- | ------------------------------------------- |
| \<files-path ... />          | Context.getFilesDir()                     | /data/data/{package}/files                  |
| \<cache-path ... />          | getCacheDir()                             | /data/data/{package}/cache                  |
| \<external-path ... />       | Environment.getExternalStorageDirectory() | /sdcard                                     |
| \<external-files-path ... /> | Context.getExternalFilesDir(String)       | /sdcard/Android/data/{package}/files/{name} |
| \<external-cache-path ... /> | Context.getExternalCacheDir()             | /sdcard/Android/data/{package}/cache        |

Tip

配置`<external-path name="xxxx" path="." />`可访问SD卡所有路径。

> 常见的link:\
> /data/user/0 → /data/data\
> /storage/emulated/0 → /sdcard

#### 2 在AndroidManifest.xml中添加如下provider[¶](https://blog.yorek.xyz/android/other/FileProvider/#2-androidmanifestxmlprovider) <a href="#2-androidmanifestxmlprovider" id="2-androidmanifestxmlprovider"></a>

```
<application
        ...>
        ...
        <provider
            android:name="android.support.v4.content.FileProvider"
            android:authorities="yorek.com.solitaire.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/filepaths"/>
        </provider>
        ...
    </application>
```

这里只需要替换`android:authorities`以及`android:resource`两个属性为自己的即可： - android:authorities基于自己控制的域名，比如你控制了`mydomain.com`，你应该使用`com.mydomain.fileprovider` - android:resource则为第一步中创建的xml文件

#### 3 在代码中使用[¶](https://blog.yorek.xyz/android/other/FileProvider/#3) <a href="#3" id="3"></a>

在代码中使用时，要注意判断正在运行设备的版本号

```
private void chooseFromCameraInternal() {
    Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);
    Uri imageUri;
    if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.M) {
        imageUri = Uri.fromFile(mTempImgFile);
    } else {    // support for N
        imageUri = FileProvider.getUriForFile(mContext, "yorek.com.solitaire.fileprovider", mTempImgFile);
        intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    }       
    intent.putExtra(MediaStore.EXTRA_OUTPUT, imageUri);
    mContext.startActivityForResult(intent, CODE_CHOOSE_FROM_CAMERA);
}
```

最后更新: 2020年1月14日\

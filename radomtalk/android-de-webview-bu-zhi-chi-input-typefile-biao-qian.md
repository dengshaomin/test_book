# Android 的 webview 不支持 input type=file 标签

项目中遇到 H5 的 `input type="file"` 标签在 Android 的 webview 中失效，查了一下是由于安全原因将其屏蔽了。重写 webview 的 WebChromeClient 可以解决。

### WebView 设置 WebChromeClient <a href="#webview-she-zhi-webchromeclient" id="webview-she-zhi-webchromeclient"></a>

重写 WebChromeClient 中关于文件选择的方法，onShowFileChooser 和 openFileChooser。

```java
mWebView.setWebChromeClient(new WebChromeClient() {

    // For 3.0+ Devices (Start)
    // onActivityResult attached before constructor
    protected void openFileChooser(ValueCallback uploadMsg, String acceptType) {
        mUploadMessage = uploadMsg;
        Intent i = new Intent(Intent.ACTION_GET_CONTENT);
        i.addCategory(Intent.CATEGORY_OPENABLE);
        i.setType("image/*");
        startActivityForResult(Intent.createChooser(i, "File Browser"), FILE_CHOOSER_RESULT_CODE);
    }

    // For Lollipop 5.0+ Devices
    (Build.VERSION_CODES.LOLLIPOP)
    
    public boolean onShowFileChooser(WebView webView, ValueCallback<Uri[]> filePathCallback, FileChooserParams fileChooserParams) {
        if (uploadMessage != null) {
            uploadMessage.onReceiveValue(null);
            uploadMessage = null;
        }
        uploadMessage = filePathCallback;
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT)
        intent.type = "audio/*"
        intent.addCategory(Intent.CATEGORY_OPENABLE)
        try {
            startActivityForResult(intent, REQUEST_SELECT_FILE);
        } catch (ActivityNotFoundException e) {
            uploadMessage = null;
            Toast.makeText(getContext(), "Cannot Open File Chooser", Toast.LENGTH_LONG).show();
            return false;
        }
        return true;
    }

    //For Android 4.1 only
    protected void openFileChooser(ValueCallback<Uri> uploadMsg, String acceptType, String capture) {
        mUploadMessage = uploadMsg;
        Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        intent.addCategory(Intent.CATEGORY_OPENABLE);
        intent.setType("image/*");
        startActivityForResult(Intent.createChooser(intent, "File Browser"), FILE_CHOOSER_RESULT_CODE);
    }

    protected void openFileChooser(ValueCallback<Uri> uploadMsg) {
        mUploadMessage = uploadMsg;
        Intent i = new Intent(Intent.ACTION_GET_CONTENT);
        i.addCategory(Intent.CATEGORY_OPENABLE);
        i.setType("image/*");
        startActivityForResult(Intent.createChooser(i, "File Chooser"), FILE_CHOOSER_RESULT_CODE);
    }

});
```

### 选择结果的回调 <a href="#xuan-ze-jie-guo-de-hui-tiao" id="xuan-ze-jie-guo-de-hui-tiao"></a>

在 onActivityResult 中获取对应的选取文件的返回结果

```java

protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
        if (requestCode == REQUEST_SELECT_FILE) {
            if (uploadMessage == null) {
                return;
            }
            uploadMessage.onReceiveValue(WebChromeClient.FileChooserParams.parseResult(resultCode, data));
            uploadMessage = null;
        }
    } else if (requestCode == FILE_CHOOSER_RESULT_CODE) {
        if (null == mUploadMessage) {
            return;
        }
        // Use MainActivity.RESULT_OK if you're implementing WebView inside Fragment
        // Use RESULT_OK only if you're implementing WebView inside an Activity
        Uri result = data == null || resultCode != WebActivity.RESULT_OK ? null : data.getData();
        mUploadMessage.onReceiveValue(result);
        mUploadMessage = null;
    } else {
        Toast.makeText(getContext(), "选择图片失败", Toast.LENGTH_LONG).show();
    }

}
```

可以解决 3.1、4.1、5.0 以上的版本的问题。

### 文件选择器格式

{% tabs %}
{% tab title="同时设置多种格式" %}
```java
intent.setType("video/*;image/*");//同时选择视频和图片
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="常用格式" %}
```java
//{后缀名，MIME类型} 
{".3gp", "video/3gpp"}, 
{".apk", "application/vnd.android.package-archive"}, 
{".asf", "video/x-ms-asf"}, 
{".avi", "video/x-msvideo"}, 
{".bin", "application/octet-stream"}, 
{".bmp", "image/bmp"}, 
{".c", "text/plain"}, 
{".class", "application/octet-stream"}, 
{".conf", "text/plain"}, 
{".cpp", "text/plain"}, 
{".doc", "application/msword"}, 
{".docx", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"},
{".xls", "application/vnd.ms-excel"}, 
{".xlsx", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"}, 
{".exe", "application/octet-stream"}, 
{".gif", "image/gif"}, 
{".gtar", "application/x-gtar"}, 
{".gz", "application/x-gzip"}, 
{".h", "text/plain"}, 
{".htm", "text/html"}, 
{".html", "text/html"}, 
{".jar", "application/java-archive"}, 
{".java", "text/plain"}, 
{".jpeg", "image/jpeg"}, 
{".jpg", "image/jpeg"}, 
{".js", "application/x-javascript"}, 
{".log", "text/plain"}, 
{".m3u", "audio/x-mpegurl"}, 
{".m4a", "audio/mp4a-latm"}, 
{".m4b", "audio/mp4a-latm"}, 
{".m4p", "audio/mp4a-latm"}, 
{".m4u", "video/vnd.mpegurl"}, 
{".m4v", "video/x-m4v"}, 
{".mov", "video/quicktime"}, 
{".mp2", "audio/x-mpeg"}, 
{".mp3", "audio/x-mpeg"}, 
{".mp4", "video/mp4"}, 
{".mpc", "application/vnd.mpohun.certificate"}, 
{".mpe", "video/mpeg"}, 
{".mpeg", "video/mpeg"}, 
{".mpg", "video/mpeg"}, 
{".mpg4", "video/mp4"}, 
{".mpga", "audio/mpeg"}, 
{".msg", "application/vnd.ms-outlook"}, 
{".ogg", "audio/ogg"}, 
{".pdf", "application/pdf"}, 
{".png", "image/png"}, 
{".pps", "application/vnd.ms-powerpoint"}, 
{".ppt", "application/vnd.ms-powerpoint"}, 
{".pptx", "application/vnd.openxmlformats-officedocument.presentationml.presentation"},
{".prop", "text/plain"}, 
{".rc", "text/plain"}, 
{".rmvb", "audio/x-pn-realaudio"}, 
{".rtf", "application/rtf"}, 
{".sh", "text/plain"}, 
{".tar", "application/x-tar"}, 
{".tgz", "application/x-compressed"}, 
{".txt", "text/plain"}, 
{".wav", "audio/x-wav"}, 
{".wma", "audio/x-ms-wma"}, 
{".wmv", "audio/x-ms-wmv"}, 
{".wps", "application/vnd.ms-works"}, 
{".xml", "text/plain"}, 
{".z", "application/x-compress"}, 
{".zip", "application/x-zip-compressed"}, 
{"", "*/*"} 
};
```
{% endtab %}
{% endtabs %}

# webview校验ssl证书

校验逻辑为：在发生ssl error时对比信任证书与当前打开网页证书的指纹数组，如果一直则执行handler.proceed继续加载网页，否则执行handler.cancel结束加载

触发各种webview错误：[https://badssl.com/](https://badssl.com/)

## 获取网站证书

mac环境下，chorme按下f12->Security->View certificate

![](<../.gitbook/assets/image (204).png>)

### 通过在线工具获取

拖动证书大图标到本地路径，通过openssl命令将cer证书格式转换成pem格式

```
openssl x509 -inform der -in demo.cer -out demo.pem
```

![](<../.gitbook/assets/image (233).png>)

在线转换网址：[https://www.myssl.cn/tools/downloadchain.html](https://www.myssl.cn/tools/downloadchain.html)

拷贝pem证书所有内容进行转换，在指纹(sha256)点击转换成java数组

![](<../.gitbook/assets/image (159).png>)



### 通过sha-256计算

查看证书细节，拖到最后复制SHA-256

![](<../.gitbook/assets/image (87).png>)

通过一下代码将16进制的SHA-256转成byte指纹数据

```kotlin
fun hexToBytes(hexString: String?): ByteArray? {
            if (hexString == null || hexString.trim { it <= ' ' }.length == 0) return null
            var str = hexString.replace(" ", "")
            val length = str.length / 2
            val hexChars = str.toCharArray()
            val bytes = ByteArray(length)
            val hexDigits = "0123456789abcdef"
            for (i in 0 until length) {
                val pos = i * 2 // 两个字符对应一个byte
                val h = hexDigits.indexOf(hexChars[pos]) shl 4 // 注1
                val l = hexDigits.indexOf(hexChars[pos + 1]) // 注2
                if (h == -1 || l == -1) { // 非16进制字符
                    return null
                }
                bytes[i] = (h or l).toByte()
            }
            return bytes
        }
```

## 重写onReceivedSslError

选择需要验证的错误类型并对证书进行校验

```kotlin
webView.webViewClient = object : WebViewClient() {
                        override fun onReceivedSslError(
                view: WebView?,
                handler: SslErrorHandler?,
                error: SslError?
            ) {
                error?.let {
                    when (error.primaryError) {
                        SslError.SSL_DATE_INVALID,// 日期不正确
                        SslError.SSL_EXPIRED, // webview BUG
                        SslError.SSL_INVALID, // 根证书丢失
                        SslError.SSL_UNTRUSTED -> {
                            if (chkMySSLCNCert(error.getCertificate())) {
                                // 如果证书一致，忽略错误
                                handler?.proceed();
                                return
                            }
                        }
                    }
                }
                super.onReceivedSslError(view, handler, error)
            }
        }
```

## openssl转换证书

在线转换：[https://www.chinassl.net/ssltools/convert-ssl.html](https://www.chinassl.net/ssltools/convert-ssl.html)


# ReProguard

retrace的工具会因是否开启R8优化而不同，如果是开发APM平台可以通过mapping.txt文件来判断app是否使用了R8；proguard-rules.pro中增加配置保持行号解析：

```
-keepattributes SourceFile,LineNumberTable
```

模拟crash code，注意crash发生在line：35

![](<../.gitbook/assets/image (162).png>)

#### 未使用R8

mapping文件格式

![disable r8](<../.gitbook/assets/image (90).png>)

retrace工具目录：

```
Library/Android/sdk/tools/proguard/bin
Library/Android/sdk/tools/proguard/lib
```

```
retrace.sh -verbose path-to-mapping-file [path-to-stack-trace-file]
```

#### 开启了R8

mapping文件格式

![R8](<../.gitbook/assets/image (115).png>)

retrace工具目录为：Library/rAndroid/sdk/cmdline-tools/latest/bin/retrace

```
retrace path-to-mapping-file [path-to-stack-trace-file] [options] 
```

![](<../.gitbook/assets/image (101).png>)

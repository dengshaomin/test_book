---
description: 转：https://www.iteye.com/topic/1113183
---

# Fastjson技术内幕

JSON协议使用方便，越来越流行。JSON的处理器有很多，为什么需要再写一个呢？因为我们需要一个性能很好的JSON Parser，希望JSON Parser的性能有二进制协议一样好，比如和protobuf一样，这可不容易，但确实做到了。有人认为这从原理上就是不可能的，但是计算机乃实践科学，看实际的结果比原理推导更重要。\
\
这篇文章告诉大家：\
\* Fastjson究竟有多快\
\* 为什么Fastjson这么快\
\* 你能用Fastjson来做什么！\
\* 如何获得fastjson？\


首先，Fastjson究竟有多快？\
我们看一下使用[https://github.com/eishay/jvm-serializers/](https://github.com/eishay/jvm-serializers/)提供的程序进行测试得到的结果：\


|          | 序列化时间 | 反序列化时间 | 大小  | 压缩后大小 |
| -------- | ----- | ------ | --- | ----- |
| java序列化  | 8654  | 43787  | 889 | 541   |
| hessian  | 6725  | 10460  | 501 | 313   |
| protobuf | 2964  | 1745   | 239 | 149   |
| thrift   | 3177  | 1949   | 349 | 197   |
| avro     | 3520  | 1948   | 221 | 133   |
| json-lib | 45788 | 149741 | 485 | 263   |
| jackson  | 3052  | 4161   | 503 | 271   |
| fastjson | 2595  | 1472   | 468 | 251   |

\
这是一个468bytes的JSON Bytes测试，从测试结果来看，无论序列化和反序列化，Fastjson超越了protobuf，可以当之无愧fast! 它比java deserialize快超过30多倍，比json-lib快100倍。由于Fastjson的存在，你可以放心使用json统一协议，达到文本协议的可维护性，二进制协议的性能。\
\
JSON处理主要包括两个部分，serialize和deserialize。serialize就是把Java对象变成JSON String或者JSON Bytes。Deserialize是把JSON String或者Json Bytes变成java对象。其实这个过程有些JSON库是分三部分的，json string <－-> json tree <－-> java object。Fastjson也支持这种转换方式，但是这种转换方式因为有多余的步骤，性能不好，不推荐使用。\
\
为什么Fastjson能够做到这么快？\
一、Fastjson中Serialzie的优化实现\
**1、自行编写类似StringBuilder的工具类SerializeWriter。**\
把java对象序列化成json文本，是不可能使用字符串直接拼接的，因为这样性能很差。比字符串拼接更好的办法是使用java.lang.StringBuilder。StringBuilder虽然速度很好了，但还能够进一步提升性能的，fastjson中提供了一个类似StringBuilder的类com.alibaba.fastjson.serializer.SerializeWriter。\
\
SerializeWriter提供一些针对性的方法减少数组越界检查。例如public void writeIntAndChar(int i, char c) {}，这样的方法一次性把两个值写到buf中去，能够减少一次越界检查。目前SerializeWriter还有一些关键的方法能够减少越界检查的，我还没实现。也就是说，如果实现了，能够进一步提升serialize的性能。\
\
**2、使用ThreadLocal来缓存buf。**\
这个办法能够减少对象分配和gc，从而提升性能。SerializeWriter中包含了一个char\[] buf，每序列化一次，都要做一次分配，使用ThreadLocal优化，能够提升性能。\
\
**3、使用asm避免反射**\
获取java bean的属性值，需要调用反射，fastjson引入了asm的来避免反射导致的开销。fastjson内置的asm是基于objectweb asm 3.3.1改造的，只保留必要的部分，fastjson asm部分不到1000行代码，引入了asm的同时不导致大小变大太多。\
\
**使用一个特殊的IdentityHashMap优化性能。**\
fastjson对每种类型使用一种serializer，于是就存在class -> JavaBeanSerizlier的映射。fastjson使用IdentityHashMap而不是HashMap，避免equals操作。我们知道HashMap的算法的transfer操作，并发时可能导致死循环，但是ConcurrentHashMap比HashMap系列会慢，因为其使用volatile和lock。fastjson自己实现了一个特别的IdentityHashMap，去掉transfer操作的IdentityHashMap，能够在并发时工作，但是不会导致死循环。\
\
**5、缺省启用sort field输出**\
json的object是一种key/value结构，正常的hashmap是无序的，fastjson缺省是排序输出的，这是为deserialize优化做准备。\
\
**6、集成jdk实现的一些优化算法**\
在优化fastjson的过程中，参考了jdk内部实现的算法，比如int to char\[]算法等等。\
\
二、fastjson的deserializer的主要优化算法\
deserializer也称为parser或者decoder，fastjson在这方面投入的优化精力最多。\
**1、读取token基于预测。**\
所有的parser基本上都需要做词法处理，json也不例外。fastjson词法处理的时候，使用了基于预测的优化算法。比如key之后，最大的可能是冒号":"，value之后，可能是有两个，逗号","或者右括号"}"。在com.alibaba.fastjson.parser.JSONScanner中提供了这样的方法：\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
public void nextToken(int expect) {  
    for (;;) {  
        switch (expect) {  
            case JSONToken.COMMA: //   
                if (ch == ',') {  
                    token = JSONToken.COMMA;  
                    ch = buf[++bp];  
                    return;  
                }  
  
                if (ch == '}') {  
                    token = JSONToken.RBRACE;  
                    ch = buf[++bp];  
                    return;  
                }  
  
                if (ch == ']') {  
                    token = JSONToken.RBRACKET;  
                    ch = buf[++bp];  
                    return;  
                }  
  
                if (ch == EOI) {  
                    token = JSONToken.EOF;  
                    return;  
                }  
                break;  
        // ... ...  
    }  
} 
```

\
从上面摘抄下来的代码看，基于预测能够做更少的处理就能够读取到token。\
\
**2、sort field fast match算法**\
fastjson的serialize是按照key的顺序进行的，于是fastjson做deserializer时候，采用一种优化算法，就是假设key/value的内容是有序的，读取的时候只需要做key的匹配，而不需要把key从输入中读取出来。通过这个优化，使得fastjson在处理json文本的时候，少读取超过50%的token，这个是一个十分关键的优化算法。基于这个算法，使用asm实现，性能提升十分明显，超过300％的性能提升。\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
{ "id" : 123, "name" : "魏加流", "salary" : 56789.79}  
  ------      --------          ----------    
```

\
在上面例子看，虚线标注的三个部分是key，如果key\_id、key\_name、key\_salary这三个key是顺序的，就可以做优化处理，这三个key不需要被读取出来，只需要比较就可以了。\
\
这种算法分两种模式，一种是快速模式，一种是常规模式。快速模式是假定key是顺序的，能快速处理，如果发现不能够快速处理，则退回常规模式。保证性能的同时，不会影响功能。\
\
在这个例子中，常规模式需要处理13个token，快速模式只需要处理6个token。\
\
实现sort field fast match算法的代码在这个类com.alibaba.fastjson.parser.deserializer.ASMDeserializerFactory\
[http://code.alibabatech.com/svn/fastjson/trunk/fastjson/src/main/java/com/alibaba/fastjson/parser/deserializer/ASMDeserializerFactory.java](http://code.alibabatech.com/svn/fastjson/trunk/fastjson/src/main/java/com/alibaba/fastjson/parser/deserializer/ASMDeserializerFactory.java)，是使用asm针对每种类型的VO动态创建一个类实现的。\
这里是有一个用于演示sort field fast match算法的代码：\
[http://code.alibabatech.com/svn/fastjson/trunk/fastjson/src/test/java/data/media/ImageDeserializer.java](http://code.alibabatech.com/svn/fastjson/trunk/fastjson/src/test/java/data/media/ImageDeserializer.java)\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
// 用于快速匹配的每个字段的前缀  
char[] size_   = "\"size\":".toCharArray();  
char[] uri_    = "\"uri\":".toCharArray();  
char[] titile_ = "\"title\":".toCharArray();  
char[] width_  = "\"width\":".toCharArray();  
char[] height_ = "\"height\":".toCharArray();  
  
// 保存parse开始时的lexer状态信息  
int mark = lexer.getBufferPosition();  
char mark_ch = lexer.getCurrent();  
int mark_token = lexer.token();  
  
int height = lexer.scanFieldInt(height_);  
if (lexer.matchStat == JSONScanner.NOT_MATCH) {  
    // 退出快速模式, 进入常规模式  
    lexer.reset(mark, mark_ch, mark_token);  
    return (T) super.deserialze(parser, clazz);  
}  
  
String value = lexer.scanFieldString(size_);  
if (lexer.matchStat == JSONScanner.NOT_MATCH) {  
    // 退出快速模式, 进入常规模式  
    lexer.reset(mark, mark_ch, mark_token);  
    return (T) super.deserialze(parser, clazz);  
}  
Size size = Size.valueOf(value);  
  
// ... ...  
  
// batch set  
Image image = new Image();  
image.setSize(size);  
image.setUri(uri);  
image.setTitle(title);  
image.setWidth(width);  
image.setHeight(height);  
  
return (T) image;  
```

\
\
**3、使用asm避免反射**\
deserialize的时候，会使用asm来构造对象，并且做batch set，也就是说合并连续调用多个setter方法，而不是分散调用，这个能够提升性能。\
\
**4、对utf-8的json bytes，针对性使用优化的版本来转换编码。**\
这个类是com.alibaba.fastjson.util.UTF8Decoder，来源于JDK中的UTF8Decoder，但是它使用ThreadLocal Cache Buffer，避免转换时分配char\[]的开销。\
ThreadLocal Cache的实现是这个类com.alibaba.fastjson.util.ThreadLocalCache。第一次1k，如果不够，会增长，最多增长到128k。\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
//代码摘抄自com.alibaba.fastjson.JSON  
public static final <T> T parseObject(byte[] input, int off, int len, CharsetDecoder charsetDecoder, Type clazz,  
                                      Feature... features) {  
    charsetDecoder.reset();  
  
    int scaleLength = (int) (len * (double) charsetDecoder.maxCharsPerByte());  
    char[] chars = ThreadLocalCache.getChars(scaleLength); // 使用ThreadLocalCache，避免频繁分配内存  
  
    ByteBuffer byteBuf = ByteBuffer.wrap(input, off, len);  
    CharBuffer charByte = CharBuffer.wrap(chars);  
    IOUtils.decode(charsetDecoder, byteBuf, charByte);  
  
    int position = charByte.position();  
  
    return (T) parseObject(chars, position, clazz, features);  
}  

```

\
**6、symbolTable算法。**\
我们看xml或者javac的parser实现，经常会看到有一个这样的东西symbol table，它就是把一些经常使用的关键字缓存起来，在遍历char\[]的时候，同时把hash计算好，通过这个hash值在hashtable中来获取缓存好的symbol，避免创建新的字符串对象。这种优化在fastjson里面用在key的读取，以及enum value的读取。这是也是parse性能优化的关键算法之一。\
\
以下是摘抄自JSONScanner类中的代码，这段代码用于读取类型为enum的value。\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
int hash = 0;  
for (;;) {  
    ch = buf[index++];  
    if (ch == '\"') {  
        bp = index;  
        this.ch = ch = buf[bp];  
        strVal = symbolTable.addSymbol(buf, start, index - start - 1, hash); // 通过symbolTable来获得缓存好的symbol，包括fieldName、enumValue  
        break;  
    }  
      
    hash = 31 * hash + ch; // 在token scan的过程中计算好hash  
  
    // ... ...  
}  
```

\
\
我们能用fastjson来作什么？\
1、替换其他所有的json库，java世界里没有其他的json库能够和fastjson可相比了。\
2、使用fastjson的序列化和反序列化替换java serialize，java serialize不单性能慢，而且体制大。\
3、使用fastjson替换hessian，json协议和hessian协议大小差不多一样，而且fastjson性能优越，10倍于hessian\
4、把fastjson用于memached缓存对象数据。\
\
\
如何获得fastjson\
h3. 官方网站\
Fastjson是开源的，基于Apache 2.0协议。你可以在官方网站了解最新信息。\
[http://code.alibabatech.com/wiki/display/FastJSON/Home](http://code.alibabatech.com/wiki/display/FastJSON/Home)\
\
**maven用户**\
\* Maven仓库 [http://code.alibabatech.com/mvn/releases/](http://code.alibabatech.com/mvn/releases/)\
Java代码  ![收藏代码](https://www.iteye.com/images/icon\_star.png)

```java
<dependency>  
     <groupId>com.alibaba</groupId>  
     <artifactId>fastjson</artifactId>  
     <version>1.1.2</version>  
</dependency>  
```

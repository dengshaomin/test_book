---
description: 转：https://www.jianshu.com/p/dfc18278b053
---

# 音视频预加载：AndroidVideoCache源码详解

github:[https://github.com/Doikki/DKVideoPlayer](https://github.com/Doikki/DKVideoPlayer)

## 前言

&#x20;   为什么写这个文章？因为之前做过一些短视频方面相关的应用，特别是在播放优化上面踩过一点坑。优化的主要目的为了让视频达到秒开，视频的预加载等，并在用户多次播放的过程中能减少流量的消耗。最初我们也做了一些播放器相关的优化，比如说我们优化播放器的内核，改变播放器起播的时机，使一有数据就开始起播。（Android自身的系统播放器需要满足一个GOP的大小才能起播）；控制视频的编码与压缩方式，使视频能又小又清晰。但是还是有一些问题没有解决，比如初始化播放器setUrl并start后，播放器需要开启网络连接并从服务端下载数据，这本身是一个耗时的操作，而且播放器下载数据播放完就会把数据从缓冲区清除掉，每次重复播放视频都会重新连接网络下载，这显然是不可取的。那么现在我们就要解决两个问题，第一，重复播放的视频应该走缓存而不是重新下载，第二，提前下载视频，使视频能达到起播态。\
&#x20;   而重复的视频边播变缓存的策略，使视频loopping时播放器不需要重新下载数据。秉着不需要重新造轮子的思想，我们先直接采用国外大神提供的AndroidVideoCache这个开源库，它是一种透明代理，也称本地代理的方案，就是拦截掉播放器的网络请求，代理播放器的下载功能，将下载的文件保存到文件，然后将文件中的数据返回给播放器。后面的文章，我将改造这个库，使它支持我们类似于抖音和快手的预加载功能。

## 特性

1.目前AndroidVideoCache只适合url直连数据，例如短视频领域用的比较多的mp4链接。像HLS，m3u8等支持的不是很友好。\
2.能离线加载资源\
3.支持多个播放器共享一个url下载\
4.缓存管理，支持设置最大缓存数和缓存数量限制

## 使用方式

我们需要先定义一个`VideoProxyManager`的单例类，在单例初始化的时候我们配置本地代理的各种参数，例如下图中的缓存的数量和大小。还可以配置缓存文件名生成的格式，缓存文件的路径配置等等。\
然后我们在使用Ijk或者系统提供的VideoView的地方按照如下方式调用，这样播放器就会直接走我们代理进行缓存了。

```
    videoView.setVideoPath(VideoProxyManager. getInstance(). getProxyUrl(VIDEO_URL));
```

```
    public class VideoProxyManager {

    private HttpProxyCacheServer httpProxyCacheServer;
    private static final long DEFAULT_MAX_SIZE = 600 * 1024 * 1024; //最大缓存容量
    public static int DEFAULT_MAX_FILE_COUNT = 50; //最大缓存数量

    public static boolean isUseCache = true; // 全局是否使用缓存,Server端下发配置

    private VideoProxyManager() {
    }

    private static class VideoProxyManagerHolder {
        private static VideoProxyManager videoProxyManager = new VideoProxyManager();
    }

    public static VideoProxyManager getInstance() {
        return VideoProxyManagerHolder.videoProxyManager;
    }

    public void init(Context context) {
        httpProxyCacheServer = new HttpProxyCacheServer.Builder(context).maxCacheSize(DEFAULT_MAX_SIZE)
            .maxCacheFilesCount(DEFAULT_MAX_FILE_COUNT)
            .build();
    }
    
    /**
      * 传给播放器的url替换成代理的url
    **/

    public String getProxyUrl(String url) {
        if (TextUtils.isEmpty(url) || !isUseCache) {
            return url;
        }
        return httpProxyCacheServer.getProxyUrl(url);
    }

    /**
     * 需要非常小心，可能会误杀多播放器共享一个url的情况
     * @param url
     */
    public void shutdownOneClient(String url) {
        if (TextUtils.isEmpty(url)) {
            return;
        }
        httpProxyCacheServer.shutdownOneClient(url);
    }

    public void shutdown() {
        httpProxyCacheServer.shutdown();
    }
}
```

## 源码分析

在一般的播放器请求数据的模型中，播放器直接通过url连接到远程服务器，播放器下载后的数据直接交给播放器缓冲区，数据使用完了以后直接淘汰掉。\
![](https://upload-images.jianshu.io/upload\_images/4993639-6a67cba9919f0c53.png?imageMogr2/auto-orient/strip|imageView2/2/w/954/format/webp)一般的播放器与远程服务器的交互

如果我们在播放器与远程server中间插入一个本地透明代理，这样透明代理就可以接管播放器的请求，透明代理从远程server下载完数据就可以先保存在本地，然后把所需要的数据交给播放器。类似于我们Charles抓包这样。下一次播放器请求相同的数据，就可以在本地代理这里找到对应的缓存文件，直接返回。\
![](https://upload-images.jianshu.io/upload\_images/4993639-8ba9feff51973533.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)在播放器和远程服务器中间插入透明代理，拦截下载

我们首先从入口函数这里开始分析，首先是`HttpProxyCacheServer`这个类，这是一个入口类。我们也通过`VideoProxyManager`对其进行了单例的包装。下面的代码主要是完成视频信息的数据库和缓存的设置。

```
public Builder(Context context) {
            this.sourceInfoStorage = SourceInfoStorageFactory.newSourceInfoStorage(context);//数据库，存储视频原始url、mine信息、视频length
            this.cacheRoot = StorageUtils.getIndividualCacheDirectory(context);//缓存文件的存储路径
            this.diskUsage = new TotalSizeLruDiskUsage(DEFAULT_MAX_SIZE); //LRU缓存设置，设置最大缓存数量和总大小
            this.fileNameGenerator = new Md5FileNameGenerator(); //文件缓存名
            this.headerInjector = new EmptyHeadersInjector(); //在请求中增加head信息
        }
```

接下来我们再来分析`HttpProxyCacheServer`的初始化:

```
 private HttpProxyCacheServer(Config config) {
        this.config = checkNotNull(config);
        try {
            InetAddress inetAddress = InetAddress.getByName(PROXY_HOST);
            this.serverSocket = new ServerSocket(0, 8, inetAddress);
            this.port = serverSocket.getLocalPort();
            IgnoreHostProxySelector.install(PROXY_HOST, port);
            CountDownLatch startSignal = new CountDownLatch(1);
            this.waitConnectionThread = new Thread(new WaitRequestsRunnable(startSignal));
            this.waitConnectionThread.start();
            startSignal.await(); // freeze thread, wait for server starts
            this.pinger = new Pinger(PROXY_HOST, port);
            Log.e(TAG,"HttpProxyCacheServer Proxy cache server started. Is it alive? " + isAlive());
        } catch (IOException | InterruptedException e) {
            socketProcessor.shutdown();
            Log.e(TAG,"HttpProxyCacheServer 线程池关闭 ");
            throw new IllegalStateException("Error starting local proxy server", e);
        }
    }
```

上面代码主要是建立一个本地的服务器，地址为127.0.0.1，端口为获取的一个本地的可用端口。注意这个地使用了`CountDownLatch`来保证线程之间的顺序执行、我们重点分析`WaitRequestsRunnable`

```
private final class WaitRequestsRunnable implements Runnable {

        private final CountDownLatch startSignal;

        public WaitRequestsRunnable(CountDownLatch startSignal) {
            this.startSignal = startSignal;
        }
        @Override
        public void run() {
            startSignal.countDown();
            waitForRequest();
        }
    }

```

继续跟进`waitForRequest()`这个方法

```
 private void waitForRequest() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                Socket socket = serverSocket.accept();
               Log.e(TAG,"HttpProxyCacheServer Accept new socket " + socket);
                socketProcessor.submit(new SocketProcessorRunnable(socket));
            }
        } catch (IOException e) {
            onError(new ProxyCacheException("HttpProxyCacheServer Error during waiting connection", e));
        }
    }

```

当播放器通过proxyUrl连接到代理服务器时，`serverSocket.accept()`就会建立一个可用的Socket连接。`socketProcessor`是一个固定线程池，我们重点关注`new SocketProcessorRunnable(socket)`

```
private final class SocketProcessorRunnable implements Runnable {

        private final Socket socket;

        public SocketProcessorRunnable(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            processSocket(socket);
        }
    }

```

继续跟进`processSocket(socket)`

```
   private void processSocket(Socket socket) {
        try {
            
            GetRequest request = GetRequest.read(socket.getInputStream());
            String url = ProxyCacheUtils.decode(request.uri);
            if (pinger.isPingRequest(url)) {
                pinger.responseToPing(socket);
            } else {
                HttpProxyCacheServerClients clients = getClients(url);
                clients.processRequest(request, socket);
            }
        } catch (SocketException e) {
            // There is no way to determine that client closed connection http://stackoverflow.com/a/10241044/999458
            // So just to prevent log flooding don't log stacktrace
            Log.e("TAG","Closing socket… Socket is closed by client.");
        } catch (ProxyCacheException | IOException e) {
            onError(new ProxyCacheException("Error processing request", e));
        } finally {
            releaseSocket(socket);
            Log.e("TAG","Opened connections: " + getClientsCount());
        }
    }

```

这个地又冒出了两个类`GetRequest`和`HttpProxyCacheServerClients`,`GetRequest`主要是根据Socket中InputStream来构建我们请求的。它里面主要保存着我们对应的url，请求的起始位置rangeOffset，和是否是分段下载partial。`HttpProxyCacheServerClients`可以理解为对应一个具体url视频的客户端，视频的下载，缓存，以及最后将数据交给播放器，都是在这里面处理的。上面的代码中，我们重点关注`clients.processRequest(request, socket)`，所有的秘密都藏在这里面。

```
    public void processRequest(GetRequest request, Socket socket) throws ProxyCacheException, IOException {
        startProcessRequest();
        try {
            clientsCount.incrementAndGet();
            proxyCache.processRequest(request, socket);
        } finally {
            finishProcessRequest();
        }
    }
```

上图中，`startProcessRequest()`主要是创建一个`HttpProxyCache`，我们来看看`HttpProxyCache`是如何创建的

```
private HttpProxyCache newHttpProxyCache() throws ProxyCacheException {
        HttpUrlSource source = new HttpUrlSource(url,this, config.sourceInfoStorage, config.headerInjector);
        FileCache cache = new FileCache(config.generateCacheFile(url), config.diskUsage);
        HttpProxyCache httpProxyCache = new HttpProxyCache(source, cache);
        httpProxyCache.registerCacheListener(uiCacheListener);
        return httpProxyCache;
    }
```

可以看出`HttpProxyCache`里面主要由两部分组成，一个为`HttpUrlSource`即网络下载部分，二为`FileCache`即文件缓存部分。我们继续跟踪上上图的`proxyCache.processRequest(request, socket)`

```
    public void processRequest(GetRequest request, Socket socket) throws IOException, ProxyCacheException {
        OutputStream out = new BufferedOutputStream(socket.getOutputStream());
        String responseHeaders = newResponseHeaders(request);
        out.write(responseHeaders.getBytes("UTF-8"));

        long offset = request.rangeOffset;
        if (isUseCache(request)) {
            responseWithCache(out, offset);
        } else {
            responseWithoutCache(out, offset);
        }
    }
```

上图中，代码的前三行主要是向播放器返回 Head信息，我们来看看`isUseCache(request)`这个方法

```
private boolean isUseCache(GetRequest request) throws ProxyCacheException {
        long sourceLength = source.length();
        boolean sourceLengthKnown = sourceLength > 0;
        long cacheAvailable = cache.available();
        // do not use cache for partial requests which too far from available cache. It seems user seek video.
        return !sourceLengthKnown || !request.partial || request.rangeOffset <= cacheAvailable + sourceLength * NO_CACHE_BARRIER;
    }
```

这个方法其实挺重要的，它决定了我们是否要使用缓存。之前有人问我，为啥我从前面一下子seek到后面，没走缓存啊。答案就在这里，`request.rangeOffset <= cacheAvailable + sourceLength * NO_CACHE_BARRIER`。说的更加直白点，就是seek超过视频总长的20%\
则跳过缓存。若是seek在20%总长以内，则会把seek部分的全部下载完全后再把对应的部分交给播放器。至于作者为啥这么设计，我后面总结的时候会说。\
我们重点关注`responseWithCache(out, offset)`的情况，因为`responseWithoutCache(out, offset)`只是少了一个缓存，其它的逻辑都一样

```
    private void responseWithCache(OutputStream out, long offset) throws ProxyCacheException, IOException {
        byte[] buffer = new byte[DEFAULT_BUFFER_SIZE];
        int readBytes;
        while ((readBytes = read(buffer, offset, buffer.length)) != -1) {
            if(out!=null) {
                out.write(buffer, 0, readBytes);
            }
            offset += readBytes;
        }
        if(out!=null) {
            out.flush();
        }
    }
```

我们重点看 `read(buffer, offset, buffer.length)`这个方法

```
 public int read(byte[] buffer, long offset, int length) throws ProxyCacheException {
        ProxyCacheUtils.assertBuffer(buffer, offset, length);
        while (!cache.isCompleted() && cache.available() < (offset + length) && !stopped) {
            readSourceAsync();
            waitForSourceData();
            checkReadSourceErrorsCount();
        }
        int read = cache.read(buffer, offset, length);
        if (cache.isCompleted() && percentsAvailable != 100) {
            percentsAvailable = 100;
            onCachePercentsAvailableChanged(100);
        }
        return read;
    }
```

其中`readSourceAsync()`主要是判断缓存中是否存在，若不存在则去下载。`cache.read(buffer, offset, length)`主要是往文件中存数据。我们来看`readSourceAsync()`的实现

```
    private synchronized void readSourceAsync() throws ProxyCacheException {
        boolean readingInProgress = sourceReaderThread != null && sourceReaderThread.getState() != Thread.State.TERMINATED;
        if (!stopped && !cache.isCompleted() && !readingInProgress) {
            sourceReaderThread = new Thread(new SourceReaderRunnable(), "Source reader for " + source);
            sourceReaderThread.start();
        }
    }
```

继续跟进`SourceReaderRunnable()`

```
    private class SourceReaderRunnable implements Runnable {

        @Override
        public void run() {
            readSource();
        }
    }
```

继续跟进`readSource()`

```
    private void readSource() {
        long sourceAvailable = -1;
        long offset = 0;
        try {
            offset = cache.available();
            source.open(offset);
            sourceAvailable = source.length();
            byte[] buffer = new byte[ProxyCacheUtils.DEFAULT_BUFFER_SIZE];
            int readBytes;
            while ((readBytes = source.read(buffer)) != -1) {
                synchronized (stopLock) {
                    if (isStopped()) {
                        return;
                    }
                    cache.append(buffer, readBytes);
                }
                offset += readBytes;
                notifyNewCacheDataAvailable(offset, sourceAvailable);
            }
            tryComplete();
            onSourceRead();
        } catch (Throwable e) {
            readSourceErrorsCount.incrementAndGet();
            onError(e);
        } finally {
            closeSource();
            notifyNewCacheDataAvailable(offset, sourceAvailable);
        }
    }
```

上图中分别调用了 `HttpUrlSource`的`source.open(offset)`,`source.read(buffer)`。`source.open(offset)`主要是打开`HttpURLConnection`连接，获取视频的mine信息，视频的length信息，并存到数据库中。

```
public void open(long offset) throws ProxyCacheException {
        try {
            connection = openConnection(offset, -1);
            String mime = connection.getContentType();
            inputStream = new BufferedInputStream(connection.getInputStream(), DEFAULT_BUFFER_SIZE);
            long length = readSourceAvailableBytes(connection, offset, connection.getResponseCode());
            this.sourceInfo = new SourceInfo(sourceInfo.url, length, mime);
            this.sourceInfoStorage.put(sourceInfo.url, sourceInfo);
        } catch (IOException e) {
            throw new ProxyCacheException("Error opening connection for " + sourceInfo.url + " with offset " + offset, e);
        }
    }
```

```
private HttpURLConnection openConnection(long offset, int timeout) throws IOException, ProxyCacheException {
        HttpURLConnection connection;
        boolean redirected;
        int redirectCount = 0;
        String url = this.sourceInfo.url;
        do {
            LOG.debug("Open connection " + (offset > 0 ? " with offset " + offset : "") + " to " + url);
            connection = (HttpURLConnection) new URL(url).openConnection();
            injectCustomHeaders(connection, url);
            if (offset > 0) {
                connection.setRequestProperty("Range", "bytes=" + offset + "-");
            }
            if (timeout > 0) {
                connection.setConnectTimeout(timeout);
                connection.setReadTimeout(timeout);
            }
            int code = connection.getResponseCode();
            redirected = code == HTTP_MOVED_PERM || code == HTTP_MOVED_TEMP || code == HTTP_SEE_OTHER;
            if (redirected) {
                url = connection.getHeaderField("Location");
                redirectCount++;
                connection.disconnect();
            }
            if (redirectCount > MAX_REDIRECTS) {
                throw new ProxyCacheException("Too many redirects: " + redirectCount);
            }
        } while (redirected);
        return connection;
    }
```

上面处理了重定向，并根据`connection.setRequestProperty("Range", "bytes=" + offset + "-")`来建立连接。然后我们回到上上上图的`readSource()` 中`source.read(buffer)`这个方法

```
@Override
    public int read(byte[] buffer) throws ProxyCacheException {
        if (inputStream == null) {
            throw new ProxyCacheException("Error reading data from " + sourceInfo.url + ": connection is absent!");
        }
        try {
            return inputStream.read(buffer, 0, buffer.length);
        } catch (InterruptedIOException e) {
            throw new InterruptedProxyCacheException("Reading source " + sourceInfo.url + " is interrupted", e);
        } catch (IOException e) {
            throw new ProxyCacheException("Error reading data from " + sourceInfo.url, e);
        }
```

这个方法就比较简单了，直接从inputStream中取数据。取完数据我们再回到 前面的`readSource()`中调用`cache.append(buffer, readBytes)`将数据存到本地文件中。此时我们继续回到`responseWithCache(out, offset)`中

```
    private void responseWithCache(OutputStream out, long offset) throws ProxyCacheException, IOException {
        byte[] buffer = new byte[DEFAULT_BUFFER_SIZE];
        int readBytes;
        while ((readBytes = read(buffer, offset, buffer.length)) != -1) {
            if(out!=null) {
                out.write(buffer, 0, readBytes);
            }
            offset += readBytes;
        }
        if(out!=null) {
            out.flush();
        }
    }
```

上图中通过`out.write(buffer, 0, readBytes)`将数据返回给播放器。至此源码分析告一段落。

## 总结&问题

1.如果只是想做一个简单的mp4视频缓存的，AndroidVideoCache显然是足够的。\
2.我们可以看见这种缓存策略都是依赖于播放器的，在类似于快手和抖音的feed里面，脱离播放器的下载还做不到。\
3.AndroidVideoCache会一直连接网络下载数据，直到把数据下载完全。这肯定是不可取的。倘若一个5分钟100M的视频，我只看了20s就要把整个视频下载了，没必要吧。根据播放器的播放进度按需加载才是最优的。\
4.之前给大家讲过如果seek的超过总长的20%(前提是seek后的文件还不在缓存时seek)，播放器会不走缓存直接下载。其实我们可以脑补一下，假设一个1G的文件，我一下子seek到最后面，AndroidVideoCache怎么存哇。目前他的实现是一个RandomAccessFile,它不可能建一个1G空文件后然后只存最后一部分啊，否则得占用多少存储空间。看样子只能本地虚拟分片了。我目前正在思考这个功能的实现，完成后再分享给大家。

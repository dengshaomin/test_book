# 白话Okhttp拦截器调用过程

拦截器调用顺序一般为顺序调用，将上一个拦截器的返回作为下一个拦截器的输入，直到所有拦截器执行完毕，返回最终结果。这里通过for循环先模拟一个简单的拦截器调用过程，再模拟okhttp拦截器工作过程

### **简单顺序拦截器调用**

模拟屏蔽词：

```
 public static Response execute() {
        //模拟所有拦截器
        List<Interceptor> mInterceptorList = new ArrayList<Interceptor>() {{
            add(new FatherInterceptor());
            add(new GrandpaInterceptor());
        }};
        Chain chain = new RealInterceptor("我是你爸爸，我是你爷爷");
        Response response = null;
        for (Interceptor interceptor : mInterceptorList) {
            response = interceptor.intercept(chain);
            chain = new RealInterceptor(response.result);
        }
        return response;
    }


    public static class RealInterceptor implements Chain {

        private String params;

        public RealInterceptor(String params) {
            this.params = params;
        }

        @Override
        public Request request() {
            return new Request(params);
        }
    }

    public static class FatherInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            Response response = new Response(chain.request().params.replace("爸爸", "**"));
            return response;
        }
    }

    public static class GrandpaInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            Response response = new Response(chain.request().params.replace("爷爷", "**"));
            return response;
        }
    }

    public static interface Interceptor {

        Response intercept(Chain chain) throws RuntimeException;

        interface Chain {

            Request request();

        }
    }

    public static class Response {

        public Response(String result) {
            this.result = result;
        }

        public String result;
    }

    public static class Request {

        public Request(String params) {
            this.params = params;
        }

        public String params;
    }
```

当调用`execute()`将得到以下reponse:\
![image.png](https://upload-images.jianshu.io/upload\_images/6793917-dd17ba9c30ad4396.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)\
image.png

### **模拟OKhttp模拟器拦截调用**

okhttp拦截器调用核心步骤

* 构建`Request`，在当前调用链`Chain`中获取当前要执行的拦截器`Interceptor`
* 创建下一个调用链`RealInterceptor`,将该调用链作为当前拦截器的参数`Chain`
* 在当前拦截器中执行拦截动作`intercept(Chain chain)`，并调用下一个拦截器的`proceed`
* 在下一个调用链的`proceed(Request request)`判断是否所有拦截器执行完毕，如果没有执行完毕重复继续以上步骤,如果全部执行完毕抛出特定`Exception`，拦截器中捕捉该异常并构建最终返回结果。

```
 public static Response execute() {
        Chain chain = new RealInterceptor(0, new Request("request params"));
        Response response = chain.proceed(chain.request());
        return response;
    }


    public static class RealInterceptor implements Interceptor.Chain {
        //请求拦截器位置
        int index = 0;
        //请求体
        Request mRequest;

        public RealInterceptor(int count, Request request) {
            this.index = count;
            this.mRequest = request;
        }
        //模拟所有拦截器
        private List<Interceptor> mInterceptorList = new ArrayList<Interceptor>() {{
            add(new BridgeInterceptor());
            add(new CacheInterceptor());
            add(new ConnectionInterceptor());
            add(new ServerInterceptor());
        }};

        @Override
        public Request request() {
            return mRequest;
        }

        @Override
        public Response proceed(Request request) {
            if (index >= mInterceptorList.size()) {
                //所有拦截器执行结束，抛出特定异常
                throw new RuntimeException("interceptors execute finish");
            }
            //执行当前下标的拦截器
            Interceptor interceptor = mInterceptorList.get(index);
            //创建下一个调用链
            RealInterceptor next = new RealInterceptor(index + 1, request);
            //获取调用链结果
            Response response = interceptor.intercept(next);
            return response;
        }
    }

    public static class BridgeInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            chain.request().params += "-" + "BridgeInterceptor";
            Response response = null;
            try {
                response = chain.proceed(chain.request());
            } catch (Exception e){

            }
            if (response == null) {
                //执行结束构建最后的Response
                return new Response(chain.request().params);
            }
            return response;
        }
    }

    public static class CacheInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            chain.request().params += "-" + "CacheInterceptor";
            Response response = null;
            try {
                response = chain.proceed(chain.request());
            } catch (Exception e){

            }
            if (response == null) {
                //执行结束构建最后的Response
                return new Response(chain.request().params);
            }
            return response;
        }
    }

    public static class ConnectionInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            chain.request().params += "-" + "ConnectionInterceptor";
            Response response = null;
            try {
                response = chain.proceed(chain.request());
            } catch (Exception e){

            }
            if (response == null) {
                //执行结束构建最后的Response
                return new Response(chain.request().params);
            }
            return response;
        }
    }
    public static class ServerInterceptor implements Interceptor {

        @Override
        public Response intercept(Chain chain) {
            chain.request().params += "-" + "Reponse";
            Response response = null;
            try {
                response = chain.proceed(chain.request());
            } catch (Exception e){

            }
            if (response == null) {
                //执行结束构建最后的Response
                return new Response(chain.request().params);
            }
            return response;
        }
    }
    public static interface Interceptor {

        Response intercept(Chain chain) throws RuntimeException;

        interface Chain {

            Request request();

            Response proceed(Request request) throws RuntimeException;
        }
    }

    public static class Response {

        public Response(String result) {
            this.result = result;
        }

        public String result;
    }

    public static class Request {

        public Request(String params) {
            this.params = params;
        }

        public String params;
    }
```

当调用`execute()`将得到以下reponse:\
![image.png](https://upload-images.jianshu.io/upload\_images/6793917-7ea355f088cf598f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

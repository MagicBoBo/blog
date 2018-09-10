---
title: hybrid原理分析
date: 2018-08-13 21:42:36
tags: hybrid
---

简历里面写到了hybrid，结果被频繁的问到。。。发现简单的原理能答上来，问的深了就蒙蔽了。。。写一点博客，把自己对hybrid的理解写进去。
<!--more-->
# 概述
所谓hybrid就是一个webview，里面运行一个本地的页面，然后通过一个叫做jsbridge的东西来调用native方法。核心部分是jsbridge要怎么写。

# 安卓jsbridge实现方式

首先先牵涉到一个问题，android原生和js怎么互相通信，先看安卓：
## js调用安卓
```
方式1： webView.evaluateJavascript(js, callback);

方式2： webView.loadUrl("javascript:" + js);
```
方法1可以获得返回值，方法2native想获得返回值必须通过iframe再设置src才行，但方法1只在sdk19以上可以用，方法1的缺点在于同步执行js，方法2js的执行是异步的，但是需要在js端再利用隐藏的iframe传回返回值
## 安卓调用js
### 映射java方法到window对象中，然后直接调用对应的方法。
1. 首先获取一个webview，将setJavaScriptEnabled设为true
2. 之后调用addJavascriptInterface方法，传入一个java对象和一个js对象名。
3. 这个java对象的方法会被映射到对应的js对象当中。
4. 你可以使用window.js对象名.方法名来调用实例中的方法。

```
webView = (WebView)this.findViewById(R.id.webview);  
//启用javascript  
webView.getSettings().setJavaScriptEnabled(true);  
//加载本地html文件  
webView.loadUrl("file:///android_asset/test.html");  
/** 
 * 添加javascriptInterface 
 * 第一个参数：这里需要一个与js映射的java对象 
 * 第二个参数：该java对象被映射为js对象后在js里面的对象名，在js中要调用该对象的方法就是通过这个来调用 
 */  
webView.addJavascriptInterface(new JSInterface(), "jsi");

private final class JSInterface{  
  /** 
   * 注意这里的@JavascriptInterface注解， target是4.2以上都需要添加这个注解，否则无法调用 
   * @param text 
   */  
  @JavascriptInterface  
  public void showToast(String text){  
      Toast.makeText(getApplicationContext(), text, Toast.LENGTH_SHORT).show();  
  }  
  @JavascriptInterface  
  public void showJsText(String text){  
      webView.loadUrl("javascript:jsText('"+text+"')");  
  }  /** 
   * 注意这里的@JavascriptInterface注解， target是4.2以上都需要添加这个注解，否则无法调用 
   * @param text 
   */  
  @JavascriptInterface  
  public void showToast(String text){  
      Toast.makeText(getApplicationContext(), text, Toast.LENGTH_SHORT).show();  
  }  
  @JavascriptInterface  
  public void showJsText(String text){  
      webView.loadUrl("javascript:jsText('"+text+"')");  
  }  
}
js部分
window.jsi.showToast('测试js调用系统api');
window.jsi.showJsText('测试java回调js');
```
这种方法具有兼容性和安全性问题，但也是一种实现jsbridge的思路，比较灵活。
### 通过schemeurl来进行调用
1. 主要是通过传一个url来给native端
2. 这个url里面需要带有需要调用的native的类名以及方法名，还有调用方法传入的参数以及方法调用之后需要回调的js方法名。
3. 在native端捕获到这个url之后，分析里面的各个结构，知道调用native的哪个方法，以及之后的回调。
4. 调用native对应的方法。然后调用回调的js。

具体怎么捕获url，我看到两种方式
### 隐藏iframe法。
通过一个隐藏的iframe，设置他的src为对应的url
```
var messagingIframe = document.createElement('iframe');
messagingIframe.style.display = 'none';
document.documentElement.appendChild(messagingIframe);
//触发scheme
messagingIframe.src = uri;
```
在安卓端，WebViewClient中有一个shouldOverrideUrlLoading方法，可以拦截到url，之后对其做相应的处理
```
public boolean shouldOverrideUrlLoading(WebView view, String url){
	//读取到url后自行进行分析处理
	
	//如果返回false，则WebView处理链接url，如果返回true，代表WebView根据程序来执行url
	return true;
}
```
### window.prompt法，讲一下实现，与上面的方法大同小异
window.prompt传递url
```
window.prompt(url);
```
WebChromeClient中有个onJsPrompt方法可以拦截prompt方法，获得url参数，之后做相应的处理
```
public class JSBridgeWebChromeClient extends WebChromeClient {
    @Override
    public boolean onJsPrompt(WebView view, String url, String message, String defaultValue, JsPromptResult result) {
        //此处的JSBridge.callJava调用native方法，传入的message就是url
        result.confirm(JSBridge.callJava(view, message));
        return true;
    }
}
```
看看MainActivity里面怎么写
```
WebView mWebView = (WebView) findViewById(R.id.webview);
WebSettings settings = mWebView.getSettings();
settings.setJavaScriptEnabled(true);
//给webviewsetClient，他就可以拦截prompt了
mWebView.setWebChromeClient(new JSBridgeWebChromeClient());
mWebView.loadUrl("file:///android_asset/index.html");
```
来看calljava方法
```
public static String callJava(WebView webView, String uriString) {
    String methodName = "";
    String className = "";
    String param = "{}";
    String port = "";
    if (!TextUtils.isEmpty(uriString) && uriString.startsWith("JSBridge")) {
        Uri uri = Uri.parse(uriString);
        className = uri.getHost();
        param = uri.getQuery();
        port = uri.getPort() + "";
        String path = uri.getPath();
        if (!TextUtils.isEmpty(path)) {
            methodName = path.replace("/", "");
        }
    }

    //exposedMethods这个hashmap存放了对外暴露的native方法，这样只暴露部分方法
    if (exposedMethods.containsKey(className)) {
        HashMap<String, Method> methodHashMap = exposedMethods.get(className);

        if (methodHashMap != null && methodHashMap.size() != 0 && methodHashMap.containsKey(methodName)) {
            Method method = methodHashMap.get(methodName);
            if (method != null) {
                try {
                    //获得method对象后，利用它的invoke方法，传入参数param，以及回调Callback
                    method.invoke(null, webView, new JSONObject(param), new Callback(webView, port));
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }
    return null;
}
```
Callback类用于回调js
```
public class Callback {
    private static Handler mHandler = new Handler(Looper.getMainLooper());
    //回调会调用JSBridge.onFinish方法，传入回调的port以及json格式的参数。
    private static final String CALLBACK_JS_FORMAT = "javascript:JSBridge.onFinish('%s', %s);";
    private String mPort;
    private WeakReference<WebView> mWebViewRef;

    public Callback(WebView view, String port) {
        mWebViewRef = new WeakReference<>(view);
        mPort = port;
    }

    public void apply(JSONObject jsonObject) {
        final String execJs = String.format(CALLBACK_JS_FORMAT, mPort, String.valueOf(jsonObject));
        if (mWebViewRef != null && mWebViewRef.get() != null) {
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    //此处利用webView.loadUrl来执行回调
                    mWebViewRef.get().loadUrl(execJs);
                }
            });

        }

    }
}
```
来看看jsBridge的注册,其实就是一个二维的hashmap
```
public class JSBridge {
    private static Map<String, HashMap<String, Method>> exposedMethods = new HashMap<>();

    public static void register(String exposedName, Class<? extends IBridge> clazz) {
        if (!exposedMethods.containsKey(exposedName)) {
            try {
                exposedMethods.put(exposedName, getAllMethod(clazz));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private static HashMap<String, Method> getAllMethod(Class injectedCls) throws Exception {
        HashMap<String, Method> mMethodsMap = new HashMap<>();
        Method[] methods = injectedCls.getDeclaredMethods();
        for (Method method : methods) {
            String name;
            if (method.getModifiers() != (Modifier.PUBLIC | Modifier.STATIC) || (name = method.getName()) == null) {
                continue;
            }
            Class[] parameters = method.getParameterTypes();
            if (null != parameters && parameters.length == 3) {
                if (parameters[0] == WebView.class && parameters[1] == JSONObject.class && parameters[2] == Callback.class) {
                    mMethodsMap.put(name, method);
                }
            }
        }
        return mMethodsMap;
    }
}
```
最后看看js端的jsbridge，主要就是拼接url，然后传给webview，onFinish方法用于最后的回调。
```
function (win) {
    var hasOwnProperty = Object.prototype.hasOwnProperty;
    var JSBridge = win.JSBridge || (win.JSBridge = {});
    var JSBRIDGE_PROTOCOL = 'JSBridge';
    var Inner = {
        callbacks: {},
        call: function (obj, method, params, callback) {
            console.log(obj+" "+method+" "+params+" "+callback);
            var port = Util.getPort();
            console.log(port);
            this.callbacks[port] = callback;
            var uri=Util.getUri(obj,method,params,port);
            console.log(uri);
            window.prompt(uri, "");
        },
        onFinish: function (port, jsonObj){
            var callback = this.callbacks[port];
            callback && callback(jsonObj);
            //回调执行完后被回调函数被释放了，如果没有执行，这段会内存泄漏
            delete this.callbacks[port];
        },
    };
    var Util = {
        getPort: function () {
            return Math.floor(Math.random() * (1 << 30));
        },
        getUri:function(obj, method, params, port){
            params = this.getParam(params);
            var uri = JSBRIDGE_PROTOCOL + '://' + obj + ':' + port + '/' + method + '?' + params;
            return uri;
        },
        getParam:function(obj){
            if (obj && typeof obj === 'object') {
                return JSON.stringify(obj);
            } else {
                return obj || '';
            }
        }
    };
    //这个地方是一个mixin
    for (var key in Inner) {
        if (!hasOwnProperty.call(JSBridge, key)) {
            JSBridge[key] = Inner[key];
        }
    }
})(window);
```
之前的native端的方法调用是用反射来做的，这里看看native端的方法具体长啥样
```
//这个接口是一个空接口，仅仅起到约束作用
public class BridgeImpl implements IBridge {
    public static void showToast(WebView webView, JSONObject param, final Callback callback) {
        String message = param.optString("msg");
        Toast.makeText(webView.getContext(), message, Toast.LENGTH_SHORT).show();
        if (null != callback) {
            try {
                JSONObject object = new JSONObject();
                object.put("key", "value");
                object.put("key1", "value1");
                //执行回调类的apply
                callback.apply(getJSONObject(0, "ok", object));
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private static JSONObject getJSONObject(int code, String msg, JSONObject result) {
        JSONObject object = new JSONObject();
        try {
            object.put("code", code);
            object.put("msg", msg);
            object.putOpt("result", result);
            return object;
        } catch (JSONException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```
具体的调用,这么写就可以了。
```
JSBridge.call('bridge','showToast',{'msg':'Hello JSBridge'},function(res){alert(JSON.stringify(res))})
```
# IOS
## js端
具体做法大同小异，要么直接把原生接口映射到js的一个对象中，要么使用shcemeurl，ios在IOS7以后可以使用jscore来实现第一种做法，之前只能使用schemeurl。
另外IOS端发送请求
1. 要么是使用隐藏iframe，用UIWebViewDelegate这个方法来捕获请求
2. 要么可以使用xhr，用NSURLProtocol来捕获请求

## native端
原生支持使用(NSString *)stringByEvaluatingJavaScriptFromString:(NSString *)script;执行js，

# 总结
JSBridge的基本原理为：
H5->通过某种方式触发一个url->Native捕获到url,进行分析->原生做处理->Native调用H5的JSBridge对象传递回调。
1. Android4.2以下，addJavascriptInterface方法有安全漏洞，js代码可以获取到Java层的运行时对象，来伪造当前用户执行恶意代码。
2. ios7以下，JavaScript无法调用native代码。
3. 通过js声明的对象，是通过loadUrl注入到页面中的，所以这个对象是js对象，而不是Java对象，没有getClass等Object方法，因此也无法获得Runtime对象，避免了恶意代码的注入。
4. JSBridge采用URL解析的交互方式，是一套成熟的解决方案，便于拓展，无重大安全性问题。
5. 从上面的分析可以发现，从js调用native，2种方式都必定是异步的。而从native回到js，却是一个同步的方法，而且是跑在主线程里
6. 调用cordova插件的代码，对返回值的处理一定要放在回调函数里，因为结果是异步返回的。同时，回调函数的执行时间不能太长，否则会阻塞native主线程

# 附言
webViewClient与webChromeClient分别用来管理webView中不同的事件
**webViewClient**
```
 /**
   在开始加载网页时会回调
  */
 public void onPageStarted(WebView view, String url, Bitmap favicon) 
 /**
   在结束加载网页时会回调
  */
 public void onPageFinished(WebView view, String url)
 /**
   拦截 url 跳转,在里边添加点击链接跳转或者操作
  */
 public boolean shouldOverrideUrlLoading(WebView view, String url)
 /**
   加载错误的时候会回调，在其中可做错误处理，比如再请求加载一次，或者提示404的错误页面
  */
 public void onReceivedError(WebView view, int errorCode,String description, String failingUrl)
 /**
   当接收到https错误时，会回调此函数，在其中可以做错误处理
  */
 public void onReceivedSslError(WebView view, SslErrorHandler handler,SslError error)
 /**
   在每一次请求资源时，都会通过这个函数来回调
  */
 public WebResourceResponse shouldInterceptRequest(WebView view, String url) {
     return null;
 }
```
**webChromeClient**
```
 /**
  当网页调用alert()来弹出alert弹出框前回调，用以拦截alert()函数
 */
 public boolean onJsAlert(WebView view, String url, String message,JsResult result)
 /**
   当网页调用confirm()来弹出confirm弹出框前回调，用以拦截confirm()函数
  */
 public boolean onJsConfirm(WebView view, String url, String message,JsResult result)
 /**
   当网页调用prompt()来弹出prompt弹出框前回调，用以拦截prompt()函数
  */
  public boolean onJsPrompt(WebView view, String url, String message,String defaultValue, JsPromptResult result) 
  /**
   打印 console 信息
  */
  public boolean onConsoleMessage(ConsoleMessage consoleMessage)
  /**
   通知程序当前页面加载进度
  */
  public void onProgressChanged(WebView view, int newProgress)
```
# 参考文献：
[安卓与JS互调之android webview addJavascriptInterface](https://blog.csdn.net/lumingwei2411/article/details/62898980)
[WebView使用解析（二）之WebViewClient/WebChromeClient](https://blog.csdn.net/huaxun66/article/details/73252592)
[Hybrid APP基础篇(四)->JSBridge的原理](http://www.cnblogs.com/dailc/p/5931324.html)
[Android JSBridge的原理与实现](https://blog.csdn.net/sbsujjbcy/article/details/50752595)
[Android中JSBridge的原理与实现](https://www.jianshu.com/p/2ec3f06d6087)
[cordova与ios native code交互的原理](https://blog.csdn.net/kyfxbl/article/details/38404471)
[深入浅出 JavaScriptCore](http://www.cocoachina.com/ios/20170720/19958.html)
[浅析 Cordova for iOS](http://zhenby.com/blog/2013/05/16/cordova-for-ios/)
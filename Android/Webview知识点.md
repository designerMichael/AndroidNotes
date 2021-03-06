---
title: Webview知识点
date: 2016-03-09 13:23:30
categories: Android 
tags: 网络
---

Android系统中内置了一款高性能 webkit 内核浏览器,在 SDK 中封装为一个叫做 WebView 组件。


WebView常用的功能基本上都离不开这三个类：WebSetting、WebViewClient、WebChromeClient，当然了如果想操作Cookie还要借助于另外两个操作类CookieSyncManager（API21标记为deprecated）和CookieManager。



<!--more-->
# 基本使用

1.权限：
```java
<uses-permission android:name="android.permission.INTERNET" />
```

2.添加webview到应用中，直接将webview添加到layout文件中

```java
<?xml version="1.0" encoding="utf-8"?>
<WebView  xmlns:android="http://schemas.android.com/apk/res/android"    
	android:id="@+id/webview"   
	android:layout_width="match_parent"    
	android:layout_height="match_parent"/>
```


3.使用webview加载url使用loadUrl方法：
```java
WebView myWebView = (WebView) findViewById(R.id.webview);
myWebView.loadUrl("http://www.example.com");
```

4.如果页面中链接,如果希望点击链接继续在当前browser中响应,而不是新开Android的系统browser中响应该链接,必须覆盖 WebView的WebViewClient对象。

```java
mWebView.setWebViewClient(new WebViewClient(){
    public boolean shouldOverrideUrlLoading(WebView view, String url){ 
        view.loadUrl(url);
        return true;
    }
});
```
如果想用系统游览器打开链接：
```java
public boolean shouldOverrideUrlLoading(WebView view, String url) {
	Intent intent = new Intent(Intent.ACTION_VIEW);
	intent.setData(Uri.parse(url));
	startActivity(intent);
	return true;
}
```
shouldOverrideUrlLoading()方法返回的是一个boolean类型，如果返回false，则WebView处理链接url，如果返回true，WebView根据程序来执行url。一般都返回true


5.如果不做任何处理 ,浏览网页,点击系统“Back”键,整个 Browser 会调用 finish()而结束自身,如果希望浏览的网页回退而不是推出浏览器,需要在当前Activity中处理并消费掉该 Back 事件。

```java
public boolean onKeyDown(int keyCode, KeyEvent event) {
    if ((keyCode == KEYCODE_BACK) && mWebView.canGoBack()) { 
        mWebView.goBack();
        return true;
    }
    return super.onKeyDown(keyCode, event);
}
```

6.前进：
```java
//判断是否可以前进，可以返回true
webView.canGoForward();
//进入上一层级
webView.goForward();
```

7.刷新：
```java
webView.reload();
```

# 一些设置

```java
mWebView.getSettings().setJavaScriptEnabled(false);//JS
mWebView.getSettings().setSupportZoom(false);//能否缩放
mWebView.getSettings().setBuiltInZoomControls(false);//是否显示缩放工具
mWebView.getSettings().setDefaultFontSize(18);//字体大小，默认为16，有效值区间在1-72之间
mWebSettings.setAllowFileAccess(true);//是否允许访问文件数据
mWebSettings.setSavePassword(true); //是否保存密码
setDefaultTextEncodingName(encoding);//设置网页默认编码

WebView.getSettings().setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);//优先使用缓存
WebView.getSettings().setCacheMode(WebSettings.LOAD_NO_CACHE);//不缓存

.setScrollBarStyle(SCROLLBARS_OUTSIDE_OVERLAY);//取消滚动条
```

# 手势缩放
如果想要WebView加载进的网页而支持缩放，只需要同时设置下面两个属性就可以了。
```java
settings.setBuiltInZoomControls(true);
settings.setSupportZoom(true);
```

但是并不是设置了上面两个属性就可以了，这还要看网页自身是否支持缩放了，如果网页设置了如下meta：

<meta name=”viewport” content=”width=device-width,initial-scale=1.0,user-scalable=no, minimum-scale=1.0, maximum-scale=1.0″>

这种情况下我们是不能手动缩放网页的，因为网页自身已经设置很明确了，用户不可以手动缩放页面，页面宽度等于屏幕宽度，最大缩放比例和最小缩放比例都是1.0倍。很多自适应网页都会设置该属性，如果加载进来的网页已经设置了自动缩放，但是却不可以缩放，不要太奇怪了。


# 加载方式


## 加载assets目录下的本地网页
一般我们都是把html文件放在assets目录下， WebView调用assets目录下的本地网页和图片等资源非常方便
```java
mWebView.loadUrl("file:///android_asset/html/test1.html");
```

## 加载网络网页
```java
mWebView.loadUrl("http://www.google.com");
```

## 加载内容
有时候我们的webview可能只是html片段，而不是一个完整的网页，事实上绝大多数时候都是如此，完整的网页无需做成应用，而直接在浏览器访问。  
这种情况我们使用 LoadData 或者 loadDataWithBaseURL方法。


先看前者：
```java
//data 为 html 代码内容，mimeType一般为"text/html"，utf-8
loadData(String data, String mimeType, String encoding)
```
如果显示为乱码，则先把要加载的字符串编码一下就行了。
```java
data = new String(data.getBytes(), "utf-8");
```


再看loadDataWithBaseURL方法
```java
loadDataWithBaseURL (String baseUrl, String data, String mimeType, String encoding, String historyUrl)
```

这个比loadData()多两个参数，可以指定HTML代码片段中相关资源的相对根路径，也可以指定历史Url，其余三个参数相同。

这里主要注意参数baseUrl，baseUrl指定了你的data参数中数据是以什么地址为基准的，因为data中的数据可能会有超链接或者是image元素，而很多网站的地址都是用的相对路径，如果没有baseUrl，webview将访问不到这些资源。当然如果全部使用绝对路径就不会有问题。。


loadDataWithBaseURL和loadData两个方法加载的HTML代码片段的不同点在于，loadData()中的html data中不能包含'#', '%', '\', '?'四中特殊字符。  
%，会报找不到页面错误，页面全是乱码。  #，会让你的goBack失效，但canGoBAck是可以使用的。于是就会产生返回按钮生效，但不能返回的情况。


下面来个栗子(加载assets下的html片段)：
```java
String data = "";
try {
    // 读取assets目录下的文件需要用到AssetManager对象的Open方法打开文件
    InputStream is = getAssets().open("html/test2.html");
    // loadData()方法需要的是一个字符串数据所以我们需要把文件转成字符串
    ByteArrayBuffer baf = new ByteArrayBuffer(500);
    int count = 0;
    while ((count = is.read()) != -1) {
        baf.append(count);
    }
    data = EncodingUtils.getString(baf.toByteArray(), "utf-8");
} catch (IOException e) {
    e.printStackTrace();
}
// 下面两种方法都可以加载成功
mWebView.loadData(data, "text/html", "utf-8");

// wv.loadDataWithBaseURL("", data, "text/html", "utf-8", "");
```



# 加载过程控制（WebViewClient）


可能会想要在加载过程中显示进度条，加载完后隐藏进度条

```java
mWebView.setWebViewClient(new WebViewClient() {
    @Override
    public void onPageStarted(WebView view, String url, Bitmap favicon) {
        super.onPageStarted(view, url, favicon);
        mProgressBar.setVisibility(View.VISIBLE);

    }

    @Override
    public void onPageFinished(WebView view, String url) {
        super.onPageFinished(view, url);
        mProgressBar.setVisibility(View.INVISIBLE);

    }
});

```

另外还有其他一些可重写的方法 ：

- 接收到Http请求的事件  
onReceivedHttpAuthRequest(WebView view, HttpAuthHandler handler, String host, String realm) 

- 打开链接前的事件  
public boolean shouldOverrideUrlLoading(WebView view, String url) { view.loadUrl(url); return true; }   
这个函数我们可以做很多操作，比如我们读取到某些特殊的URL，于是就可以不打开地址，取消这个操作，进行预先定义的其他操作，这对一个程序是非常必要的。
 
- 载入页面完成的事件  
public void onPageFinished(WebView view, String url){ }   
同样道理，我们知道一个页面载入完成，于是我们可以关闭loading条，切换程序动作。
 
- 载入页面开始的事件  
public void onPageStarted(WebView view, String url, Bitmap favicon) { }   
这个事件就是开始载入页面调用的，通常我们可以在这设定一个loading的页面，告诉用户程序在等待网络响应。

- 加载错误  
```java
/**
 * api23过时
 */
public void onReceivedError(WebView view, int errorCode, String description, String failingUrl) {
    super.onReceivedError(view, errorCode, description, failingUrl);
    ...
}
 
/**
 * api23新增
 */
public void onReceivedError(WebView view, WebResourceRequest request, WebResourceError error) {
    super.onReceivedError(view, request, error);
    ...
}

```
onReceivedError方法有两个，第二个是在Android6.0后新增加的，如果是在高版本上开发的，很可能我们复写的是第二个方法，
在低版本上测试时如果出现错误并不会进入onReceivedError方法，所以在开发中一定要注意使用的方法。



# 支持javascript

如果访问的页面中有 Javascript,则 WebView 必须设置支持 Javascript。
```java
WebView.getSettings().setJavaScriptEnabled(true);
```

通过创建JavaScript的接口可以让JavaScript和安卓客户端进行交互。比如说JavaScript可以调用本地的安卓代码展示一个Dialog而不是使用JavaScript的alter()方法。通过addJavascriptInterface()方法创建接口。

java中调用JS代码：WebView.loadUrl("javascript:方法名()");
JS中调用java代码：window.对象名.方法名（window可以省略）

栗子：  

实现功能：利用 android 中的 WebView 加载一个 html 网页,在 html 网页中定义一个按钮,点击按钮弹出一 个 toast。

先定义一个“接口”类，将上下文对象传进去,在接口类中定义要在 js 中实现的方法。 
```java
public class WebAppInterface {
    Context mContext;

    /** Instantiate the interface and set the context */
    WebAppInterface(Context c) {
        mContext = c;
    }

    /** Show a toast from the web page */
    @JavascriptInterface
    public void showToast(String toast) {
        Toast.makeText(mContext, toast, Toast.LENGTH_SHORT).show();
    }
}
```
接着在assets资源包下定义一个 html 文件,在文件中定义一个 button。button 的点击事件定义为一个 js 函数。

在addJavascriptInterface方法中创建接口
```java
WebView webView = (WebView) findViewById(R.id.webview);
myWebView.getSettings().setJavaScriptEnabled(true);
webView.addJavascriptInterface(new WebAppInterface(this), "Android");
```

在HTML的JavaScript中调用WebView显示toast消息:
```java
	<inputt type="button" value="Say hello" onClick="showAndroidToast('Hello Android!')" />
	
	<script type="text/javascript">
	    function showAndroidToast(toast) {
	        Android.showToast(toast);
	    }
	</script>
```
> 注意：  
1 addJavascriptInterface方法中要绑定的Java对象及方法运行在另外的线程中,而不是运行在构造他的线程中。  
2 使用addJavascriptInterface将会使javaScript可以操控安卓本地代码。因此有可能导致安全性问题，请确保所有HTML代码都是你所知道的。  
3 如果targetSdkVersion是17或者更高（android 4.2以上），必须使用 @JavascriptInterface 注解，才能让网页访问到本地代码。




再一个栗子：
```java
public class WebViewDemo extends Activity { 
    private WebView mWebView;
    private Handler mHandler = new Handler(); 

    public void onCreate(Bundle icicle) { 

    setContentView(R.layout.WebViewdemo);
    mWebView = (WebView) findViewById(R.id.WebView); 
    WebSettings webSettings = mWebView.getSettings(); 
    webSettings.setJavaScriptEnabled(true); 
    mWebView.addJavascriptInterface(new Object() {
      public void clickOnAndroid() {
          mHandler.post(new Runnable() {
              public void run() { 
                  mWebView.loadUrl("javascript:wave()");
              }
          });
      }
    }, "demo"); 
    mWebView.loadUrl("file:///android_asset/demo.html"); 

    }
}
```
我们看 addJavascriptInterface(Object obj,String interfaceName)这个方法 ,该方法将一个java对象绑定到一个javascript对象中,javascript对象名就是 interfaceName(demo),作用域是Global.这样初始化 WebView 后,在WebView加载的页面中就可以直接通过javascript:window.demo访问到绑定的java对象了. 来看看在html中是怎样调用的。
```java
<html>
<script language="javascript">
  function wave() {
    document.getElementById("droid").src="android_waving.png";
  }
</script>
<body>
  <a onClick="window.demo.clickOnAndroid()">
  <img id="droid" src="android_normal.png" mce_src="android_normal.png"/><br> Click me! </a>
</body>
</html>
```

这样在 javascript 中就可以调用 java 对象的 clickOnAndroid()方法了,同样我们可以在此对象中定义很多方法(比如发短信,调用联系人列表等手机系统功能),这里 wave()方法是 java 中调用 javascript 的例子.
需要说明一点:addJavascriptInterface方法中要绑定的Java对象及方法要运行另外的线程中,不能运行在构造他的线程中,这也是使用 Handler 的目的.



# 缓存

## 缓存位置
在项目中如果使用到 WebView 控件,当加载 html 页面时,会在/data/data/包名目录下生成 database 与 cache 两个文件夹。
请求的 url 记录是保存在 WebViewCache.db,而 url 的内容是保存在 WebViewCache 文件夹下. 大家可以自己动手试一下,定义一个html文件,在里面显示一张图片,用WebView加载出来,然后再试着从缓存里把这张图片读取出来并显示 。

## 删除缓存
其实已经知道缓存保存的位置了，那么删除就很简单了，获取到这个缓存，然后删掉他就好了。
```java
//删除保存于手机上的缓存
private int clearCacheFolder(File dir,long numDays) { 
  int deletedFiles = 0;
  if (dir!= null && dir.isDirectory()){
    try {
      for (File child:dir.listFiles()){
          if (child.isDirectory()) {
            deletedFiles += clearCacheFolder(child, numDays);
        }
        if (child.lastModified() < numDays) {
          if (child.delete()) {
           deletedFiles++; 
          }
        }
      }
    } catch(Exception e) {
      e.printStackTrace(); 
    }
  }
  return deletedFiles; 
}
```

## 清空缓存

```java
File file = CacheManager.getCacheFileBaseDir();
    if (file != null && file.exists() && file.isDirectory()) {
        for (File item : file.listFiles()) {
            item.delete();
        }
        file.delete();
    }
    context.deleteDatabase("WebView.db"); 
    context.deleteDatabase("WebViewCache.db");
```

## 是否启用缓存功能

```java
	//优先使用缓存: 
    WebView.getSettings().setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK); 
    //不使用缓存: 
    WebView.getSettings().setCacheMode(WebSettings.LOAD_NO_CACHE);
```

更多缓存芝士看
<http://blog.csdn.net/moubenmao_jun/article/details/9076917>

#  处理 404 错误

显示网页还会遇到一个问题，就是网页有可能会找不到，WebView当然也是可以处理的
```java
public class WebView_404 extends Activity { 
    private Handler handler = new Handler() {
    public void handleMessage(Message msg) {
      if(msg.what==404) {//主页不存在
        //载入本地 assets 文件夹下面的错误提示页面 404.html 
        web.loadUrl("file:///android_asset/404.html");
      }else{
        web.loadUrl(HOMEPAGE);
      }
    }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      web.setWebViewClient(new WebViewClient() {
    public boolean shouldOverrideUrl(WebView view,String url) { 
        if(url.startsWith("http://") && getRespStatus(url)==404) {
          view.stopLoading();
          //载入本地 assets 文件夹下面的错误提示页面 404.html 
          view.loadUrl("file:///android_asset/404.html");
        }else{
          view.loadUrl(url);
        }
          return true;
        }
      });
      new Thread(new Runnable() {
    public void run() {
          Message msg = new Message();
          //此处判断主页是否存在,因为主页是通过 loadUrl 加载的,
          //此时不会执行 shouldOverrideUrlLoading 进行页面是否存在的判断 //进入主页后,点主页里面的链接,链接到其他页面就一定会执行
          shouldOverrideUrlLoading 方法了 
          if(getRespStatus(HOMEPAGE)==404) {
          msg.what = 404;
          }
          handler.sendMessage(msg);
      }).start();
  }
}
```

# 判断 WebView 是否已经滚动到页面底端

在View中有一个getScrollY()方法，可以返回当前可见区域的顶端距整个页面顶端的距离,也就是当前内容滚动的距离。  
还有getHeight()或者 getBottom()方法都返回当前 View 这个容器的高度   
在webView中还有getContentHeight() 方法可以返回整个 html 页面的高度,但并不等同于当前整个页面的高度 ,因为 WebView 有缩放功能。你可以通过如下代码来启动或关闭webview的缩放功能。
```java
mWebView.getSettings().setSupportZoom(true);
mWebView.getSettings().setBuiltInZoomControls(true);
```

所以当前整个页面的高度实际上应该是原始 html 的高度再乘上缩放比例. 因此,更正后的结果 ,准确的判断是否处于底部的方法应该是:

```java
// 如果已经处于底端
if(WebView.getContentHeight*WebView.getScale() -(webvi ew.getHeight()+WebView.getScrollY())){ 
  //XXX
}
```

# WebView清除本地cookies

首先，要清除肯定要会添加，这里给大家提供一个工具方法：
```java
	/***
     * 如果用户已经登录，则同步本地的cookie到webview中
     */
    public void synCookies() {
        if (!CacheUtils.isLogin(this)) return;
        CookieSyncManager.createInstance(this);
        CookieManager cookieManager = CookieManager.getInstance();
        cookieManager.setAcceptCookie(true);
        cookieManager.removeSessionCookie();//移除
        String cookies = PreferenceHelper.readString(this, AppConfig.COOKIE_KEY, AppConfig.COOKIE_KEY);
        KJLoger.debug(cookies);
        cookieManager.setCookie(url, cookies);
        CookieSyncManager.getInstance().sync();
    }
```
在使用网页版淘宝或百度登录时,WebView会自动登录上次的帐号!(因为WebView 记录了帐号和密码的cookies) 所以,需要清除 SessionCookie也是有必要的。 
那么CookieManager同样也为我们提供了清除cookie的方法  
```java
CookieManager.getInstance().removeSessionCookie();
```
这里顺便说一下WebView本身也是会记录html缓存的，webview本身就提供了清理缓存的方法，其中参数true是指是否包括磁盘文件也一并清除：  
```java
webview.clearCache(true);
webview.clearHistory();
```


# WebView同步Cookie免登录


没接触过，以后再说。<http://www.sunnyang.com/490.html>文章最后有提及


# WebView获取服务器中的 session 问题

接下来我们讲如下两个问题：
Android 中的 WebView 如何获取服务器页面的 jsessionid 的值
Android 的 WebView 又是如何把得到的 jsessionid 的值在 set 到服务器中,一致达到他们在同一个 jsessionid 的回话中.
其实非常非常简单，只不过是几个方法罢了:
```java
CookieManager cm = CookieManager.getInstance(); 
cm.removeAllCookie();
cm.getCookie(url);
cm.setCookie(url, cookie);
```

另外还有个 CookieSyncManager,也许你会在一些旧的项目中看到它。从名字来理解，它实际上应该是一个异步缓存器。不过我们看到这个方法已经被标记为过时了，查看源码可以看到过时原因是现在WebView已经是自动的异步缓存了，所以这个类已经没有存在的意义了。

# android 中webView控件 padding不起作用

在一个布局文件中有一个WebView，想使用padding属性让左右向内留出一些空白，但是padding属性不起左右，内容照样贴边显示，反而移动了右边滚动条的位置。

正确的做法是在webView的加载的css中增加padding,没必要为了padding而更改xml布局文件。

# WebView、WebViewClient、WebChromeClient 之间的区别: 
在 WebView 的设计中,不是什么事都要 WebView类干的,有些杂事是分给其他人的,这样 WebView 专心干好 自己的解析、渲染工作就行了.  
WebViewClient 就是帮助 WebView 处理各种通知、请求事件,加载过程控制等   
WebChromeClient 是辅助 WebView 处理 Javascript 的对话框,网站图标,网站 title.


# webview的两种使用场景

WebView在实际使用中可以分为两种使用方法，第一种就是类似于QQ微信那种，使用loadUrl直接去显示一个链接，这种方式太简单了，传一个url就行，我就不多说了。

那么需要详细讲的是第二种，类似的实现大家可以看看开源中国客户端，网易新闻客户端，爱看博客，等客户端的实现方式，它们实际上也是通过webview来显示的一个网页内容，但是并不是单纯的loadurl,而是以字符串的形式去加载一个已经获取到了的html源代码。这样做的好处在于显示的页面可以完全的根据自己喜好来定义，比如我想在末尾添加一张图片，那么简单，在这个html字符串的末尾插入一个img标签就可了。
```java
mWebView.loadData(htmlText,”text/html”, “utf-8”);
```

两种方法的适用场景，前一种载入链接的方法适合一个界面(Activity或Fragment)只有一个WebView或者说WebView占很大一块的时候，同时我们要显示的内容是未知的，那么自然是使用loadurl方法更合适，例如QQ聊天的时候对方发送一条链接，当QQ解析出这个文本是一个网址时就通过webview去加载它。

而后一种则适合于定制化内容，一般是那种你可以明确的制度网页内容以及要显示的内容时使用，至于好处就是上面说的，定制性要好很多。


# 参考链接

<http://www.36nu.com/post/154.html>  
<http://kymjs.com/code/2015/05/04/01/>  
<http://www.jianshu.com/p/e3965d3636e7>
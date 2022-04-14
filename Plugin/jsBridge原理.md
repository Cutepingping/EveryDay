## JSBridgen做了些什么
在Hybrid模式下，H5会经常需要使用Native的功能，比如打开二维码扫描、调用原生页面、获取用户信息等，同时Native也需要向Web端发送推送、更新状态等，而JavaScript是运行在单独的JS Context中（Webview容器、JSCore等），与原生有运行环境的隔离，所以需要有一种机制实现Native端和Web端的双向通信，这就是JSBridge：以JavaScript引擎或Webview容器作为媒介，通过协定协议进行通信，实现Native端和Web端双向通信的一种机制。

通过JSBridge，Web端可以调用Native端的Java接口，同样Native端也可以通过JSBridge调用Web端的JavaScript接口，实现彼此的双向调用

## webview
首先了解下webView，webView是移动端提供的运行JavaScript的环境，是系统渲染Web网页的一个控件，可与页面JavaScript交互，实现混合开发，其中Android和iOS又有些不同：

Android的WebView采用的是低版本和高版本使用了不同的webkit内核，4.4后直接使用了Chrome。

iOS中UIWebView算是自IOS2就有，但性能较差，特性支持较差，WKWebView是iOS8之后的升级版，性能更强特性支持也较好。

WebView控件除了能加载指定的url外，还可以对URL请求、JavaScript的对话框、加载进度、页面交互进行强大的处理，之后会提到拦截请求、执行JS脚本都依赖于此。

## jsBridge实现原理
Web端和Native可以类比于Client/Server模式，Web端调用原生接口时就如同Client向Server端发送一个请求类似，JSB在此充当类似于HTTP协议的角色，实现JSBridge主要是两点：
- 将Native端原生接口封装成JavaScript接口
- 将Web端JavaScript接口封装成原生接口

### Native->Web
首先来说Native端调用Web端，这个比较简单，JavaScript作为解释性语言，最大的一个特性就是可以随时随地地通过解释器执行一段JS代码，所以可以将拼接的JavaScript代码字符串，传入JS解析器执行就可以，JS解析器在这里就是webView。

**Android**
Android 4.4之前只能用loadUrl来实现，并且无法执行回调：
```javascript
String jsCode = String.format("window.showWebDialog('%s')", text);
webView.loadUrl("javascript: " + jsCode);
```

Android 4.4之后提供了evaluateJavascript来执行JS代码，并且可以获取返回值执行回调：
```javascript
String jsCode = String.format("window.showWebDialog('%s')", text);
webView.evaluateJavascript(jsCode, new ValueCallback<String>() {
  @Override
  public void onReceiveValue(String value) {

  }
});
```

**iOS**
iOS的UIWebView
使用stringByEvaluatingJavaScriptFromString：
```javascript
NSString *jsStr = @"执行的JS代码";
[webView stringByEvaluatingJavaScriptFromString:jsStr];
```

iOS的WKWebView使用evaluateJavaScript：
```javascript
[webView evaluateJavaScript:@"执行的JS代码" completionHandler:^(id _Nullable response, NSError * _Nullable error) {

}];
```

### Web->Native
Web调用Native端主要有两种方式

**拦截Webview请求的URL Schema**
URL Schema是类URL的一种请求格式，格式如下：
`<protocol>://<host>/<path>?<qeury>#fragment`

我们可以自定义JSBridge通信的URL Schema，比如：`jsbridge://showToast?text=hello`

Native加载WebView之后，Web发送的所有请求都会经过WebView组件，所以Native可以重写WebView里的方法，从来拦截Web发起的请求，我们对请求的格式进行判断：
- 如果符合我们自定义的URL Schema，对URL进行解析，拿到相关操作、操作，进而调用原生Native的方法
- 如果不符合我们自定义的URL Schema，我们直接转发，请求真正的服务

Web发送URL请求使用`iframe.src`，a标签需要用户操作，location.href可能会引起页面的跳转丢失调用，发送ajax请求Android没有相应的拦截方法，所以使用iframe.src是经常会使用的方案
- 安卓提供了shouldOverrideUrlLoading方法拦截
- UIWebView使用shouldStartLoadWithRequest，WKWebView则使用decidePolicyForNavigationAction

这种方式从早期就存在，兼容性很好，但是由于是基于URL的方式，长度受到限制而且不太直观，数据格式有限制，而且建立请求有时间耗时

**向Webview中注入JS API**
这个方法会通过webView提供的接口，App将Native的相关接口注入到JS的Context（window）的对象中，一般来说这个对象内的方法名与Native相关方法名是相同的，Web端就可以直接在全局window下使用这个暴露的全局JS对象，进而调用原生端的方法。

这个过程会更加简单直观，不过有兼容性问题，大多数情况下都会使用这种方式

Android（4.2+）提供了addJavascriptInterface注入
```java
// 注入全局JS对象
webView.addJavascriptInterface(new NativeBridge(this), "NativeBridge");

class NativeBridge {
  private Context ctx;
  NativeBridge(Context ctx) {
    this.ctx = ctx;
  }

  // 增加JS调用接口
  @JavascriptInterface
  public void showNativeDialog(String text) {
    new AlertDialog.Builder(ctx).setMessage(text).create().show();
  }
}
```
在Web端直接调用这个方法即可：
```javascript
window.NativeBridge.showNativeDialog('hello');
```

iOS的UIWebView提供了JavaSciptCore
iOS的WKWebView提供了WKScriptMessageHandler

## 带回调的调用
上面已经说到了Native、Web间双向通信的两种方法，但站在一端而言还是一个单向通信的过程 ，比如站在Web的角度：Web调用Native的方法，Native直接相关操作但无法将结果返回给Web，但实际使用中会经常需要将操作的结果返回，也就是JS回调。

其实基于之前的单向通信就可以实现，我们在一端调用的时候在参数中加一个callbackId标记对应的回调，对端接收到调用请求后，进行实际操作，如果带有callbackId，对端再进行一次调用，将结果、callbackId回传回来，这端根据callbackId匹配相应的回调，将结果传入执行就可以了。

以Android，在Web端实现带有回调的JSB调用为例：
```JavaScript
// Web端代码：
<body>
  <div>
    <button id="showBtn">获取Native输入，以Web弹窗展现</button>
  </div>
</body>
<script>
  let id = 1;
  // 根据id保存callback
  const callbackMap = {};
  // 使用JSSDK封装调用与Native通信的事件，避免过多的污染全局环境
  window.JSSDK = {
    // 获取Native端输入框value，带有回调
    getNativeEditTextValue(callback) {
      const callbackId = id++;
      callbackMap[callbackId] = callback;
      // 调用JSB方法，并将callbackId传入
      window.NativeBridge.getNativeEditTextValue(callbackId);
    },
    // 接收Native端传来的callbackId
    receiveMessage(callbackId, value) {
      if (callbackMap[callbackId]) {
        // 根据ID匹配callback，并执行
        callbackMap[callbackId](value);
      }
    }
  };

    const showBtn = document.querySelector('#showBtn');
  // 绑定按钮事件
  showBtn.addEventListener('click', e => {
    // 通过JSSDK调用，将回调函数传入
    window.JSSDK.getNativeEditTextValue(value => window.alert('Natvie输入值：' + value));
  });
</script>
// Android端代码
webView.addJavascriptInterface(new NativeBridge(this), "NativeBridge");

class NativeBridge {
  private Context ctx;
  NativeBridge(Context ctx) {
    this.ctx = ctx;
  }

  // 获取Native端输入值
  @JavascriptInterface
  public void getNativeEditTextValue(int callbackId) {
    MainActivity mainActivity = (MainActivity)ctx;
    // 获取Native端输入框的value
    String value = mainActivity.editText.getText().toString();
    // 需要注入在Web执行的JS代码
    String jsCode = String.format("window.JSSDK.receiveMessage(%s, '%s')", callbackId, value);
    // 在UI线程中执行
    mainActivity.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        mainActivity.webView.evaluateJavascript(jsCode, null);
      }
    });
  }
}
```
以上代码简单实现了一个demo，在Web端点击按钮，会获取Native端输入框的值，并将值以Web端弹窗展现，这样就实现了Web->Native带有回调的JSB调用，同理Native->Web也是同样的逻辑，不同的只是将callback保存在Native端罢了，在此就不详细论述了。

## 总结
至此，大家应该对JSBridge的原理、使用有了一个比较深入的认知，这里对文章做一个总结：

Hybrid开发是目前移动端开发的主流技术选项，其中Native和Web端的双向通信就离不开JSBridge

其中Native调用Web端是直接在JS的Context直接执行JS代码，Web端调用Native端有两种方法，一种是基于URL Schema的拦截操作，另一种是向JS的Context（window）注入Api，其中注入Api是目前最好的选择。完整的调用是双向通信，需要一个回调函数，技术实现上就是使用了两次单向通信

其次，相对于造轮子，更推荐使用目前已经开源的JSBridge：DSBridge、jsBridge。
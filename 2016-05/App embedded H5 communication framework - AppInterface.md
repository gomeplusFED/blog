[AppInterface](https://github.com/yanglang1987500/AppInterface) — 让JSBridge更简单一点
==================================================

简介
----

事情起源于公司一个内部项目，App那边说要采用内嵌H5的形式来做，然而此前部门并没有一个成型的框架予以支持，于是上网搜集了一些关于App内嵌H5通信的资料，基于安卓与H5实现了一个通过拦截H5请求与JSBridge的框架，纯REST风格，安卓基于注解与反射实现，类似于SpringMVC的Controller实现

在讨论如何使用这套框架之前，咱们先从简单的原理说起
----
从搜索到的结果来看，采用的技术无非就以下几种
* 页面内嵌入一个iframe，通过修改iframe的src来让Webview拦截到来自网页的请求；
* 修改页面的location.href，让Webview拦截到来自网页的请求；
* 使用安卓的JSBridge；
* 使用iOS的JavascriptCore（iOS7.0版本后可用）

方案一应该是目前（或遗留项目）采用最多的方案，方案二是针对iOS9识别不到方案一而采用的打补丁方案，方案三与方案四应该是同级的，同时可以使用。<br>
那么，我们先来讨论方案一，从描述上来看，用js实现其实很简单：
```javascript
function call(url){
    var appIframe = doc.createElement('iframe');
    appIframe.id = 'appInterfaceNativeFrame';
    appIframe.style.display = 'none';
    body.appendChild(appIframe);
    appIframe.src = url;
}
```
其中，url内容由我们组织，比如：`gomeplus://common/toast?msg=弹个提示&jsCallback=TEMPORARYEVENT_1462255203268`，类似http请求一样，我们也将它分为三部分：
* `gomeplus://`，与`http://`一样，称为协议；
* `common`，与`www.google.com`一样，称为host地址；
* `toast`，与`/query`一样，定位资源，称为path；
* `msg`与`jsCallback`，url参数，其中`jsCallback`是H5传递给App的回调方法名称；

这样，我们就实现了一个基本的通信方法。
<br>
然而这只是一个基本的实现，你肯定很快就发现这样实现很不优雅，回调方法都绑定在window对象下面，随着调用次数的增多如果不删除，垃圾方法会越来越多，而且这样直接暴露方法在window下其实既不优雅也不安全。<br>
所以重构之后，加入了事件订阅与发布机制，在修改iframe.src之前先订阅一个事件，事件名称根据时间缀生成（比如上文中的`TEMPORARYEVENT_1462255203268`），然后修改src发送请求，App在接收到请求后处理该url，提取host与path进行业务处理，最后根据回调事件名称通过`Webview.loadUrl("javascript:try%7Bwindow.AppInterface.notify('TEMPORARYEVENT_1462255203268'%2C%7Bdata%7D)%3B%7Dcatch(e)%7B%7D")`进行事件发布即可。<br>
但是很快App同事说在iOS9上识别不到请求，于是把方案二补充进来：
```javascript
function doCall(url,force){
    var doc = document,body = doc.body;
    if(AppInterface.isIOS9) {
        //IOS9特殊处理
        window.location = url;
    } else {
        if(!appIframe){
            appIframe = doc.createElement('iframe');
            appIframe.id = 'appInterfaceNativeFrame';
            appIframe.style.display = 'none';
            body.appendChild(appIframe);
        }
        appIframe.src = url;
    }
}
```
<br>
然后新需求来了，需要设定超时时间，若App在给定时间内未回调的话，需要告知调用者调用超时了，所以补充一个定时器：
```javascript
if( timeout !== 0 ){
    var timer = setTimeout(function(){
        var endTime = new Date().getTime();
        //此处针对外部浏览器呼起APP做兼容。非浏览器或非ios9，应该通知  未进入后台，超时等于timeout，也应该通知
        if(!(endTime - parseInt(eventName.split('_')[1]) > (timeout+20)) || !(that.isBrowser && that.isIOS9)){
            that.notify(eventName,packageData(null,false,'客户端未响应'));
        }
        clearTimeout(timer);
        timer = null;
    }, timeout);
}
```
定时器这里也有一些可以聊的，那就是普通浏览器内针对App的唤醒，不过不在我们此次讨论的范围内，因为我们的目的只是通信，而非唤醒（因为唤醒也有一个很大的坑，由于iOS8之后会弹一个“是否使用****应用打开链接？”的确认框，所以目前在iOS8以上版本我们仍然解决不了用户未安装时引导用户去下载页的需求）<br>

你以为这样就完了么？还有一堆坑等着去填呢！~不过目前来说至少已经比较优雅的封装了H5这块的调用功能了。<br>
接下来看看AppInterface是如何实现的吧！<br><br>


使用指南 — 安卓方面
----

通过使用：
```Java
AppInterface.getInstance().init(this,"com.webview.sniyve.webview.controllers");
```
进行初始化工作。
第一个参数代表安卓`Context`对象，第二个参数是控制器所处包路径，框架会自动扫描此包路径下所有实现了`Controller`注解的类，并为其建立REST索引与反射并缓存。

###提供两种交互方式
* 使用URL拦截形式，此方式需要在`WebViewClient`实现类的`shouldOverrideUrlLoading`方法中进行拦截处理，直接调用
```Java
AppInterface.getInstance().handle(view,url);
```
即可，此方法会返回布尔值，为真代表匹配到了处理器，为假代表未匹配到处理器，理应进行放行。
* 使用JSBridge形式，此方式可以在webView实例化时直接调用
```Java
AppInterface.getInstance().initJsBridge(webView);
```
即可，框架会提供一个名为`ApplicationInterface`的js对象以供调用，js调用方法为
```javascript 
ApplicationInterface.call(url);
```
这两种方式可以并存，怎么调都行，拦截器都会拦截并通知相应的Controller进行处理，并统一进行回调处理。

###协议的实现-控制器的写法
控制器需要继承自`BaseController`，并在类上加`@Controller("host")`注解，并且需要在相应方法上加`@RequestMapping("/path")`注解。
待映射的协议方法需要实现两个入参`Map<String,Object> params`与`AppInterfaceCallback callback`，前一个是参数包，后一个是回调接口。`Controller`可以通过调用父类的`getContext()`方法获取Android上下文对象。<br>
使用示例如下：
```Java
@Controller("common")
public class CommonController extends BaseController{

    @RequestMapping("/toast")
    public void toast(Map<String,Object> params,AppInterfaceProvider provider){
        String message = (String)params.get("msg");
        Toast.makeText(this.getContext(), message, Toast.LENGTH_SHORT).show();
        provider.callback(null);
    }
}
```
如此便实现了一个简单的`common/toast`协议，所有参数都在params对象中，callback用于执行回调。
此框架建议与AppInterface.js进行搭配使用。

###新增了一套框架的广播订阅机制

test包中已经给出了使用示例<br>
具体使用方法如下：<br>
订阅：
    
```Java
AppInterface.getInstance().subscribe("onClick", new Callback() {
        @Override
        public void call(Map<String, Object> params) {
            TextView textView = (TextView) MainActivity.this.findViewById(R.id.textView);
            textView.setText((String) params.get("value"));
        }
    });
```
发布：
```Java
@RequestMapping("/order_manage")
public void orderManage(Map<String,Object> params, final AppInterfaceCallback callback){
        Map<String,Object> pms = new HashMap<String,Object>();
        pms.put("value", params.get("shopId"));
        AppInterface.getInstance().notify("onClick",pms);
    }
}
```
此模式可以解决不同类或不同Activity中互通事件的问题。

关于性能
-----
实测，Nexus4手机，框架初始化12个Controller，35个协议约35ms-70ms时间，转发请求耗时（不包括协议本身业务实现的耗时）为1ms，非常快！


使用指南 — H5方面
----

###H5主要通过AppInterface.js进行

AppInterface.js是一个通过js调用APP的工具包，同样内置了一套广播订阅机制与APP协议调用接口。但是调用APP不用关心广播，只需要使用
```javascript
AppInterface.call('/common/login',{
  	user:'test',
  	password:'test'
},function(data){
  	console.log(data.message);
});
```
即可与APP进行交互。其中，`/common/login`是约定协议，`{user:'test',password:'test'}`是参数包，第三个参数是APP回调方法；不论是通过拦截请求还是JsBridge方式，最终都会解析此协议，并通知相应的Controller进行处理。`AppInterface.call()`内部会优先使用JsBridge形式，如果APP环境尚未启用此功能，则使用发送请求的形式，最终达到的效果是一致的，AppInterface.js所提供的call功能不论是对于安卓还是对于iOS都是兼容的。
<br>
我们知道浏览器其实是单线程的，为了避开浏览器同一线程内的处理缺陷，AppInterface.js内部也实现了一个队列，比如连续两句代码修改location.href或iframe.src，APP的webview只能识别到第二次，第一次始终识别不到。于是便需要一个队列，内部使用定时器错开20ms，将多次内嵌调用错开在不同的线程时间段内：
```javascript
  queue:function(callback){
      if(arguments.length == 0 && !isCalling){
          _reCall();
          return this;
      }
      if(isCalling || !windowLoaded){
          toBeCall.push(callback);
          return this;
      }

      isCalling = true;
      callback();

      window.setTimeout(_reCall,QUEUE_TIMEOUT);
      function _reCall(){
          var flag = false;
          for(var i = 0;i < toBeCall.length;i++) {
              flag = true;
              toBeCall[i].call();
              window.setTimeout(arguments.callee,QUEUE_TIMEOUT);
              toBeCall.splice(i,1);
              break;
          }
          isCalling = flag;
      }
      return this;
  }
```
另外还有一种特殊情况需要拿出来说明一下，比如超链接上绑定埋点事件，埋点的实现主要是调用APP相关埋点协议，此时便会出现上面所说的情况！当用户点击超链接进行跳转之前，浏览器会先执行这个超链接上所绑定的事件，比如实现埋点的单击事件，而埋点势必会调用APP协议，然后AppInterface.js在APP未启用JSBridge时内部的实现是修改iframe.src（iOS9为修改location.href），紧接着浏览器实现超链接自身的跳转，这些操作都是在同一个线程内完成的，那么由上文所提供的结论我们可以得知--埋点肯定跪了。于是AppInterface.js内部提供了一个全局代理超链接跳转的方法，只需要在业务代码任意处调用该方法即可，代码如下所示：
```javascript
  /**
   * 由于内嵌的特殊性，需要代理超链接的默认跳转行为，将此跳转行为用js压入队列实现，否则APP会识别不到同一线程内的别的调用请求
   * @param $dom 可选a标签dom对象，也可以不传，默认代理所有页面内有href值的超链接
   */
  delegateHyperlink:function($dom){
      var that = this;
      //由于代理了a标签的点击事件，以免影响外部事件顺序，所以不能直接在主线程绑定
      //切入下一线程绑定，预留20ms，不保证外部延迟20ms以上再绑定事件顺序
      window.setTimeout(function(){
          if($dom){
              $dom.on('click',function(e){
                  delegate(e,$dom);
              });
          }else{
              $('body').on('click','a',function(e){
                  delegate(e,$(this));
              });
          }
      },20);

      function delegate(e,$dom){
          var _href = $dom.attr('href');
          if(_href && _href != '#' && !(/^javascript.*$/g).test(_href)){
              that.queue(function(){
                  window.location.href = _href;
              });
              e.preventDefault();//阻止超链接自身默认跳转事件发生
          }
      }
      return this;
  }
```
那么我们为什么不简单点直接对超链接使用点击事件进行跳转而要弄这么复杂呢？因为我们作为框架提供者，肯定不能去约束用户不能使用超链接的href跳转而采用绑定事件进行跳转，不过如果你硬是想要这样，也行，将跳转代码推入队列执行即可：
```javascript
$('a').on('click',function(){
  AppInterface.call('/common/statistics', {
      code: 'M000W005',
      desc: '为你推荐',
      param: base64.encode(JSON.stringify({productId: dataValue}))
  }).queue(function(){
    window.location.href = '';
  });
});
```

<br>
除此之外，AppInterface.js内部还封装了toast,alert,confirm,prompt四个方法，适配浏览器（主要便于调试）与APP两种环境，实现如下：
```javascript
  var AppAdapter = {
      toast:function(msg,timeout){
          AppInterface.call('/common/toast',{msg:msg,timeout:timeout?timeout:2000});
      },
      alert:function(msg,callback){
          AppInterface.call('/common/alert',{msg:msg},callback);
      },
      prompt:function(msg,content,callback){
          AppInterface.call('/common/prompt',{msg:msg,content:content},callback);
      },
      confirm:function(msg,callback){
          var title = '提示',content,callback, args = arguments;
          arguments.length == 2?function(){
              content = args[0];
              callback = args[1];
          }():arguments.length == 3 ? function(){
              title = args[0];
              content = args[1];
              callback = args[2];
          }():content = args[0];
          AppInterface.call('/common/confirm',{msg:content,title:title},callback);
      }
    };
    var BrowserAdpater = {
      toast:function(msg,timeout){
          var toast = document.createElement('div');
          toast.style.opacity = '0';
          toast.style.padding = '7px 10px';
          toast.style.minWidth = '80px';
          toast.style.color = '#fff';
          toast.style.textAlign = 'center';
          toast.style.position = 'fixed';
          toast.style.bottom = '10%';
          toast.style.left = '50%';
          toast.style.borderRadius = '3px';
          toast.style.fontSize = '14px';
          toast.style.transform = 'translateX(-50%)';
          toast.style.transition = 'opacity .3s ease';
          toast.style.backgroundColor = 'rgba(39, 39, 39, 0.6)';
          toast.innerHTML = '<p>'+msg+'</p>';
          document.body.appendChild(toast);
          setTimeout(function(){
              toast.style.opacity = '1';
          },50);
          setTimeout(function(){
              toast.style.opacity = '0';
              setTimeout(function(){
                  document.body.removeChild(toast);
              },300);
          },timeout?timeout:2000);
      },
      alert:function(msg,callback){
          alert(msg);
          callback && callback(packageData({},true,''));
      },
      prompt:function(msg,content,callback){
          var content = window.prompt(msg,content);
          callback && callback(packageData({content:content,isCancel:content == null},true,''));
      },
      confirm:function(msg,callback){
          var title = '提示',content,callback, args = arguments;
          arguments.length == 2?function(){
              content = args[0];
              callback = args[1];
          }():arguments.length == 3 ? function(){
              title = args[0];
              content = args[1];
              callback = args[2];
          }():content = args[0];
          var flag = window.confirm(content);
          callback && callback(packageData({isCancel:!flag},true,''));
      }
    };
    
    ['toast','alert','prompt','confirm'].forEach(function(key){
      AppInterface[key] = function(){
          AppInterface.isBrowser ?
              BrowserAdpater[key].apply(null,toBeNotify.slice.call(arguments)) :
              AppAdapter[key].apply(null,toBeNotify.slice.call(arguments));
      };
    });
```

<br><br><br>AppInterface最终的目标就是简化开发，提升开发效率与通信效率。目前只实现了安卓而没有实现iOS，一来是因为我iOS水平一般，就不班门弄斧了；二来iOS官方尚未提供注解实现，所以要实现智能配置比较别扭。不过AppInterface.js是支持iOS的，只需要iOS监听Webview的拦截请求方法或是通过JavascriptCore给Webview注入一个实现了call(url)方法的名为ApplicationInterface的javascript对象即可，只不过协议的实现仍需要自己解析url并回调罢了，没有框架的安卓实现那么简洁而已。

---
layout:     post
title:      "浏览器同源策略及跨域方案"
subtitle:   "浏览器同源策略及跨域方案"
date:       2016-6-10
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - browser
    - Browser
    - origin
    - Origin
    - same-origin
    - SAME-ORIGIN
    - same origin
    - Same Origin
    - cross-domain
    - CROSS-DOMAIN
    - cross domain
    - Cross Domain
    - ajax
    - Ajax
    - AJAX
    - cookie
    - Cookie
    - localStorage
    - LocalStorage
    - indexDB
    - indexDB
    - dom
    - DOM
    - document.domain
    - window.name
    - 片段识别符
    - #
    - fragment identifier
    - Fragment Identifier
    - window.postMessage
    - jsonp
    - JSONP
    - webSocket
    - WebSocket
    - cors
    - CORS
---

如何实现ajax的跨域请求,这几乎是所有前端面试的必问内容,即使是后端被问到的几率也不低.

最近针对这个问题进行了比较全面的了解,本篇文章总结了相关内容.

**想要理解跨域,就得从"同源策略"开始说起.**

# 一、同源策略
## **1.目的**

什么是同源策略,目的是为了什么?

我们都知道的是web环境是一个公开的环境,数不胜数的网站无法一一经过详细的审核.那么我们就无法避免恶意网站的存在.

试想一下,假如在这种情况下,浏览器没有任何限制.那么恶意站点就可以读取用户在其他所有站点的内容.

尤其是cookie通常存储着用户的登录状态,这种情况下,恶意站点可以模仿用户做任何操作,这样的事情想象一下就知道多么严重.

**同源策略就是**为了防止这种事情出现,**为了保证互联网用户的web环境安全而制定的规范.**

## **2.含义**

**同源策略(Same Origin Policy),顾名思义,就是指只有同源的网站能任意访问,而对不同源的网站则限制访问.**

具体要求为**三点相同,即协议相同、域名(iP)相同、端口相同**

>https://github.com/google/guava 以此为例
>
>https://github.com/google/auto：同源
>
>https://v2.github.com/google/auto：不同源,(域名不同)
>
>https://github.com:81/google/auto：不同源,(端口不同)
>
>http://github.com/google/auto：不同源,(协议不同)

**注意:一个操作的源以发起操作的URL为准**

例如:以下网址为我的博客的网址
>https://liudaomanbu.github.io/

我在博客的html页面中为了统计分析流量而引入了Baidu Analytic和Google Analytics的js代码

>\<script src="http://www.baidu.com/baidu-analytics.js">\</script>

虽然js文件来自http://www.baidu.com/baidu-analytics.js这个URL,但是js最终是在我的博客页面执行的.

因此执行时的源是https://liudaomanbu.github.io/

## **3.限制内容**

>(一) Cookie、LocalStorage 和 indexDB 无法读取
>
>(二) DOM 无法获得
>
>(三) AJAX 请求不能发送

**所以并不是只有ajax才有跨域,只是ajax的跨域最常使用.**

下面介绍有哪些跨域方案能够规避上面三种资源和行为的同源限制

# 二、跨域方案
## **1. document.domain**

### **(一)原理**
**document对象拥有domain属性用于获取/设置当前文档的原始域部分.可以使用js修改document.domain的值**

比如你在我的博客网页使用F12打开开发者工具,在控制台中输入document.domain,控制台就会输出"liudaomanbu.github.io".

### **(二)具体操作**
**通过document.domain属性设置站点的域,使需要互相读取cookie的两个站点的该属性值一致,就可以达到共享cookie的目标.**

### **(三)前提条件**
**两个域名必须属于同一个一级域名(因为无法修改domain为顶级域),而且所用的协议,端口都要一致.**(域名的简单相关知识不做介绍,如有疑问,请自行了解)

例如在我的博客页面输入如下代码
```javascript
document.domain="github.io"
```

结果将会是报错
>Uncaught DOMException: Failed to set the 'domain' property on 'Document': 'github.io' is a top-level domain.

例如对于这样一个域名
>a.b.c.d.com

如果输入
```javascript
document.domain="liudaomanbu.github.io"
```

那么结果将会是

>VM67:1 Uncaught DOMException: Failed to set the 'domain' property on 'Document': 'liudaomanbu.github.io' is not a suffix of 'a.b.c.d.com'.

其中is not a suffix of 'a.b.c.d.com'信息提示你set的值必须是原来域名的后缀,其实本质就是在限定为同一个基础域名.

```javascript
document.domain="b.c.d.com";
document.domain="c.d.com";
document.domain="d.com";
```

以上这些值就能被设置成功

### **(四)适用资源**

**Cookie、一级域名相同的iframe窗口和window.open方法打开的窗口与父窗口之间的Dom**

## **2. Cookie的domain**

### **(一)原理**

**服务器设置Cookie的时候,可以指定Cookie的所属域名**

### **(二)具体操作**

>**Set-Cookie: key=value; domain=.example.com; path=/**

### **(三)适用资源**

**Cookie**

## **3. window.name**

### **(一)原理**
**window.name属性拥有被生命周期内所有网页共享的特点.**

### **(二)具体操作**

**使用隐藏的iframe窗口读取不同源的内容存入window.name属性中,再在本页面使用**

### **(三)限制**

**window.name的值只能是字符串**

### **(四)适用资源**

**iframe窗口和window.open方法打开的窗口与父窗口之间的数据交换**

## **4. 片段识别符(fragment identifier)**

### **(一)原理**

**片段识别符指的是URL的#号后面的部分,由于这是设计用来在同一个页面中定位用的锚点,所以修改这部分内容并不会刷新和跳转页面,且是同一窗口下所有window共享的资源.**

但是这种做法其实属于hack型的行为

### **(二)具体操作**
父窗口写入子窗口的片段标识符
```javascript
document.getElementByid('id').onload = function(){
    var src = target + '#' + data;
    this.src = src;
}
```
子窗口写入父窗口的片段标识符
```javascript
parent.location.href= target + "#" + data;
```

**监听hashchange事件得到通知**
```javascript
window.onhashchange = getData;

function getData() {
  var data = window.location.hash;//结果为"#xxxxx"
}
```
### **限制**

**值只能是字符串**

### **适用资源**

**iframe窗口和window.open方法打开的窗口与父窗口之间的数据交换**

## **5. window.postMessage**

### **(一)原理**

**HTML5引入了一个全新的APi：跨文档通信 APi,(Cross-document messaging).** 
**这个APi为window对象新增了一个window.postMessage方法,允许跨窗口通信,不论这两个窗口是否同源.**

### **(二)具体操作**
**postMessage方法的第一个参数是具体的信息内容,第二个参数是接收消息的窗口的源,(origin),即"协议 + 域名 + 端口",也可以设为*,表示不限制域名,向所有窗口发送.**

发送信息代码
```javascript
window.postMessage('message', 'http://aaa.com');
```

通过**message事件,监听收到的消息**,接收信息代码
```javascript
window.onmessage = function(e){
    console.log(e.data);//message
}
```
**message事件的事件对象event,提供以下三个属性,**

>event.source：发送消息的窗口
>
>event.origin: 消息发向的网址
>
>event.data: 消息内容

### **(三)适用资源**

**iframe窗口和window.open方法打开的窗口与父窗口之间的数据交换**

## **6. JSONP**

### **(一)原理**

严格的同源策略很多时候为开发带来了许多不便,考虑到这一情况,某些方面会放松一些限制.

**为了便于开发,\<script>、\<img>、\<iframe>、\<link>等拥有src属性的标签都是不受同源策略限制的,可以跨域加载资源的.**

**当然,为了安全考虑,浏览器限制了js对这些标签加载的资源的操作权限,无法读写这些标签加载的资源.**

这些标签每次加载时,实际上是由浏览器发起了一次**GET请求**.

事实上**由于将js文件存在外站带来的不可靠性,script标签的正常跨域功能形同虚设.**

**但是JSONP的原理就是利用\<script>标签的跨域性来发起请求,利用\<script>标签自动将返回内容视为js文件并自动执行的特性实现跨域.**

### **(二)具体操作**
```javascript
function addScriptTag(src) {
  var script = document.createElement('script');
  script.setAttribute("type","text/javascript");
  script.src = src;
  document.body.appendChild(script);
}

window.onload = function () {
  addScriptTag('http://example.com/ip?callback=foo');
}

function foo(data) {
  console.log('Your public iP address is: ' + data.ip);
};
```

jquery框架中$.getJSON封装了JSONP跨域方式,$.getJSON方法会自动判断是否跨域,不跨域的话,就调用普通的ajax方法.跨域的话,则会以异步加载js文件的形式来调用jsonp的回调函数.

### **(三)限制**

**只能是Get方式,不能使用RESTful规范设计APi,并且浏览器中有参数长度限制**

### **(四)适用资源**

**ajax**

## **7. WebSocket**

### **(一)原理**

**WebSocket是一种通信协议,使用ws://,(非加密)和wss://,(加密)作为协议前缀,该协议不实行同源政策,只要服务器支持,就可以通过它进行跨源通信.**

### **(二)具体操作**

以下是WebSocket的浏览器请求头信息,摘自[维基百科](https://en.wikipedia.org/wiki/WebSocket)

>GET /chat HTTP/1.1
>
>Host: server.example.com
>
>Upgrade: websocket
>
>Connection: Upgrade
>
>Sec-WebSocket-Key: x3JJHMbDL1EzLkh9GBhXDw==
>
>Sec-WebSocket-Protocol: chat, superchat
>
>Sec-WebSocket-Version: 13
>
>Origin: http://example.com

**Origin字段表示该请求的请求源,由服务器根据这个字段,判断是否许可本次通信.**

### **(三)适用资源**

**ajax**

## **8. CORS(Cross-Origin Resource Sharing)**

### **(一)原理**

CORS是跨源资源分享的缩写,它是W3C标准,是跨源AJAX请求的根本解决方法.

**CORS方案与WebSocket原理相近,浏览器请求头附带Origin字段表示该请求的请求源,由服务器根据这个字段,判断是否许可本次通信.**

**注意：这个跨域访问方案的安全基础就是信任“JavaScript无法控制该HTTP头”，如果此信任基础被打破，则此方案也将不再安全.**

**相比JSONP只能发GET请求,CORS允许任何类型的请求.**

**浏览器方面对于CORS的支持是自动完成的**,不需要任何操作和设置,因此**实现CORS通信的关键是服务器.只要服务器实现了CORS接口,就可以跨源通信.**

### **(二)限制**

**CORS需要浏览器和服务器同时支持**

![浏览器对于CORS的支持情况](/img/in-post/2016-06-10-js-browser-same-origin-policy-and-cross-domain/cors-browser-support.png)

**现代浏览器基本都能支持,iE浏览器不能低于iE10,移动终端上,除了opera Mini.**

### **(三)具体操作**

仅列出Java后端的配置,其他后端请自行寻找.

#### **(1)添加CORS过滤器**

首先是添加依赖,maven依赖如下
```xml
<dependency>
    <groupId>com.thetransactioncompany</groupId>
    <artifactId>cors-filter</artifactId>
    <version>[ version ]</version>
</dependency>
```
接下来是将CORS过滤器配置到Web.xml中
```xml
<!-- 跨域配置-->    
<filter>
        <!-- The CORS filter with parameters -->
        <filter-name>CORS</filter-name>
        <filter-class>com.thetransactioncompany.cors.CORSFilter</filter-class>
        
        <!-- Note: All parameters are options, if omitted the CORS 
             Filter will fall back to the respective default values.
          -->
        <init-param>
            <param-name>cors.allowGenericHttpRequests</param-name>
            <param-value>true</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.allowOrigin</param-name>
            <param-value>*</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.allowSubdomains</param-name>
            <param-value>false</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.supportedMethods</param-name>
            <param-value>GET, HEAD, POST, OPTIONS</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.supportedHeaders</param-name>
            <param-value>Accept, Origin, X-Requested-With, Content-Type, Last-Modified</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.exposedHeaders</param-name>
            <!--这里可以添加一些自己的暴露Headers   -->
            <param-value>X-Test-1, X-Test-2</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.supportsCredentials</param-name>
            <param-value>true</param-value>
        </init-param>
        
        <init-param>
            <param-name>cors.maxAge</param-name>
            <param-value>3600</param-value>
        </init-param>

    </filter>

    <filter-mapping>
        <!-- CORS Filter mapping -->
        <filter-name>CORS</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
```

注意请将该过滤器放在第一个,并且注意与安全权限框架的冲突影响.

#### **(2)Spring Boot配置**

```java
@Configuration
public class CorsConfig {

    private CorsConfiguration buildConfig() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        
        // 可以自行筛选
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        
        return corsConfiguration;
    }

    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        
        source.registerCorsConfiguration("/**", buildConfig());
        
        return new CorsFilter(source);  
    }
}
```

### **(三)流程详解**
#### **(1)两种请求**

**浏览器将CORS请求分成两类：简单请求(simple request)和非简单请求(not-so-simple request).**

**只要同时满足以下两大条件,就属于简单请求,否则就是非简单请求(也可称为复杂请求).**

>（1) 请求方法是以下三种方法之一：
>
> * HEAD
>
> * GET
>
> * POST
>
>（2）HTTP的头信息不超出以下几种字段：
>
> * Accept
>
> * Accept-Language
>
> * Content-Language
>
> * Last-Event-iD
>
> * Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

浏览器对于两种请求的处理方式不同.

#### **(2)简单请求**

**浏览器发现这次跨源AJAX请求是简单请求,就自动在头信息之中,添加一个Origin字段.**

例子

>GET /cors HTTP/1.1
>
>Origin: http://api.bob.com
>
>Host: api.alice.com
>
>Accept-Language: en-US
>
>Connection: keep-alive
>
>User-Agent: Mozilla/5.0...

**服务器根据Origin字段的值是否在许可范围内,来决定是否正常回应请求.**

**Origin字段的值在许可范围内时,服务器返回的响应头会多出三个跨域字段.**

> Access-Control-Allow-Origin: http://api.bob.com
>
> Access-Control-Allow-Credentials: true
>
> Access-Control-Expose-Headers: FooBar

##### **服务器响应头字段含义**

* Access-Control-Allow-Origin

**该字段是必须的.它的值要么是请求时Origin字段的值,要么是一个 * ,表示接受任意域名的请求.**

**如果要发送Cookie,Access-Control-Allow-Origin就不能设为星号,必须指定明确的、与请求网页一致的域名.**

* Access-Control-Allow-Credentials

**该字段可选.**
它的值是一个布尔值,**表示是否允许发送Cookie**.
设为true,即表示服务器明确许可,Cookie可以包含在请求中,一起发给服务器.这个值也只能设为true,如果服务器不要浏览器发送Cookie,删除该字段即可.

**默认情况下,Cookie不包括在CORS请求之中.**

**Cookie依然遵循同源政策，只有用服务器域名设置的Cookie才会上传，其他域名的Cookie并不会上传**

* Access-Control-Expose-Headers

**该字段可选.**
CORS请求时,**XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma.**

**如果想拿到其他字段,就必须在Access-Control-Expose-Headers里面指定.**

上面的例子指定,getResponseHeader('FooBar')可以返回FooBar字段的值.

##### **请求字段含义**

* withCredentials

**AJAX请求中该属性值为true才能发送cookie.**

``` javascript
var xhr = new XMLHttpRequest();
xhr.withCredentials = true;
```

#### **(3)非简单请求**

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json.
##### **①预检请求（preflight）**

**非简单请求的CORS请求，会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求.**

浏览器先询问服务器，当前网页所在的域名是否在服务器的许可名单之中，以及可以使用哪些HTTP动词和头信息字段.只有得到肯定答复，浏览器才会发出正式的XMLHttpRequest请求，否则就报错.

下面是一个预检请求的请求头信息

>OPTIONS /cors HTTP/1.1
>
>Origin: http://api.bob.com
>
>Access-Control-Request-Method: PUT
>
>Access-Control-Request-Headers: X-Custom-Header
>
>Host: api.alice.com
>
>Accept-Language: en-US
>
>Connection: keep-alive
>
>User-Agent: Mozilla/5.0...

**"预检"请求用的请求方法是OPTIONS，表示这个请求是用来询问的.**

##### **请求头字段含义**

* Access-Control-Request-Method

**该字段必须,用来说明正式的CORS请求的HTTP方法类型**

* Access-Control-Request-Headers

**该字段可选,一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段.**

##### **②预检请求的回应**

服务器收到"预检"请求以后，检查了Origin、Access-Control-Request-Method和Access-Control-Request-Headers字段以后，确认允许跨源请求，就可以做出回应.

##### **I.预检请求通过**

**预检请求通过后,服务器会在响应头中返回CORS相关字段.**

##### **响应头字段含义**

* Access-Control-Allow-Origin

**该字段必需,表示服务器允许的源.**

* Access-Control-Allow-Methods

**该字段必需,逗号分隔的一个字符串，表明服务器支持的所有跨域请求的方法.**

**注意，返回的是所有支持的方法，而不单是浏览器请求的那个方法.这是为了避免多次"预检"请求.**

* Access-Control-Allow-Headers

**如果浏览器请求包括Access-Control-Request-Headers字段，则Access-Control-Allow-Headers字段是必需的.**

**一个逗号分隔的字符串，表明服务器支持的所有头信息字段，不限于浏览器在"预检"中请求的字段.**

* Access-Control-Allow-Credentials

该字段与简单请求时的含义相同.

* Access-Control-Max-Age

**该字段可选，用来指定本次预检请求的有效期，单位为秒.在此期间，不用发出另一条预检请求.**

##### **Ⅱ.预检请求不通过**

**如果预检请求不通过，会返回一个正常的HTTP回应，但是没有任何CORS相关的头信息字段.**

**浏览器**就会认定，服务器不同意预检请求，因此**触发一个错误，被XMLHttpRequest对象的onerror回调函数捕获.**

控制台会打印出如下的报错信息.
```javascript
XMLHttpRequest cannot load http://api.alice.com.
Origin http://api.bob.com is not allowed by Access-Control-Allow-Origin.
```

##### **③浏览器的正常请求和回应**

**一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段.服务器的回应，也都会有一个Access-Control-Allow-Origin头信息字段.**

### **(三)常见错误**

#### **(1)HTTP Code 404**

##### **①现象**
>No 'Access-Control-Allow-Origin' header is present on the requested resource,并且The response had HTTP status code 404

![404](/img/in-post/2016-06-10-js-browser-same-origin-policy-and-cross-domain/cors-error-404.png)

##### **②原因**

* 本次ajax请求是“非简单请求”,所以请求前会发送一次预检请求(OPTIONS)
* 服务器端后台接口没有允许OPTIONS请求,导致无法找到对应接口地址

##### **③解决方案**

后端允许options请求

#### **(2)HTTP Code 405**

##### **①现象**
>No 'Access-Control-Allow-Origin' header is present on the requested resource,并且The response had HTTP status code 405

![405](/img/in-post/2016-06-10-js-browser-same-origin-policy-and-cross-domain/cors-error-405.png)

##### **②原因**

* 本次ajax请求是“非简单请求”,所以请求前会发送一次预检请求(OPTIONS)
* 后台方法允许OPTIONS请求,但是一些配置文件中(如安全配置),阻止了OPTIONS请求

##### **③解决方案**

后端关闭对应的安全配置

#### **(3)HTTP Code 200**

##### **①现象**
>No 'Access-Control-Allow-Origin' header is present on the requested resource,并且status 200

![200](/img/in-post/2016-06-10-js-browser-same-origin-policy-and-cross-domain/cors-error-200.png)

##### **②原因**

* 本次ajax请求是“非简单请求”,所以请求前会发送一次预检请求(OPTIONS)
* 服务器端后台允许OPTIONS请求,并且接口也允许OPTIONS请求,但是头部匹配时出现不匹配现象

比如origin头部检查不匹配,比如少了一些头部的支持(如常见的X-Requested-With头部),然后服务端就会将response返回给前端,前端检测到这个后就触发XHR.onerror,导致前端控制台报错

##### **③解决方案**

后端增加对应的头部支持

# 总结
**如果没有兼容老版本浏览器、请求的网站不支持CORS等特殊的原因,不同源之间的数据通讯应该使用ajax采用最新的CORS方案完成.**

**如果是需要共享cookie,则采用修改domain值的方式来完成.**

其他跨域方案都是历史方案,可以仅作了解即可.

# 参考列表
[浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)

[JS解决跨域汇总](http://blog.csdn.net/FrontEnder_way/article/details/70568113)

[跨域资源共享 CORS 详解](http://www.ruanyifeng.com/blog/2016/04/cors.html)

[ajax跨域，这应该是最全的解决方案了](https://segmentfault.com/a/1190000012469713)
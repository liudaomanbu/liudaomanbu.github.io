---
layout:     post
title:      "debug记录:图标加载与tomcat的session机制引发的bug"
subtitle:   "由请求资源路径错误和火狐浏览器的网络请求显示自动过滤引发的debug血案"
date:       2017-4-18
author:     "caotc"
header-img: "/img/home-bg.jpg"
tags:
    - Java
    - java
    - Debug
    - debug
    - Bug
    - bug
    - favicon.ico
    - tomcat
    - Tomcat
    - firefox
    - Firefox
    - 火狐
    - session
    - Session
---


前几天,被分到修一个线上项目存在已久的闪退bug.

**该闪退bug特点:**
**1.只在线上环境出现,本地的环境不出现该bug.**

**2.只在某个浏览器被打开后,第一次进入登录页,登录该项目时出现.**

**即闪退后再登录,第二次开始进入登录页(无论是否同一标签页)均不会出现闪退.**

**3.补充:使用的运行环境服务器是tomcat.**


该bug的这些特点给debug造成了极大的麻烦.

**关于这种闪退的bug,我自然第一反应就是跟session有关.**

**我第一步就决定从浏览器开发者工具里的http请求的sessionId开始查起,使用的是火狐浏览器.**

**请注意,我特地提到了火狐浏览器.**

对于火狐浏览器中http请求的记录如下截图.
![闪退首次进入登录页请求](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/first-login-logout-request.png)
![其他请求](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/ohter-request.png)

由于登录到闪退阶段的请求没法详细看到,所以只能暂时不管.
看其他请求,可以看出,在闪退后再次进入登录页,以及登录后的浏览器端的cookie的sessionId都是同一个id,所以一切正常.
而闪退前的登录页请求中,浏览器所发送的id发生了3次变化.显然有不正常的事情在里面.

但是在浏览器段已经得不到更多的信息,所以只能进入艰苦的服务器端debug历程.
由于该bug的特点,服务器端的debug只能通过在各个拦截器和控制器中增加日志的打印,然后传到测试线上环境,重启服务,然后看日志.
这个过程异常的艰苦,重启一次线上项目要3.4分钟,然后将自己操作的日志保存下来,在大量的日志里面一行行的查看,然后发现某个地方的日志不够详细,再增加日志记录,循环往复.

就这样断断续续地持续了两天多.得到的筛选后的详细日志如下.
闪退第一次进登录页:
```
[2017-04-16 00:07:48] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for [/gov/]
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():false [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@14018e9a(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 115
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@14018e9a(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:07:48 115 [2017-04-16
00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():true [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:48] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/login] [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@601093d9(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 115
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@652333b7(JSESSIONID:28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@601093d9(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:07:48 115 [2017-04-16
00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.web.controllers.LoginController - /login:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:07:48] WARN  c.u.web.controllers.LoginController -
/login: request.isRequestedSessionIdValid():true [2017-04-16 00:07:48]
WARN  c.u.web.controllers.LoginController - /login: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.web.controllers.LoginController -
cookie:javax.servlet.http.Cookie@652333b7(JSESSIONID:28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.web.controllers.LoginController -
/login: Cookie[] cookies = request.getCookies(): [2017-04-16 00:07:48]
WARN  c.u.web.controllers.LoginController - /login: HttpSession
session =
request.getSession(false):org.apache.catalina.session.StandardSessionFacade@601093d9(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.web.controllers.LoginController -
/login: session.getCreationTime():2017-04-16 00:07:48 115 [2017-04-16
00:07:48] WARN  c.u.web.controllers.LoginController - /login:
session.isNew():false [2017-04-16 00:07:48] WARN 
c.u.web.controllers.LoginController - /login:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:48] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/loginBodyPage] [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@3b2ac1e0(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 115
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@3c8e34b1(JSESSIONID:28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@3b2ac1e0(id=28BD603105F2B2BC00159BB4F9F42D71-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:07:48 115 [2017-04-16
00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:48] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/validate] [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():false [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@774abc95(JSESSIONID:3820B6B1BC094DE63AEDC6C8A974C17C)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@4ba31762(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:48] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:48] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@4ba31762(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:07:48] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():true [2017-04-16 00:07:48] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:07:48] WARN
c.u.w.c.ValidateCodeController - validateCode:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:07:48] WARN  c.u.w.c.ValidateCodeController -
validateCode: request.isRequestedSessionIdValid():false [2017-04-16
00:07:48] WARN  c.u.w.c.ValidateCodeController - validateCode:
HttpSession session =
reqeust.getSession():org.apache.catalina.session.StandardSessionFacade@4ba31762(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:48] WARN  c.u.w.c.ValidateCodeController -
validateCode: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:48] WARN  c.u.w.c.ValidateCodeController -
validateCode: session.isNew():true [2017-04-16 00:07:48] WARN 
c.u.w.c.ValidateCodeController - validateCode:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:49] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/assets/js/module/dialog/jquery.dialog.js] [2017-04-16 00:07:49]
WARN  com.unengli.web.SessionListener - sessionCreated: HttpSession
session =
event.getSession():org.apache.catalina.session.StandardSessionFacade@6d1201b5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:49] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/assets/js/module/ua/jquery.ua.js] [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@3757d7c4(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:49] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/assets/js/module/screen/jquery.screen_adaptive.js] [2017-04-16
00:07:49] WARN  com.unengli.web.SessionListener - sessionCreated:
HttpSession session =
event.getSession():org.apache.catalina.session.StandardSessionFacade@7fa404e9(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:49] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/assets/js/module/form/jquery.form.js] [2017-04-16 00:07:49] WARN
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@69fca8e2(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:07:49] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/assets/js/loginBody.js] [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated: HttpSession session
= event.getSession():org.apache.catalina.session.StandardSessionFacade@50bcea00(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:07:49] WARN  com.unengli.web.SessionListener -
sessionCreated: session.isNew():true [2017-04-16 00:07:49] WARN 
com.unengli.web.SessionListener - sessionCreated:
session.getAttribute(USER_SESSION_KEY):null
```
闪退至再次进入登录页日志:

```
[2017-04-16 00:08:30] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing POST request for [/gov/loginBodyLogin]
[2017-04-16 00:08:30] WARN  com.unengli.web.SessionListener - sessionCreated: HttpSession session = event.getSession():org.apache.catalina.session.StandardSessionFacade@e8a1ca5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  com.unengli.web.SessionListener - sessionCreated: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:30] WARN  com.unengli.web.SessionListener - sessionCreated: session.isNew():true
[2017-04-16 00:08:30] WARN  com.unengli.web.SessionListener - sessionCreated: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@3c7fdca5(JSESSIONID:BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@e8a1ca5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:30] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - cookie:javax.servlet.http.Cookie@3c7fdca5(JSESSIONID:BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: authFacade.queryUserModelByloginName(loginName):com.unengli.model.auth.UserModel@340bd72e
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: HttpSession session = request.getSession(false):org.apache.catalina.session.StandardSessionFacade@e8a1ca5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.isNew():false
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: userModel = account.get():com.unengli.model.auth.UserModel@7374d04a
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@e8a1ca5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.isNew():false
[2017-04-16 00:08:30] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:31] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.setAttribute(SessionConstants.USER_SESSION_KEY, userModel):org.apache.catalina.session.StandardSessionFacade@e8a1ca5(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:31] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:31] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.isNew():false
[2017-04-16 00:08:31] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@7374d04a
[2017-04-16 00:08:31] WARN  c.u.web.controllers.LoginController - /loginBodyLogin: userModel:com.unengli.model.auth.UserModel@7374d04a

[2017-04-16 00:08:31] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/]
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@460401f8(accountRegionCode:0)
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@311d64b1(JSESSIONID:BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:08:31] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - addSession: sessions:
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - session.getLastAccessedTime():2017-04-16 00:08:31 071
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:08:31] WARN  com.unengli.web.SessionManager - addSession: sessions:

[2017-04-16 00:08:32] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/countryEnergyMonitor/countryEnergyMonitor]
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@674f7aa8
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@12fde095(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@59a35ff0(JSESSIONID:BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - addSession: sessions:
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getLastAccessedTime():2017-04-16 00:08:32 042
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getCreationTime():2017-04-16 00:07:48 979
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getLastAccessedTime():2017-04-16 00:08:32 319
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionManager - addSession: sessions:

[2017-04-16 00:08:32] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/countryEnergyMonitor/totalEnergyConsumption]
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@674f7aa8
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():false
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@3075a2af(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@7b0ba803(JSESSIONID:22A581417C09727EB8F4251F45E3E6AF)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: HttpSession session = event.getSession():org.apache.catalina.session.StandardSessionFacade@3a439c11(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.isNew():true
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@3a439c11(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():true
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:08:32] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/loginview]
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: HttpSession session = event.getSession():org.apache.catalina.session.StandardSessionFacade@2854203b(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.isNew():true
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@2b10162c(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@2d1fb8ca(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@2854203b(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: model.put('userModel', new UserModel()):com.unengli.model.auth.UserModel@7d9cecea
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@2854203b(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - cookie:javax.servlet.http.Cookie@2b10162c(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - cookie:javax.servlet.http.Cookie@2d1fb8ca(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.web.controllers.LoginController - /loginview: Cookie[] cookies = request.getCookies():

[2017-04-16 00:08:32] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/loginBodyPage]
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: HttpSession session = event.getSession():org.apache.catalina.session.StandardSessionFacade@5a3acf7a(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.isNew():true
[2017-04-16 00:08:32] WARN  com.unengli.web.SessionListener - sessionCreated: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@674f7aa8
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@53a36bf2(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@e0407c4(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@5a3acf7a(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null

[2017-04-16 00:08:32] DEBUG com.excenergy.ewf.FrontendServlet - DispatcherServlet with name 'ewf' processing GET request for [/gov/validate]
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@8a84c86(accountRegionCode:0)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - cookie:javax.servlet.http.Cookie@6137f5da(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: Cookie[] cookies = request.getCookies():
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: HttpSession session = request.getSession(true):org.apache.catalina.session.StandardSessionFacade@5a3acf7a(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.w.c.SecurityInterceptor - before: session.getAttribute(USER_SESSION_KEY):null
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: request.isRequestedSessionIdValid():true
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: HttpSession session = reqeust.getSession():org.apache.catalina.session.StandardSessionFacade@5a3acf7a(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: session.isNew():false
[2017-04-16 00:08:32] WARN  c.u.w.c.ValidateCodeController - validateCode: session.getAttribute(USER_SESSION_KEY):null
```

正常登陆日志:
```
[2017-04-16 00:09:13] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing POST request for
[/gov/loginBodyLogin] [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@57bac06b(accountRegionCode:0)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@25aeddcc(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@7b638f67(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:09:13] WARN
c.u.web.controllers.LoginController - /loginBodyLogin:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: request.isRequestedSessionIdValid():true [2017-04-16
00:09:13] WARN  c.u.web.controllers.LoginController - /loginBodyLogin:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:13] WARN 
c.u.web.controllers.LoginController -
cookie:javax.servlet.http.Cookie@57bac06b(accountRegionCode:0)
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
cookie:javax.servlet.http.Cookie@25aeddcc(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: Cookie[] cookies = request.getCookies(): [2017-04-16
00:09:13] WARN  c.u.web.controllers.LoginController - /loginBodyLogin:
authFacade.queryUserModelByloginName(loginName):com.unengli.model.auth.UserModel@6485447f
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: HttpSession session =
request.getSession(false):org.apache.catalina.session.StandardSessionFacade@7b638f67(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.isNew():false [2017-04-16 00:09:13] WARN 
c.u.web.controllers.LoginController - /loginBodyLogin:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:09:13] WARN
c.u.web.controllers.LoginController - /loginBodyLogin: userModel =
account.get():com.unengli.model.auth.UserModel@235cd792 [2017-04-16
00:09:13] WARN  c.u.web.controllers.LoginController - /loginBodyLogin:
session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@7b638f67(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.isNew():false [2017-04-16 00:09:13] WARN 
c.u.web.controllers.LoginController - /loginBodyLogin:
session.getAttribute(USER_SESSION_KEY):null [2017-04-16 00:09:13] WARN
c.u.web.controllers.LoginController - /loginBodyLogin:
session.setAttribute(SessionConstants.USER_SESSION_KEY,
userModel):org.apache.catalina.session.StandardSessionFacade@7b638f67(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.getCreationTime():2017-04-16 00:08:32 476
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: session.isNew():false [2017-04-16 00:09:13] WARN 
c.u.web.controllers.LoginController - /loginBodyLogin:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@235cd792
[2017-04-16 00:09:13] WARN  c.u.web.controllers.LoginController -
/loginBodyLogin: userModel:com.unengli.model.auth.UserModel@235cd792

[2017-04-16 00:09:13] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for [/gov/]
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@6720cdff(accountRegionCode:0)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@1ab4553e(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:13] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:13] WARN  c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:13] WARN 
com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 042 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 354 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:13 420 [2017-04-16
00:09:13] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:13] WARN  com.unengli.web.SessionManager -
addSession: sessions:

[2017-04-16 00:09:14] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/countryEnergyMonitor/countryEnergyMonitor] [2017-04-16 00:09:14]
WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@674f7aa8
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@1ba02b7f(accountRegionCode:0)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@3087ad04(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@1b6ffa80(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:14] WARN 
com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 042 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 354 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:13 733 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@1b6ffa80(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 012 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions:

[2017-04-16 00:09:14] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing GET request for
[/gov/countryEnergyMonitor/totalEnergyConsumption] [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@712d1ae4(accountRegionCode:0)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@4840a8c8(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@d090787(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@4bcd36c0
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@4bcd36c0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:14] WARN 
com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 042 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 354 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:13 733 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@1b6ffa80(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 032 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@d090787(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 154 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@4bcd36c0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions:

[2017-04-16 00:09:14] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing POST request for
[/gov/countryEnergyMonitor/getTotalEnergyConsumptionMapData]
[2017-04-16 00:09:14] DEBUG com.excenergy.ewf.FrontendServlet -
DispatcherServlet with name 'ewf' processing POST request for
[/gov/countryEnergyMonitor/getYearTotalEnergyConsumptions] [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@1b16164
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@6d084260(accountRegionCode:0)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@393de48c(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@e67c0b0(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request:org.apache.catalina.connector.RequestFacade@7b97c65d
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
request.isRequestedSessionIdValid():true [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: Cookie[] cookies =
request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@5d194044(accountRegionCode:0)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor -
cookie:javax.servlet.http.Cookie@24ca601c(JSESSIONID:8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
Cookie[] cookies = request.getCookies(): [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before: HttpSession session =
request.getSession(true):org.apache.catalina.session.StandardSessionFacade@e67c0b0(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.isNew():false [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:14] WARN 
com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 042 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 354 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:13 733 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@1b6ffa80(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 032 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@d090787(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 801 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@4bcd36c0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@e67c0b0(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 935 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:14] WARN 
c.u.w.c.SecurityInterceptor - before:
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions: [2017-04-16 00:09:14] WARN 
com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@bdb9a8(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 042 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@27e7c4f9
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@74efb389(id=BC4A03518FE5CD8F9B1C4088A2A3CBB5-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:07:48 979 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:08:32 354 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@24f42359
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@35486724(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:13 733 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@246b87d0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@1b6ffa80(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 032 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@54a6f792
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@d090787(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 801 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@4bcd36c0
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@e67c0b0(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 935 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session:org.apache.catalina.session.StandardSessionFacade@e67c0b0(id=8FC50A8E085CC69C0679B31CCBB5A355-n1)
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getCreationTime():2017-04-16 00:08:32 476 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager -
session.getLastAccessedTime():2017-04-16 00:09:14 935 [2017-04-16
00:09:14] WARN  com.unengli.web.SessionManager - session.isNew():false
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
session.getAttribute(USER_SESSION_KEY):com.unengli.model.auth.UserModel@3d9b6a33
[2017-04-16 00:09:14] WARN  com.unengli.web.SessionManager -
addSession: sessions:
```
期间,怀疑过多个方向.
1.从闪退第一次进登录页和闪退至再次进入登录页日志来看,项目在request.isRequestedSessionIdValid()为true,即请求中带有有效的SessionId时依然会触发监听器中的sessionCreated事件,并且还是在进入拦截器SecurityInterceptor之前,即根本还没执行到request.getSession(true)方法时.(例子见闪退第一次进登录页"/gov/login"的请求日志).
是否会是在登录过程中,在已有正常session的情况下,拿到了刚创建的空session,里面没有用户属性,所以在拦截器中被弹回登录页?

2.即使在request.isRequestedSessionIdValid()为true,且没有触发监听器中的sessionCreated事件的请求中,每次得到的session对象依然是不同的.(例如闪退至再次进入登录页日志中,"/gov/loginBodyLogin","/gov/","/gov/countryEnergyMonitor/countryEnergyMonitor"这顺序的三次请求,只有首次请求"/gov/loginBodyLogin"触发了sessionCreated事件.)
是否会是在登录过程中,某次请求时由于getSession方法拿到的session对象不是放入用户属性的那个session对象,所以认为其没有登录,被弹回登录页?

3.是否是SessionManager这个自定义的Session维护类中的定时清理方法将登录后进入首页时的session误清理,调用了session.invalidate()方法,将session状态标识为无效,导致拦截器认为其没有登录,被弹回登录页?

然而以上三种可能性,随着最后这版详细的日志出来后都否定了.
**从日志来看,最直接的原因是在登录后,首页的请求"/gov/countryEnergyMonitor/totalEnergyConsumption"中,浏览器发送的sessionId"22A581417C09727EB8F4251F45E3E6AF"为从来没被服务器端生成过的Id,与登录时记录的sessionId不符,于是只能由服务器新建一个不同id的session,而连id都是新生成的session,自然是空的session,无法通过拦截器中的登录校验,于是就被弹回到了登录页.**

但是搜遍了整个服务器日志,也没有搜到"22A581417C09727EB8F4251F45E3E6AF"这个sessionId是在哪里产生的,按理说浏览器的sessionId只会由服务器设置后返回,我又设置了sessionCreate事件的监听,怎么会出现浏览器莫名其妙发来了一个sessionId,然而服务器端却根本没有生成过这个sessionId的事情发生呢?

我开始怀疑,难道这是火狐浏览器特有的bug,会在某种情况下自己创建或者不知道从哪里得到一个sessionId然后发送给服务器?
但是我又返回去看bug描述,写的是最先在谷歌浏览器发现该bug,火狐浏览器是后来加的,并且bug描述中并没有说这个bug会因为浏览器的不同而消失.
那么多个浏览器都写出了一模一样的bug,这个可能性也实在太低了.但是不管怎么说,看看其他浏览器是个什么状况吧,就先从谷歌开始看.
![谷歌浏览器请求](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/chrome-request.png)
跟之前的火狐浏览器请求记录统计相比,你们是否发现有什么明显的不同?
没错,![火狐浏览器请求](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/firefox-request.png)
**跟火狐浏览器相比,明显的谷歌浏览器多出了一个请求,而且是状态为404的请求"unengli.ico".**
打开具体内容一看![unengli.ico详情](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/unengli.ico-request.png)
**果然这个请求是关键性的请求,在这个"unengli.ico"的请求中,明明request的cookie中带了正确的sessionId,但是response对象中却给cookie设置了新的,不存在于项目中的sessionId.**

根据Referer:http://10.30.22.18:9537/gov/login这个属性,我开始找这个请求的发起来源.果然来自于这个路径返回的页面login.html中的资源请求.
![ico发起请求地点](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/ico-from.png)
很明显,其他资源的路径全部都是由框架映射过的,会自动在前面补上项目名作为路径前缀.而这三个资源请求是直接写在html页面的路径,直接就是以/开头的,所以发出来的请求实际上是http://10.30.22.18:9537/unengli.ico.
大概是由于在tomcat看来,这不是gov路径项目的请求,而是另一个项目的请求,所以返回了新的sessionId?但是为什么要这么做呢?
我开始为了验证自己的猜测和解决自己的疑惑去寻找tomcat的的session管理机制.
找到了这么一段内容:
按照Servlet规范,session的作用范围应该仅仅限于当前应用程序下,不同的应用程序之间是不能够互相访问对方的session的.各个应用服务器从实际效果上都遵守了这一规范,但是实现的细节却可能各有不同,因此解决跨应用程序session共享的方法也各不相同. 

首先来看一下Tomcat是如何实现web应用程序之间session的隔离的,从Tomcat设置的cookie路径来看,它对不同的应用程序设置的 cookie路径是不同的,这样不同的应用程序所用的session id是不同的,因此即使在同一个浏览器窗口里访问不同的应用程序,发送给服务器的session id也可以是不同的.
![这里写图片描述](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/tomcat-session.png)
![这里写图片描述](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/weblogic-session.png)

如果这个说法是正确的话,那么就能验证我的猜测,并且解释我的疑惑了.
**由于tomcat实现session的方式是所有项目共用同一个hashtable的session表,所以为了实现Servlet规范.**
**如果当前请求的sessionId所对应的项目与当前的请求的根路径不同,且状态码为404时认为是请求了另一个项目,所以改变sessionId**

**这就是火狐浏览器请求统计表中,闪退首次进入登录页的请求中"/gov/assets/js/base/jquery.js"的请求到"/gov/assets/js/module/dialog/jquery.dialog.js"的请求,莫名其妙地更改了sessionId的原因所在.**

*但是火狐浏览器却没有显示这个请求,看来火狐浏览器是会屏蔽不是同一个项目的http请求,这是坑爹啊.完全是在误导啊,要是用的是谷歌浏览器,我第一天就会发现这个异常了.*

**但是即使如此,这个问题,我也不认为和闪退有关系,毕竟在加载登录页的时候,最后的sessionId已经恢复正常了,我猜想闪退的原因应该是在登录过程中发送了一个类似的请求,导致在登录过程中sessionId发生了改变,所以闪退.**

于是我再次用谷歌浏览器进行闪退的登录,并且谷歌浏览器还有一个好处就是,能够在闪退后跳回登录页时保留登录至闪退的浏览器请求,而火狐浏览器我找了半天都没找到这个设置到底在哪里.

以下为谷歌浏览器登录至闪退的http请求的截图
![这里写图片描述](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/chrome-first-login-logout-request.png)
**果然我发现了在登录至闪退过程中也有"/favicon.ico"这样的一个请求,改变了sessionId导致了闪退,而第二次登录项目时,是不会再次发送这个请求的,所以第二次开始就不会闪退了.**

也就是说,只要把"/favicon.ico"的请求找到之后,修正为正确路径或者直接干掉,那么这个闪退的bug应该就解决了.
**但是奇怪的事情发生了,我完全没有在请求来源的页面找到这个请求,无论是在js还是在html页面中都没有这个路径的请求.使用ide的全局搜索,还是没有找到任何包含"favicon"的字符.**
于是debug又陷入了僵局.没办法,我只好先把login页面的三个错误路径请求删除.没想到删除之后,闪退的bug就直接消失了.我奇怪地再去查看首次进入登录页的请求.
![这里写图片描述](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/chrome-first-login-logout-request2.png)
**原来这个"/favicon.ico"请求其实不是项目发的,而是浏览器自己发的,而且大多数浏览器都会这么干,因为这是历史遗留的做法."/favicon.ico"是默认的网站左上角图片的加载路径.**就是这种![这里写图片描述](http://img.blog.csdn.net/20170420194047495?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY2FvMTk5MjA0MjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
**而登录页那3个错误路径请求存在其实是在定义浏览器,左上角的图片不要从默认路径取,而是从指定的这个路径取.只是不知道为什么路径下也没有文件.但是登陆后的首页没有这三个资源定义,会导致浏览器重新从默认路径"/favicon.ico"去请求.**


所以终于解决这个bug.

留下的疑问:

**1.火狐浏览器的屏蔽规则到底是什么?**
**本来我以为是不同根路径的请求会屏蔽,但是我在登录页加上了**
```html
<link rel="stylesheet" type="text/css" media="all" href="/login_cs_ie.css" />
<link rel="stylesheet" type="text/css" media="all" href="/login_cs_ie" />
```
**这两个错误路径的请求之后,火狐浏览器依旧会显示这两个错误路径的http请求.**
![这里写图片描述](/img/in-post/2017-04-18-java-debug-session-logout-favicon.ico-tomcat/question1.png)*

**2.为什么项目运行在本地的tomcat时,并不会发出"/unengli.ico"和"/favicon.ico"请求,也就不会产生闪退的bug?**

**3.根据上面weblogic的session内存结构图,采用这种session实现机制,在这种情况下,是否就不会更改sessionId?**
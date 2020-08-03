---
title: "SpringBoot and Web容器"
category: SpringBoot
tag: [SpringBoot, mvc, web, servlet]
---
## 前言
现在很多程序员接触java web开发都是从SpringBoot开始，其独立jar运行和免配置的易用性很好的屏蔽了很多繁琐的工作，但也容易让人忽略底层的实现原理，比如：web容器。本文以一个请求为例，简单描述一下其到达服务器后所经历的调用流程。

首先，一个http请求通过域名解析到的ip连接web服务器，然后，根据端口号找到对应的提供服务的应用，这个应用就是web容器，如：Tomcat容器。我们写的SpringBoot应用在启动时就会先启动一个web容器，默认是Tomcat容器。
## Tomcat
Tomcat会首先启动一个Server，Server里面可以启动多个Service，一个Service可以有多个Connector和一个Engine，每个Connector负责监听一个ip:port对，负责与客户端建立连接，每个Connector可以有自己的线程连接池也可以共用连接池。Connector接受到请求之后，会封装成Request和Response对象，然后调用Engine容器的invoke方法。一共有四种不同层级的容器：Engine、Host、Context和Wrapper，一个Engine包含多个Host，一个Host包含多个Context(web应用)，一个Context包含多个Wrapper，每个Wrapper对应一个Servlet。Engine是容器的总入口，Host可以看做是虚拟主机，DNS域名解析的时候，可以将不同的域名解析到同一个ip，http/1.1的请求头中可以通过Host字段来指定域名，这些域名分别对应不同的Host。然后会根据规则，根据每个域名后面的path匹配最相似的Context和Wrapper(Servlet)。Wrapper是Servlet的封装，负责其创建、调用和销毁，
## Servlet
Servlet内就是我们实现业务逻辑的地方。狭义的讲，一个Servlet程序就是实现了Sevlet接口的类，其主要有三个方法：init，service和destroy。之前Web应用的开发都是Servlet的开发，没有main函数，打成war包，由web容器来进行调度。
1.Web容器首先检查是否已经装载并创建了该Servlet的实例对象。如果是，则直接执行第3步，否则，执行第2步。
2.调用Servlet实例对象的init()方法，创建并装载该Servlet的一个实例对象。
3.创建一个用于封装HTTP请求消息的HttpServletRequest对象和一个代表HTTP响应消息的HttpServletResponse对象，然后调用Servlet的service()方法并将请求和响应对象作为参数传递进去。
4.WEB应用程序被停止或重新启动之前，将调用Servlet的destroy()方法。
默认web容器是采用单实例多线程的方式调用Servlet的，所以应尽量避免使用实例变量，只要局部变量。

Web应用的配置是在web.xml中，主要包括context-param，listener，filter，servlet等，其加载和调用顺序如下：context-param --> listener --> filter --> servlet。context-param用来配置初始化参数，会被存入ServletContext中，由所有Sevlet共享。listener用于监听web容器中发生的各种event，例如：客户端的请求，session的创建和销毁，应用的创建和销毁，ServletContext属性的改变等。filter用于实现调用Servlet之前和之后的各种通用操作，比如：日志、审核、入参校验等。
## Spring Web MVC
Spring Web MVC就是遵循MVC（Model-View-Controller）的Web应用架构，其核心就是围绕一个Dispatcher Servlet展开的，所有的请求都会指派给这个Servlet，然后根据url从HandlerMapping中获取HandlerMethod和所有的HandlerInterceptor放入HandlerExecutionChain，然后通过HandlerMethod获取对应的HandlerAdapter。之后调用完所有拦截器的preHandle后，才是通过HandlerAdapter.handle真正调用controller方法。这里首先需要通过HandlerMethodArgumentResolver做参数解析，然后通过HandlerMethodReturnValueHandler做返回值解析，这里都是采用了策略模式。然后就是执行所有拦截器的postHandle方法，处理全局异常、视图渲染以及执行所有拦截器的afterCompletion方法。

在Spring Web MVC中，主要的配置也是在web.xml中，会配置ContextLoaderListener，负责启动和关闭Spring root WebApplicationContext（父IOC容器），会读取context-param中contextConfigLocation对应的文件，负责bean的创建、保存、获取。rootWebApplicationContext会作为一个属性注入ServletContext中。在DispatcherServlet配置里的init-param参数内也会定义一个contextConfigLocation参数，其对应的文件会定义该Servlet需要的类，如：Controller、view resolvers等和web相关的一些bean。在Servlet的init函数里会创建Servlet WebApplicationContext（子IOC容器），负责该Servlet相关类的创建、保存、获取，且rootWebApplicationContext会作为父容器被传入进来。

可以看到，在Spring Web MVC中还是会需要配置dispatcher servlet，a view resolver, web jars，DataSource等等，于是引入了SpringBoot的auto configuration功能。
## SpringBoot
SpringBoot可以根据classpath中引入的类来进行自动配置，比如：如果spring-webmvc jar被引入则自动配置Dispatcher Servlet。在application.properties中配置logging.level.org.springframework=debug可以看到自动配置日志。

其实现逻辑都在spring-boot-autoconfigure.jar中，所有用到的自动配置类都列在 /META-INF/spring.factories中。当应用使用了@EnableAutoConfiguration注解后，应用会读取spring.factories里列出的auto configure类来进行配置,比如：DispatcherServletAutoConfiguration.

[Java Web从如何启动到Servlet&Tomcat](https://www.jianshu.com/p/a4f652eb1fbe)

[深入理解Tomcat（八）Container](https://www.jianshu.com/p/1a2d2f868143)

[Context Path](https://blog.csdn.net/xh16319/article/details/8014193)

[Spring Boot vs. Spring MVC vs. Spring: How Do They Compare?](https://dzone.com/articles/spring-boot-vs-spring-mvc-vs-spring-how-do-they-compare)

[What is spring boot auto configuration](https://techrocking.com/what-is-spring-boot-auto-configuration)

[Web MVC framework](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/mvc.html)

[这一次搞懂SpringMVC原理](https://blog.csdn.net/l6108003/article/details/106770028)

[Spring系列（一）：Spring MVC bean 解析、注册、实例化流程源码剖析](https://zhuanlan.zhihu.com/p/62979297)

[Spring 系列（二）：Spring MVC的父子容器](https://zhuanlan.zhihu.com/p/63212113)

[这一次搞懂Spring Web零xml配置原理以及父子容器关系](https://www.bbsmax.com/A/D854YZjVdE)

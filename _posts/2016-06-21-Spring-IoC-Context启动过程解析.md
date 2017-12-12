---
title: Spring IoC Context启动过程解析
date: 2016-06-21 21:57:12
tags: Spring
toc: true
---

## ServletContext

`Web`容器在启动的过程中，会为每个`Web`应用程序创建一个对应的`ServletContext`对象，它代表了当前的`Web`应用，为`Spring IoC`容器提供宿主环境。

在部署`Web`工程的时候，`Web`容器会读取`web.xml`，创建`ServletContext`，当前`Web`工程所有部分都共享这个`Context`。`context-param`为`ServletContext`提供键值对，即`Servlet`上下文的信息，这些信息`Listener`、`Filter`和`Servlet`都有可能使用到，因此先加载`context-param`，创建`ServletContext`，然后加载`Listener`，再加载`Filter`，最后加载`Servlet`。

接下来我将按照这个加载顺序来分析`Spring`容器的启动过程。

## ContextLoaderListener

`web.xml`中配置有`ContextLoaderListener`，也可以自定义一个实现了`ServletContextListener`接口的`Listener`类，`web.xml`中的配置实例如下。

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

`Web`容器在启动的过程中会触发`ServletContextEvent`事件，会被`ContextLoaderListener`监听到，并调用`ContextLoaderListener`中的`contextInitialized`方法，`contextInitialized`方法如下所示。

```java
public void contextInitialized(ServletContextEvent event) {
        this.initWebApplicationContext(event.getServletContext());
}
```

`ContextLoaderListener`类继承了`ContextLoader`，在初始化`Context`的过程中，调用`ContextLoader`的`initWebApplicationContext`方法初始化`WebApplicationContext`。`WebApplicationContext`是一个接口，`Spring`默认的实现类为`XmlWebApplicationContext`，`XmlWebApplicationContext`就是`Spring`的`IoC`容器。

在初始化`XmlWebApplicationContext`之前，`Web`容器已经加载了`context-param`，`web.xml`中的`context-param`实例如下所示。作为`Spring`的`IoC`容器，其对应的`Bean`定义的配置正是`context-param`指定的。

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:applicationContext.xml</param-value>
</context-param>
```
接着进入到`initWebApplicationContext`方法内，`initWebApplicationContext`方法定义如下（已省略部分代码）。

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {      
    if(servletContext.getAttribute(
        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
            throw new IllegalStateException("...");
    }else{
        if(this.context == null) {
            this.context = this.createWebApplicationContext(servletContext);
        }
    }
    servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
}
```
在`Spring IoC`容器初始化前，`initWebApplicationContext`先检测以`ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`为`key`的值是否为空，若不为空，则初始化`IoC Context`，并在初始化完毕后，以`ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`为`key`将`IoC Context`存储到`ServletContext`中。

## 初始化Servlet

`Servlet`可以在`web.xml`中配置多个，在`Spring`中，最基本的`Servlet`为`DispatcherServlet`，对应的配置实例如下所示。
```xml
<servlet>
    <servlet-name>appServlet</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring/appServlet/appServlet-context.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

`DispatcherServlet`会建立自己的`IoC Context`，用以持有相关的`Bean`，在初始化自己的`IoC Context`的过程中，先通过`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`，从`ServletContext`中获取`WebApplicationContext`，将`WebApplicationContext`作为`DispatcherServlet的IoC Context`的 `parent Context`。`DispatcherServlet`自己的`IoC Context`的初始化工作在`DispatcherServlet`的`initStrategies`方法中完成，包括控制器映射，视图解析等，`initStrategies`
方法如下所示。

```java
protected void initStrategies(ApplicationContext context) {
    this.initMultipartResolver(context);
    this.initLocaleResolver(context);
    this.initThemeResolver(context);
    this.initHandlerMappings(context);
    this.initHandlerAdapters(context);
    this.initHandlerExceptionResolvers(context);
    this.initRequestToViewNameTranslator(context);
    this.initViewResolvers(context);
    this.initFlashMapManager(context);
}
```

`DispatcherServlet`自己的`IoC Context`的类型也是`XmlWebApplicationContext`，初始化完毕后，`Spring`将以与`DispatcherServlet`的`servlet-name`属性相关的符号作为`key`，将`IoC Context`保存到	`ServletContext`中。这样每个`Servlet`就都可以持有自己的`Context`，也就是都拥有自己的`Bean`空间，同时，各个`Servlet`之间还共享着`key`为`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`的`WebApplicationContext`，其中定义的`Bean`为各个`Servlet`共享的`Bean`。

## 参考

* [segmentfault](https://segmentfault.com/q/1010000000210417)


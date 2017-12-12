---
title: Spring IoC容器初始化 — Resource定位源码分析
date: 2016-12-07 15:53:56
tags: Spring
toc: true
---

在`Spring IoC`容器的设计中，有两个主要的容器系列。一个是实现了`BeanFactory`接口的简单容器系列，这系列容器只实现了容器基本的功能；另一个是`ApplicationContext`应用上下文，它在简单容器的基础上增加了许多面向框架的特性，同时对应用环境做了许多适配。

## IoC容器的初始化过程

`Spring IoC`容器的初始化过程分为三个阶段：`Resource`定位、`BeanDefinition`的载入和向`IoC`容器注册`BeanDefinition`。`Spring`把这三个阶段分离，并使用不同的模块来完成，这样可以让用户更加灵活的对这三个阶段进行扩展。

* `Resource`定位指的是`BeanDefinition`的资源定位，它由`ResourceLoader`通过统一的`Resource`接口来完成，`Resource`对各种形式的`BeanDefinition`的使用都提供了统一的接口。
* `BeanDefinition`的载入是把用户定义好的`Bean`表示成`IoC`容器内部的数据结构，而这个容器内部的数据结构就是`BeanDefinition`，`BeanDefinition`实际上就是`POJO`对象在`IoC`容器中的抽象。通过`BeanDefinition`，`IoC`容器可以方便的对`POJO`对象进行管理。
* 向`IoC`容器注册`BeanDefinition`是通过调用`BeanDefinitionRegistry`接口的实现来完成的，这个注册过程是把载入的`BeanDefinition`向`IoC`容器进行注册。实际上，在`IoC`容器内部维护着一个`HashMap`，而这个注册过程其实就将`BeanDefinition`添加至这个`HashMap`。

我们可以自己定义`Resource`、`BeanFactory`和`BeanDefinitionReader`来初始化一个容器。如下代码片段使用了`DefaultListableBeanFactory`作为实际使用的`IoC`容器。同时，创建`IoC`配置文件（`dispatcher-servlet.xml`）的抽象资源，这个抽象资源包含了`BeanDefinition`的定义信息。最后，还需要创建一个载入`BeanDefinition`的读取器，此处使用`XmlBeanDefinitionReader`，通过一个回调配置给`BeanFactory`。

```java
ClassPathResource res = new ClassPathResource("dispatcher-servlet.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

我们也可以通过`ApplicationContext`创建一个`IoC`容器。在`Spring`中，系统已经提供许多定义好的容器实现，而不需要自己组装。如下代码片段以`FileSystemXmlApplicationContext`为例创建了一个`IoC`容器。

```java
FileSystemXmlApplicationContext context = 
        new FileSystemXmlApplicationContext("classpath:dispatcher-servlet.xml");
```

无论使用哪种方式初始化`IoC`容器，都会经历上述三个阶段。本篇文章将结合`Spring 4.0.2`源码，并以`FileSystemXmlApplicationContext`为例对`IoC`容器初始化的第一阶段，也就是`Resource`定位阶段进行分析。

## BeanDefinition的Resource定位

下图展示了`FileSystemXmlApplicationContext`的继承体系，`FileSystemXmlApplicationContext`继承自`AbstractApplicationContext`，而`AbstractApplicationContext`又继承自`DefaultResourceLoader`，`DefaultResourceLoader`实现了`ResourceLoader`接口。因此`FileSystemXmlApplicationContext`具备读取定义了`BeanDefinition`的`Resource`的能力。

![Alt text](/img/2016-12-07-Image 1.png)

我们的分析入口是`new FileSystemXmlApplicationContext("classpath:dispatcher-servlet.xml");`，这句代码调用了`FileSystemXmlApplicationContext`的构造方法。`FileSystemXmlApplicationContext`的构造方法源码如下（只提取与本次分析关联的代码）。

```java
    public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
    }

    public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, 	
                                           ApplicationContext parent) throws BeansException {
        super(parent);
        this.setConfigLocations(configLocations);
        if(refresh) {
            this.refresh();
        }
    }
```

在创建`FileSystemXmlApplicationContext`时，我们仅传入了包含`BeanDefinition`的配置文件路径（`classpath:dispatcher-servlet.xml`），由此调用`FileSystemXmlApplicationContext(String configLocation)`构造方法。接着，`FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)`构造方法被间接调用，在该构造方法内部，`refresh`方法完成了整个`IoC`容器的初始化。因此，`refresh`方法是我们分析的下一个入口。

`refresh`方法的具体实现定义在`FileSystemXmlApplicationContext`的父类`AbstractApplicationContext`中，对应的源码如下。

```java
    public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var5) {
                this.destroyBeans();
                this.cancelRefresh(var5);
                throw var5;
            }

        }
    }
```

在`refresh`方法中，通过`obtainFreshBeanFactory`方法，`ConfigurableListableBeanFactory`类型的`BeanFactory`被创建。我们接着进入`obtainFreshBeanFactory`方法，`obtainFreshBeanFactory`方法也定义在`AbstractApplicationContext`中。

```java
    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if(this.logger.isDebugEnabled()) {
            this.logger.debug("Bean factory for " + this.getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
    }

    protected abstract void refreshBeanFactory() throws BeansException, IllegalStateException;
```

我们重点关注`refreshBeanFactory`方法的实现。在`AbstractApplicationContext`中，`refreshBeanFactory`方法仅仅是个声明，具体的实现委托给了子类完成。此处，`refreshBeanFactory`方法的具体实现定义在了`AbstractRefreshableApplicationContext`，`AbstractRefreshableApplicationContext`正是继承自`AbstractApplicationContext`，这点我们可以从上文的继承体系图可以得知。`refreshBeanFactory`方法在`AbstractRefreshableApplicationContext`中的定义如下。

```java
    protected final void refreshBeanFactory() throws BeansException {
        if(this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
            DefaultListableBeanFactory ex = this.createBeanFactory();
            ex.setSerializationId(this.getId());
            this.customizeBeanFactory(ex);
            this.loadBeanDefinitions(ex);
            Object var2 = this.beanFactoryMonitor;
            synchronized(this.beanFactoryMonitor) {
                this.beanFactory = ex;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
        }
    }

    protected abstract void loadBeanDefinitions(DefaultListableBeanFactory var1) throws BeansException, IOException;
```

`refreshBeanFactory`方法首先会判断是否已经建立的`BeanFactory`，如果已经建立，那么需要销毁并关闭该`BeanFactory`。接着，`refreshBeanFactory`方法通过`createBeanFactory`方法创建了一个`IoC`容器供`ApplicationContext`使用，且这个`IoC`容器的实际类型为`DefaultListableBeanFactory`。同时，`refreshBeanFactory`方法将这个`IoC`容器作为参数，调用`loadBeanDefinitions`载入了`BeanDefinition`（本文暂不分析载入过程的具体操作）。

`loadBeanDefinitions`方法也仅仅在`AbstractRefreshableApplicationContext`中声明，具体的实现定义在`AbstractXmlApplicationContext`中，从继承体系图我们可以得知`AbstractXmlApplicationContext`正是`AbstractRefreshableApplicationContext`的子类。`loadBeanDefinitions`方法对应的源码如下。

```java
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        this.initBeanDefinitionReader(beanDefinitionReader);
        this.loadBeanDefinitions(beanDefinitionReader);
    }

    protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = this.getConfigResources();
        if(configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if(configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }
    }
```

在`loadBeanDefinitions(DefaultListableBeanFactory beanFactory)`中，定义了`BeanDefinition`的读入器`beanDefinitionReader`。`Spring`把定位、读入和注册的过程解耦，这正是体现之处之一。接着`beanDefinitionReader`作为参数，调用`loadBeanDefinitions(XmlBeanDefinitionReader reader)`方法，如果`configResources`为空，那么`reader`就会根据`configLocations`调用`reader`的`loadBeanDefinitions`去加载相应的`Resource`。在`AbstractBeanDefinitionReader`和`XmlBeanDefinitionReader`中个自定义了不同的`loadBeanDefinitions`方法，与我们本次分析相关的代码定义在`AbstractBeanDefinitionReader`中，如下所示。

```java
    public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
        Assert.notNull(locations, "Location array must not be null");
        int counter = 0;
        String[] var3 = locations;
        int var4 = locations.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String location = var3[var5];
            counter += this.loadBeanDefinitions(location);
        }

        return counter;
    }

    public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
        return this.loadBeanDefinitions(location, (Set)null);
    }

    public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
        ResourceLoader resourceLoader = this.getResourceLoader();
        if(resourceLoader == null) {
            throw new BeanDefinitionStoreException("Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
        } else {
            int loadCount;
            if(!(resourceLoader instanceof ResourcePatternResolver)) {
                Resource var11 = resourceLoader.getResource(location);
                loadCount = this.loadBeanDefinitions((Resource)var11);
                if(actualResources != null) {
                    actualResources.add(var11);
                }

                if(this.logger.isDebugEnabled()) {
                    this.logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
                }

                return loadCount;
            } else {
                try {
                    Resource[] resource =
                        ((ResourcePatternResolver)resourceLoader).getResources(location);
                    loadCount = this.loadBeanDefinitions(resource);
                    if(actualResources != null) {
                        Resource[] var6 = resource;
                        int var7 = resource.length;

                        for(int var8 = 0; var8 < var7; ++var8) {
                            Resource resource1 = var6[var8];
                            actualResources.add(resource1);
                        }
                    }

                    if(this.logger.isDebugEnabled()) {
                        this.logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
                    }

                    return loadCount;
                } catch (IOException var10) {
                    throw new BeanDefinitionStoreException("Could not resolve bean definition resource pattern [" + location + "]", var10);
                }
            }
        }
    }

```

在`loadBeanDefinitions(String location, Set<Resource> actualResources)`方法中，我们可以看到，`Resource`的定位工作交给了`ResourceLoader`来完成。对于取得`Resource`的具体过程，我们可以看看`DefaultResourceLoader`是怎样完成的，对应源码如下。

```java
    public Resource getResource(String location) {
        Assert.notNull(location, "Location must not be null");
        if(location.startsWith("classpath:")) {
            return new ClassPathResource(location.substring("classpath:".length()),
                                         this.getClassLoader());
        } else {
            try {
                URL ex = new URL(location);
                return new UrlResource(ex);
            } catch (MalformedURLException var3) {
                return this.getResourceByPath(location);
            }
        }
    }
```

由于我们传入的`location`为`classpath:dispatcher-servlet.xml`，因此`getResource`方法会生成一个`ClassPathResource`并返回，如果我们传入的是一个文件路径，那么会调用`getResourceByPath`方法，`getResourceByPath`方法定义在`FileSystemXmlApplicationContext`中，对应的源码如下。

```java
    protected Resource getResourceByPath(String path) {
        if(path != null && path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemResource(path);
    }
```

到此，我们完成了`IoC`容器在初始化过程中的`Resource`定位过程的流程分析，这为接下来进行`BeanDefinition`数据的载入和解析创造了条件。

后续我会对`BeanDefinition`的载入和解析过程结合源码进行分析，欢迎关注。若本文存在分析不妥之处，建议发送邮件至`tinylcy (at) gmail.com`交流，直接在页面评论亦可。
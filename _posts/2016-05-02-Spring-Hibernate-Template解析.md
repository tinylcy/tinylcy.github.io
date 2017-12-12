---
title: Spring Hibernate Template解析
date: 2016-05-02 23:34:16
tags: Spring
toc: true
---


在`Spring`中，如果选用`Hibernate`作为持久层框架，往往需要在`beans.xml`中配置好`SessionFactory`，然后将`SessionFactory`注入到对应的`DAO`类。当我们使用`SessionFactory`来进行`CRUD`，配合对应的异常处理，会使得真正有用的业务逻辑代码显得微不足道。而且，除了那部分业务逻辑，创建`Session`、开启事务、处理异常、关闭资源这一系列代码在大多数场景下都是重复的。为了解决这个问题，`Spring`引入`Hibernate Template`让我们专注于业务逻辑代码。

## 问题


举个简单栗子，当前我们需要将`User`保存到数据库中，并且在保存用户的同时，在日志表中插入一条记录。因此，我们引入`Spring`中针对`Hibernate`的声明式事务管理，并在`Service`层方法上添加事务。需要两个`DAO`分别负责`User`和`Log`，此处忽略`Transaction`的配置，将重心放在`Hibernate Template`。

在`beans.xml`中配置好`Component`，`DataSource`，`SessionFactory`和`TransactionManager`，`UserDAOImpl`负责将`User`插入到数据库中。

```java
@Component(value = "userDAO")
public class UserDAOImpl implements UserDAO {

    private SessionFactory sessionFactory;

    public void save(User user) {
        try {
            Session session = sessionFactory.getCurrentSession();
            session.save(user);
        } catch (HibernateException e) {
            e.printStackTrace();
        }
    }

    public SessionFactory getSessionFactory() {
        return sessionFactory;
    }

    @Resource(name = "mySessionFactory")
    public void setSessionFactory(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }
}

```

`LogDAOImpl`将一条日志插入到数据库。

```java
@Component(value = "logDAO")
public class LogDAOImpl implements LogDAO {

    private SessionFactory sessionFactory;

    public void save(Log log) {
        try {
            Session session = sessionFactory.getCurrentSession();
            session.save(log);
        } catch (HibernateException e) {
            e.printStackTrace();
        }
    }

    public SessionFactory getSessionFactory() {
        return sessionFactory;
    }

    @Resource(name = "mySessionFactory")
    public void setSessionFactory(SessionFactory sessionFactory) {
        this.sessionFactory = sessionFactory;
    }
}

```

注意到在`UserDAOImpl` 和`LogDAOImpl` 的`save`方法中，真正和业务逻辑相关的代码只有`session.save()`。`Hibernate Template`的作用就是将这部分重复的代码抽出来，作为一个模板，我们在使用`Hibernate Template`时只需关注具体业务。

## 配置Hibernate Template

配置`Hibernate Template`只需引入`HibernateTemplate`，并将`SessionFactory`注入到`HibernateTemplate`即可。此处为了说明的连续性，将`DataSource`，`SessionFactory`和`HibernateTemplate`的配置信息都附上。

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <!--===============================Spring整合Hibernate=============================-->
    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <value>classpath:jdbc.properties</value>
        </property>
    </bean>

    <!--配置数据源-->
    <bean id="myDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
        <property name="driverClassName" value="${jdbc.driverClassName}"></property>
        <property name="url" value="${jdbc.url}"></property>
        <property name="username" value="${jdbc.username}"></property>
        <property name="password" value="${jdbc.password}"></property>
    </bean>

    <!--设置Hibernate的SessionFactory-->
    <bean id="mySessionFactory" class="org.springframework.orm.hibernate3.annotation.AnnotationSessionFactoryBean">
        <!--配置数据源-->
        <property name="dataSource" ref="myDataSource"></property>

        <!--设置Hibernate的配置信息-->
        <property name="hibernateProperties">
            <props>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
                <prop key="hibernate.show_sql">true</prop>
            </props>
        </property>

        <!--告诉容器去扫描哪些包里的实体类-->
        <property name="packagesToScan">
            <list>
                <value>entity</value>
            </list>
        </property>
    </bean>

    <!--========================设置HibernateTemplate=======================-->
    <bean id="hibernateTemplate" class="org.springframework.orm.hibernate3.HibernateTemplate">
        <property name="sessionFactory" ref="mySessionFactory"></property>
    </bean>
</beans>

```

将`HibernateTemplate`注入到`UserDAOImpl`和`LogDAOImpl`，代码精简了好多。

```java
@Component(value = "userDAO")
public class UserDAOImpl implements UserDAO {

    private HibernateTemplate hibernateTemplate;

    public void save(User user) {
        hibernateTemplate.save(user);
    }

    public HibernateTemplate getHibernateTemplate() {
        return hibernateTemplate;
    }

    @Resource(name = "hibernateTemplate")
    public void setHibernateTemplate(HibernateTemplate hibernateTemplate) {
        this.hibernateTemplate = hibernateTemplate;
    }
}
```

```java
@Component(value = "logDAO")
public class LogDAOImpl implements LogDAO {

    private HibernateTemplate hibernateTemplate;

    public void save(Log log) {
        hibernateTemplate.save(log);
    }

    public HibernateTemplate getHibernateTemplate() {
        return hibernateTemplate;
    }

    @Resource(name = "hibernateTemplate")
    public void setHibernateTemplate(HibernateTemplate hibernateTemplate) {
        this.hibernateTemplate = hibernateTemplate;
    }
}
```


## 浅析Hibernate Template


那么`Hibernate Template`内部到底是怎么实现的？查看`HibernateTemplate`的源码，其中的`save`方法如下。

```java
    public Serializable save(final Object entity) throws DataAccessException {
        return (Serializable)this.executeWithNativeSession(new HibernateCallback() {
            public Object doInHibernate(Session session) throws HibernateException {
                HibernateTemplate.this.checkWriteOperationAllowed(session);
                return session.save(entity);
            }
        });
    }
```

`save`方法抛出了`DataAccessException`，实际上，`Spring`将所有的异常都封装成了`DataAccessException` 。`DataAccessException` 继承了`NestedRuntimeException`，而`NestedRuntimeException`又继承了`RuntimeException`。所以在`HibernateTemplate`中抛出的任何异常都会导致事务的回滚。

在`save`方法中，调用了`executeWithNativeSession`方法，方法的参数是一个匿名内部类，该匿名内部类实现了`HibernateCallback`接口。`executeWithNativeSession`方法正是调用了匿名内部类的`doInHibernate`方法，并向其传递了当前的`Session`对象，然后执行了`save`操作。

其实这是模板方法这一设计模式的使用，为了更好的理解，我来模拟整个过程。创建接口`MyHibernateCallback`和类`MyHibernateTemplate`。

```java
public interface MyHibernateCallback {

    void doInHibernate(Session session);
}

```

```java
public class MyHibernateTemplate {

    public void executeWithNativeSession(MyHibernateCallback callback) {
        Session session = null;
        try {
            session = getSession();
            session.beginTransaction();

            callback.doInHibernate(session);

            session.getTransaction().commit();
        } catch (Exception e) {
            session.getTransaction().rollback();
        } finally {
            if (session != null) {
                session.close();
            }
        }
    }

    public Session getSession() {
        return null;
    }

    public void save(final Object entity) {
        this.executeWithNativeSession(new MyHibernateCallback() {
            public void doInHibernate(Session session) {
                session.save(entity);
            }
        });
    }
}
```

将模板代码定义在`executeWithNativeSession`方法中，具体的业务逻辑通过参数`callback`传递进来，在执行`callback`的`doInHibernate`方法时，将当前的`Session`作为参数传递过去。不要在意此处的`getSession`方法，纯粹是为了说明。可以看到`MyHibernateTemplate` 的save方法与`HibernateTemplate`中的`save`方法已经非常类似了。


## 总结


理解了`Hibernate Template`的封装之后，使用起来非常方便。感觉比较惊艳的是`Spring`在实现`Hibernate Template`时的设计思想。自己对设计模式的了解还是太少，这部分的知识还需要补充。




---
title: 谈谈ThreadLocal
date: 2017-02-24 22:21:03
tags: Java
toc: true
---


[Java并发编程实战](https://book.douban.com/subject/10484692/) 一书在介绍`ThreadLocal`类时（第`3`章），书中有这么两段话，在我初次阅读时不知道如何去理解。

> ThreadLocal对象通常用于防止对可变的单实例变量（Singleton）或全局变量进行共享。

> 当某个频繁执行的操作需要一个临时变量，例如一个缓冲区，而同时又希望避免在每次执行时都重新分配该临时对象，就可以使用这项技术。例如，在Java 5.0之前，Integer.toString()方法使用ThreadLocal对象来保存一个12字节大小的缓冲区，用于对结果进行格式化，而不是使用共享的静态缓冲区（这需要锁机制）或者在每次调用时都分配一个新的缓冲区。

如果我们要在某种程度上理解这两句话，首先我们需要知道什么是`ThreadLocal`？更重要的是为什么要提出`ThreadLocal`这个概念？

我们知道，线程并不独立拥有用户空间，用户空间是归进程所有，为同一进程中的所有线程所共享的。所以，用户空间中的任何一个区域，只要有一个线程可以访问，那么同一进程中的所有其它的线程就都能访问。在这个意义上，整个用户空间都是（由同一进程中的）所有线程共享的，不存在只归一个线程使用的变量或数据结构。可是，一般而言，程序对变量或数据结构的访问都是按变量名访问的，经过编译/连接之后就是按地址访问，要是不知道一个变量的地址，实际上就无法正常和正确地加以访问。在这个意义上，则只归一个线程使用的变量或数据结构又是可能的。

注意`ThreadLocal`只是对全局量和静态变量才有意义。局部量存在于具体线程的堆栈上，而每个线程都有自己的堆栈，所以局部量本来就是“局部”于具体线程的。至于通过动态分配的缓冲区，则取决于保存着缓冲区指针的变量。如果缓冲区指针是全局量，那么同一进程中的所有线程都能访问这个缓冲区；而若是局部量，则别的线程自然就不得其门而入。

## synchronized

在介绍`ThreadLocal`之前，我们先通过一段代码来理解`synchronized`是如何实现在同一时刻只允许单个线程访问同步代码块的。

```java
public class SynchronizedTest {

    public synchronized void func1() {
    }

    public void func2() {
        synchronized (this) {
        }
    }   
  
}
```

将`SynchronizedTest`编译之后，`SynchronizedTest`中的两个同步方法的字节码（只截取需要的部分 ）如下。

```assembly
  public synchronized void func1();
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=0, locals=1, args_size=1
         0: return
      LineNumberTable:
        line 10: 0

  public void func2();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_1
         5: monitorexit
         6: goto          14
         9: astore_2
        10: aload_1
        11: monitorexit
        12: aload_2
        13: athrow
        14: return
```

`synchronized`关键字经过编译之后，会在同步代码块的前后分别形成`monitorenter`和`monitorexit`这两个字节码指令，例如`func2`方法对应的第`3`条和第`5`条字节码，这两个字节码都需要一个`reference`类型的参数来指明要锁定和解锁的对象。那这里提到的锁和对象之间的关系是什么？关于这个问题，可以参考我的另一篇文章 [Java对象内存布局](http://tinylcy.me/2016/11/30/Java%E5%AF%B9%E8%B1%A1%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80/)，在对象的`Mark Word`部分，存储着锁标志位，线程通过检查对象头的锁标志位，获知对象的锁状态，然后决定是获取锁还是进入阻塞状态。`synchronized`正是通过这个机制实现对共享资源的串行访问。

## 什么是ThreadLocal

关于`ThreadLocal`的概念，直接`从ThreadLocal`源码注释入手。

> 
This class provides thread-local variables. These variables differ from their normal counterparts in that each thread that accesses one (via its get or set method) has its own, independently initialized copy of the variable. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread (e.g., a user ID or Transaction ID).
> 

根据注释，我们认识到`ThreadLocal`可以为线程提供一个线程局部的值，既然该值是一个线程的局部变量，自然不存在线程同步的问题。但是注释又说到：*ThreadLocal instances are typically private static fields in classes*，这句话如何理解？如果一个变量是`static`的变量，那么它就是进程级别的全局变量，那不是意味着`ThreadLocal`是一个线程共享的变量吗？为了解决这个问题，我们需要阅读`ThreadLocal`的源码。

首先是`ThreadLocal`的`set`方法，`set`方法的作用是为当前线程设置一个`ThreadLocal`的值`value`，源码如下。

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

我们可以看到，`set`方法首先调用`getMap`方法从当前线程获取类型为`ThreadLocalMap`的对象`map`，如果`map`还没有创建，就通过`createMap`方法创建一个。然后`set`方法以当前的`ThreadLocal`对象为键，`value`为值，存储到当前线程的`ThreadLocalMap`对象中。`Thread`类中`ThreadLocalMap`变量的声明如下。

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

`ThreadLocal`的`get`方法用于获取与当前线程关联的`ThreadLocal`值，源码如下。

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }
```

`get`方法首先通过`getMap`方法获取当前线程的`ThreadLocalMap`对象，然后以`ThreadLocal`对象为键获取与该`ThreadLocal`对象关联的值。

从这两个方法我们知道，`ThreadLocal`对象的确是线程共享的，但是当线程向`ThreadLocal`设置值时，实际上是给当前线程维护的`ThreadLocalMap`设置了值。因此线程设置的值为线程私有，但是`ThreadLocal`对象为线程共享。

## 为什么这么设计

如果我们要设置线程本地的变量，我们只需要在方法内声明局部变量即可，为什么要通过`ThreadLocal`来设置？对于`ThreadLocal`的设计理念，我们通过`Linux/Unix`的`C`程序库`libc`的全局变量`errno`来理解。当系统调用从内核空间返回用户空间时，如果系统调用出错，那么便设置`errno`的值为一个负值，这样就不需要每次在函数内部定义局部变量。但是当多线程的概念和技术被提出后，这套机制就不再适用了，可以使用局部变量，但是不太可能去更改已有的代码了，比较好的解决方案是让每个线程都有自己的`errno`。实际上，现在的`C`库函数不是把出错代码写入全局量`errno`，而是通过一个函数`__errno_location()`获取一个地址，再把出错代码写入该地址，其意图就是让不同的线程使用不同的出错代码存储地点，而`errno`，现在一般已经变成了一个宏定义。

```c
#define errno (*__errno_location())
```

考虑另一个场景：我们现在需要设置一个线程局部变量，于是我们在方法内设置了一个局部变量，当我们需要把这个局部变量从一个方法传递到另一个方法，只需要将这个变量作为参数传递即可。假设`funcA`需要访问该变量，`funcZ`也需要访问该变量，但是`funcA`需要通过调用`funcB`，`funcC...funcY`才能调用`funcZ`，于是该变量需要被声明在所有的方法的签名中。为了避免这个麻烦，我们可以把这个变量设置为进程级别的全局变量，但是此时就需要我们控制线程同步了。于是，`ThreadLocal`就可以发挥作用了。

我们可以认为`ThreadLocal`存储了一个线程的上下文信息，线程通过访问`ThreadLocal`这个进程级别的变量实现了线程级别的访问。



## 参考

* [Java并发编程实战](https://book.douban.com/subject/10484692/)
* [深入理解Java虚拟机](https://book.douban.com/subject/24722612/)
* [漫谈兼容内核](http://www.longene.org/techdoc/0328131001224576926.html)
* [理解 ThreadLocal](http://github.thinkingbar.com/threadlocal/)
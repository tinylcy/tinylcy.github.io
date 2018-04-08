---
title: 'Why wait/notify must be called in a synchronized block'
---
在 Java 并发编程中，调用 Object wait/notify 方法的代码段必须要被包含在 synchronized 块中，接着即是耳熟能详的：调用 wait 方法时，先释放锁，然后线程进入阻塞状态，直至被 notify，然后重新尝试获得锁。看似一气呵成的一顿猛如虎的操作，其个中缘由到底是什么？有些我们看起来理所当然的东西，难道真的是理所当然的吗？本篇文章即谈谈，为什么调用 wait/notify 方法的代码段必须要被包含在 synchronized 块中。

先从[官方文档](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html)入手，官方文档展示了一段代码告诉读者们 wait 方法应如何使用。如果当前线程（即调用 obj.wait 方法的线程）不是 monitor 的 owner，抛出 IllegalMonitorStateException。synchronized 关键字在经过编译后，会在同步代码块的前后生成 monitorenter 和 monitorexit 这两个字节码指令，它们都需要一个 reference 类型的参数指明要锁定和解锁的对象，这也是「monitor」的由来。

```java
synchronized (obj) {
	while (<condition does not hold>)
		obj.wait(timeout);
    ... // Perform action appropriate to condition
}
```

假设要利用 wait/notify 方法实现 BlockingQueue 数据结构，很快的写下了如下代码。

```java
public class BlockingQueue {

    private Queue<String> buffer = new LinkedList<String>();

    public void put(String data) {
        buffer.add(data);
        notify();
    }

    public String get() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait();
        }
        return buffer.remove();
    }
}
```

注意！BlockingQueue 在调用 wait/notify 方法时并没有使用 synchronized 关键字，思考会发生什么问题，考虑以下场景。

* consumer 线程调用 get 方法，反复测试 buffer.isEmpty( )。
* 在 consumer 线程调用 wait 方法之前，producer 线程调用了 put 方法将 data 添加至 buffer，并调动了 notify 方法。
* 此时 consumer 调用了 wait 方法，但是很不幸 consumer 错失了来自 producer 的 notify。
* 更不幸的，不再有 producer 往 buffer 添加新的 data，consumer 会永远阻塞于 wait 方法，虽然实际上此时 buffer 中存在一个 data。

产生上述场景所描述问题的关键原因在于 put & notify 和 get & wait 不被原子性（atomic）的调用。很自然的 synchronized 可以发挥保证原子性的作用。但是注意！为了保证原子性，synchronized 完全可以使用任意一个 Object，为什么必须和 wait/notify 方法一致的 Object？其实很简单，结合以下代码：如果使用 BlockingQueue.class 作为锁，当 wait 方法释放 this 锁并进入阻塞状态后，get 方法依旧拥有着 BlockingQueue.class 锁，producer 永远会被阻塞于调用 put 方法，形成死锁。

```java
public class BlockingQueue {

    private Queue<String> buffer = new LinkedList<String>();

    public void put(String data) {
        synchronized (BlockingQueue.class) {
            buffer.add(data);
            notify();
        }
    }

    public String get() throws InterruptedException {
        synchronized (BlockingQueue.class) {
            while (buffer.isEmpty()) {
                wait();
            }
            return buffer.remove();
        }
    }
}
```

EOF.
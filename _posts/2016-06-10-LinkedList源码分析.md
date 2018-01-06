---
title: LinkedList源码分析
date: 2016-06-10 21:48:56
tags: [JDK,Java]
toc: true
---

本文涉及的`JDK`源码版本为`1.7.0_79`。

## LinkedList类定义

`LinkedList`是基于双向链表实现的，`LinkedList`除了被当做数组来使用，还可以作为栈、队列来使用。由于`LinkedList`内部采用链表的形式存储元素，因此随机访问会比较慢，但是插入、删除元素比较快。

和`ArrayList`一样，`LinkedList`也是非线程安全的，只有在单线程下才可以使用。为了防止非同步访问，可以采用如下方式创建`LinkedList`。
```java
List list= Collections.synchronizedList(new LinkedList());
```

`LinkedList`的类定义如下。

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```

* `LinkedList`继承了抽象类`AbstractSequentialList`。
* `LinkedList`实现了`Serializable`接口，可以被序列化，能通过序列化传输。
* `LinkedList`实现了`Cloneable`接口，可以被克隆。
* `LinkedList`实现了`Deque`接口，`Deque`是双端队列接口，提供了类似`push`、`pop`和`peek`等适用于栈和队列的方法。

`LinkedList`中定义了内部类`Node`来代表链表中的节点。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
`Node`节点 一共有三个属性：`item`代表节点值，`prev`代表节点的前一个节点，`next`代表节点的后一个节点。

## LinkedList类属性

`LinkedList`一共包含三个类成员变量，`size`代表链表含有节点的个数，`first`指向链表的首节点，`last`指向链表的尾节点。

```java
transient int size = 0;

transient Node<E> first;

transient Node<E> last;
```

## LinkedList类构造函数

`LinkedList`含有两个构造函数，分别是默认构造函数和带有一个集合对象的构造函数。
```java
public LinkedList() {
}

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
默认构造函数构建了一个空链表，带参数的构造函数将结合对象传入，调用`addAll`方法初始化链表。既然这里涉及了`addAll`方法，所以继续追踪下去，`addAll`方法如下所示。

```java
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
```

`addAll`方法调用了另一个`addAll`方法，并将链表长度`size`作为参数传入，该`addAll`方法如下所示。

```java
public boolean addAll(int index, Collection<? extends E> c) {
    checkPositionIndex(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    if (numNew == 0)
        return false;

    Node<E> pred, succ;
    if (index == size) {
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }

    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        pred = newNode;
    }

    if (succ == null) {
        last = pred;
    } else {
        pred.next = succ;
        succ.prev = pred;
    }

    size += numNew;
    modCount++;
    return true;
}
```
对该`addAll`方法进行分析：当调用`new LinkedList(Collection c)`时，此时`size = 0`, `first = last = null`，且此时`index = size = 0`。`addAll`方法内创建了两个`Node`节点的引用`pred`和`succ`，`pred`指向链表构建过程中节点插入位置的前一个节点。当创建新的`LinkedList`时，`succ = null`，当向已经存在的`LinkedList`中插入元素时，`succ`指向了以当前链表的第`index`个结点，也就是指向插入位置的后一个结点（此处调用了`node(index)`方法，后面会介绍）。将`Collection`转化为`Array`,进行遍历，将每个元素插入到双向链表。对于一个插入到链表的`Node`，通过`Node`的构造函数维护`Node`的和`pred`的关系，即让`Node`的`prev`指向`pred`且让`pred`的`next`指向`Node`。当所有的元素都插入到链表后，使`pred`的`next`指向`succ`且让`succ`的`prev`指向`pred`。最后，更新`size`的大小。

##  LinkedList类核心方法

`linkFirst`、`linkLast`和`linkBefore`，以及未显示的`unlinkFirst`、`unlinkLast`和`unlink`方法是用于实现一系列的`add`和`remove`方法，是`LinkedList`的实现基础。这些方法如果学习过数据结构的话很好理解。`linkFisrst`将元素`e`插入到当前链表的首节点之前，并作为新的首节点。同理，`linkLast`将元素插入到当前链表末尾，并作为新的尾节点。`linkBefore`是将元素插入到指定的某个节点之前，其操作同向双向链表插入一个节点一致。

```java
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}

void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

接下来是一系列`get`方法、`remove`方法和`add`方法，这些方法都是借助于上面提到的`linkXX`、`unlinkXX`方法实现。另外，在分析``LinkedList`构造函数时，曾涉及`addAll(Collcetion c)`方法。
```java
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}

public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}

public void addFirst(E e) {
    linkFirst(e);
}

public void addLast(E e) {
    linkLast(e);
}

public boolean add(E e) {
    linkLast(e);
    return true;
}

public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```
注意到在`addAll(Collection c)`方法内，调用过`node(int index)`方法，`node`方法返回双向链表的第`index`个节点，`node`方法定义如下。这儿用到了一个小策略，当`index`小于链表长度的一半，那么从前往后遍历，否则，从后往前遍历，在`O(n/2)`时间内可以找到节点。
```java
Node<E> node(int index) {
    // assert isElementIndex(index);

    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

## 问题

在`addAll(int index, Collection<? extends E> c)`方法内，先将`Collection`转化为`Array`，然后再遍历，这是出于什么考虑？


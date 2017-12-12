---
title: Why String is Immutable or Final in Java
date: 2016-05-15 01:50:54
tags: Java
toc: true
---

## 引入问题

`Thinking in Java`中写道：“`String`对象是不可变的，`String`类中每一个看起来会修改`String`值的方法，实际上都是创建了一个全新的`String`对象，以包含修改后的字符串内容。而最初的`String`对象则丝毫未动”。通过查看源代码，了解到`String`类被设计成`final`类型，那么`String`类被设计成`final`类型是出于哪些考虑？

## final关键字

首先需要理解`Java`中`final`的含义，通常`final`指的是"这是无法改变的"。不想做改变可能出于两种原因：设计或效率。而许多编程语言都有某种方法来想编译器告知一块数据是恒定不变的。在`Java`中，可能使用到`final`的情况有三种：数据、方法和类。

有时数据的恒定不变会很有用，比如：
* 一个永不改变的编译时常量。
* 一个在运行时初始化的值，而你不希望它被改变。


对于编译时常量这种情况，编译器可以将该常量代入任何可能用到它的计算式中，也就是说，可以在编译时执行计算式，这减轻了一些运行时的负担。

使用`final`方法的原因有两个。第一个原因是把方法锁定，以防任何继承类修改它的定义。这是出于设计的考虑：确保在集成中使用方法行为保持不变，并且不会被覆盖。过去建议使用`final`方法的第二个原因是效率。在`Java`的早期实现中，如果将一个方法指明为`final`,就是同意编译器将针对该方法的所有调用都转为内嵌调用。这将消除方法调用的开销。不过不要陷入对仓促优化性能的强烈渴望中，如果系统速度很慢，企图使用`final`来解决问题是难以奏效的。

如果使用`final`来修饰类的话，那么这个类是不能被继承的。

## String类定义

查看`String`类在`JDK1.7`源码中的定义(部分)。

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {

    /** The value is used for character storage. */
    private final char value[];
}
```

可以看出`String`类正是被`final`修饰，表示`String`类不能被其它类继承。另外，可以看出在创建`String`对象的时候，`String`对象还是使用字符数组来存储字符串的。并且，该字符数组也是被`final`修饰，因此`String`的内容一旦被初始化后是不能被改变的。 查看`String`类的官方注释。

```java
/**
* The String class represents character strings. All
* string literals in Java programs, such as "abc", are
* implemented as instances of this class.
* 
* Strings are constant; their values cannot be changed after they
* are created. String buffers support mutable strings.
* Because String objects are immutable they can be shared. 
*/
```

字符串缓冲区支持可变的字符串，因为缓冲区内的不可变字符串可以被共享，实际上这指的是对象的引用发生了改变。

而关于`String`类为什么被声明为`final`，在[`StackOverflow`](http://stackoverflow.com/questions/2068804/why-is-string-class-declared-final-in-java)上给出的解释是这样的。 

> - Security: the system can hand out sensitive bits of read-only information without worrying that they will be altered.
> - Performance: immutable data is very useful in making things thread-safe.

`String`类的基本约定中，最重要的一条是`immutable`，将`String`声明为`final`避免了`String`的子类打破这一约定。假设现在`String`类并没有被`final`修饰，那么我定义`SubString`类，继承自`String`类。

```java
public class SubString extends String {
    private char[] chars;

    public SubString(String string) {
        chars = string.toCharArray();
    }

    public void setChar(int index, char ch) {
        chars[index] = ch;
    }

    public void toString() {
        return new String(chars);
    }
}
```

这一瞬间就改变了`String`的`immutable`的约定，`String`对象创建后仍可以修改`String`对象的内容。


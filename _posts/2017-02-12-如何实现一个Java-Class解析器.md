---
title: 如何实现一个Java Class解析器
date: 2017-02-12 23:26:08
tags: [Java,JVM]
toc: true
---

最近在写一个私人项目，名字叫做`ClassAnalyzer`，`ClassAnalyzer`的目的是能让我们对`Java Class`文件的设计与结构能够有一个深入的理解。主体框架与基本功能已经完成，还有一些细节功能日后再增加。实际上`JDK`已经提供了命令行工具`javap`来反编译`Class`文件，但本篇文章将阐明我实现解析器的思路。

## Class文件

作为类或者接口信息的载体，每个`Class`文件都完整的定义了一个类。为了使`Java`程序可以“编写一次，处处运行”，[Java虚拟机规范](http://files.cnblogs.com/files/zhuYears/Java%E8%99%9A%E6%8B%9F%E6%9C%BA%E8%A7%84%E8%8C%83%EF%BC%88JavaSE7%EF%BC%89.pdf)对`Class`文件进行了严格的规定。构成`Class`文件的基本数据单位是字节，这些字节之间不存在任何分隔符，这使得整个`Class`文件中存储的内容几乎全部是程序运行的必要数据，单个字节无法表示的数据由多个连续的字节来表示。

根据`Java`虚拟机规范，`Class`文件采用一种类似于`C`语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：无符号数和表。`Java`虚拟机规范定义了`u1`、`u2`、`u4`和`u8`来分别表示`1`个字节、`2`个字节、`4`个字节和`8`个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者是字符串。表是由多个无符号数或者其它表作为数据项构成的复合数据类型，表用于描述有层次关系的复合结构的数据，因此整个`Class`文件本质上就是一张表。在`ClassAnalyzer`中，`byte`、`short`、`int`和`long`分别对应`u1`、`u2`、`u4`和`u8`数据类型，`Class`文件被描述为如下`Java`类。

```java
public class ClassFile {

    public U4 magic;                            // magic
    public U2 minorVersion;                     // minor_version
    public U2 majorVersion;                     // major_version
    public U2 constantPoolCount;                // constant_pool_count
    public ConstantPoolInfo[] cpInfo;           // cp_info
    public U2 accessFlags;                      // access_flags
    public U2 thisClass;                        // this_class
    public U2 superClass;                       // super_class
    public U2 interfacesCount;                  // interfaces_count
    public U2[] interfaces;                     // interfaces
    public U2 fieldsCount;                      // fields_count
    public FieldInfo[] fields;                  // fields
    public U2 methodsCount;                     // methods_count
    public MethodInfo[] methods;                // methods
    public U2 attributesCount;                  // attributes_count
    public BasicAttributeInfo[] attributes;     // attributes

}
```

## 如何解析

组成`Class`文件的各个数据项中，例如魔数、`Class`文件的版本、访问标志、类索引和父类索引等数据项，它们在每个`Class`文件中都占用固定数量的字节，在解析时只需要读取相应数量的字节。除此之外，需要灵活处理的主要包括`4`部分：常量池、字段表集合、方法表集合和属性表集合。字段和方法都可以具备自己的属性，`Class`本身也有相应的属性，因此，在解析字段表集合和方法表集合的同时也包含了属性表集合的解析。

常量池占据了`Class`文件很大一部分的数据，用于存储所有的常量信息，包括数字和字符串常量、类名、接口名、字段名和方法名等。`Java`虚拟机规范定义了多种常量类型，每一种常量类型都有自己的结构。常量池本身是一个表，在解析时有几点需要注意。

* 每个常量类型都通过一个`u1`类型的`tag`来标识。

* 表头给出的常量池大小（`constantPoolCount`）比实际大`1`，例如，如果`constantPoolCount`等于`47`，那么常量池中有`46`项常量。
* 常量池的索引范围从`1`开始，例如，如果`constantPoolCount`等于`47`，那么常量池的索引范围为`1 ~ 46`。设计者将第`0`项空出来的目的是用于表达“不引用任何一个常量池项目”。
* 如果一个`CONSTANT_Long_info`或`CONSTANT_Double_info`结构的项在常量池中的索引为`n`，则常量池中下一个有效的项的索引为`n+2`，此时常量池中索引为`n+1`的项有效但必须被认为不可用。
* `CONSTANT_Utf8_info`型常量的结构中包含一个`u1`类型的`tag`、一个`u2`类型的`length`和由`length`个`u1`类型组成的`bytes`，这`length`字节的连续数据是一个使用`MUTF-8`（`Modified UTF-8）`编码的字符串。`MUTF-8`与`UTF-8`并不兼容，主要区别有两点：一是`null`字符会被编码成`2`字节（`0xC0`和`0x80`）；二是补充字符是按照`UTF-16`拆分为代理对分别编码的，相关细节可以看[这里（变种UTF-8）](https://zh.wikipedia.org/wiki/UTF-8)。

属性表用于描述某些场景专有的信息，`Class`文件、字段表和方法表都有相应的属性表集合。`Java`虚拟机规范定义了多种属性，`ClassAnalyzer`目前实现了对常用属性的解析。与常量类型的数据项不同，属性并没有一个`tag`来标识属性的类型，但是每个属性都包含有一个`u2`类型的`attribute_name_index`，`attribute_name_index`指向常量池中的一个`CONSTANT_Utf8_info`类型的常量，该常量包含着属性的名称。在解析属性时，`ClassAnalyzer`正是通过`attribute_name_index`指向的常量对应的属性名称来得知属性的类型。

字段表用于描述类或者接口中声明的变量，字段包括类级变量以及实例级变量。字段表的结构包含一个`u2`类型的`access_flags`、一个`u2`类型的`name_index`、一个`u2`类型的`descriptor_index`、一个`u2`类型的`attributes_count`和`attributes_count`个`attribute_info`类型的`attributes`。我们已经介绍了属性表的解析，`attributes`的解析方式与属性表的解析方式一致。

`Class`的文件方法表采用了和字段表相同的存储格式，只是`access_flags`对应的含义有所不同。方法表包含着一个重要的属性：`Code`属性。`Code`属性存储了`Java`代码编译成的字节码指令，在`ClassAnalyzer`中，`Code`对应的`Java`类如下所示（仅列出了类属性）。

```java
public class Code extends BasicAttributeInfo {

    private short maxStack;
    private short maxLocals;
    private long codeLength;
    private byte[] code;
    private short exceptionTableLength;
    private ExceptionInfo[] exceptionTable;
    private short attributesCount;
    private BasicAttributeInfo[] attributes;
    ...

    private class ExceptionInfo {
        public short startPc;
        public short endPc;
        public short handlerPc;
        public short catchType;
      	...
    }
}
```

在`Code`属性中，`codeLength`和`code`分别用于存储字节码长度和字节码指令，每条指令即一个字节（`u1`类型）。在虚拟机执行时，通过读取`code`中的一个个字节码，并将字节码翻译成相应的指令。另外，虽然`codeLength`是一个`u4`类型的值，但是实际上一个方法不允许超过`65535`条字节码指令。

## 代码实现

`ClassAnalyzer`的源码已放在了[GitHub](https://github.com/tinylcy/ClassAnalyzer)上。在`ClassAnalyzer`的[README](https://github.com/tinylcy/ClassAnalyzer/blob/master/README.md)中，我以一个类的`Class`文件为例，对该`Class`文件的每个字节进行了分析，希望对大家的理解有所帮助。

## 参考

* [深入理解Java虚拟机](https://book.douban.com/subject/24722612/)
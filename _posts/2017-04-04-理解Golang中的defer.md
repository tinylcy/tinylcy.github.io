---
title: 理解Golang中的defer
date: 2017-04-04 21:01:21
tags: Golang
---

关键字`defer`用于实现延迟调用，根据`Golang`官方的定义：*A defer statement defers the execution of a function until the surrounding function returns.* 。但是，当返回值与`defer`相互关联时，如果没有正确理解`defer`与`return`真正的执行顺序，那么容易出现一些不可描述的现象。

我们先运行如下代码，根据运行结果来理解`defer`，在查看运行结果之前，不妨先想想`main`函数的输出是什么。

```go
package main

import "fmt"

func main() {
	fmt.Println(funcA())
	fmt.Println(funcB())
}

func funcA() int {
	var i int
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return i
}

func funcB() (i int) {
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return i
}
```

运行结果如下。

```go
defer1: 1
defer2: 2
0
defer1: 1
defer2: 2
2
```

根据运行结果，我们可以看到，`defer`语句的执行顺序以及打印出来的变量`i`的值是意料之内的，区别在于函数的返回值，而`funcA`和`funcB`这两个函数唯一的区别则是函数的返回值有没有被命名，因此导致两个函数返回值不同的原因也应该和返回值是否命名有关。

在计算机科学中，我相信很多原理性的东西都是可以相互解释的。在本篇文章中，我决定用`Java`中的`try-catch-finally`来解释`defer`的运行机制。首先，先看看如下`Java`代码片段，并考虑返回值有哪几种情况。

```java
    public int func() {
        int x;
        try {
            x = 1;
            return x;
        } catch (Exception e) {
            x = 2;
            return x;
        } finally {
            x = 3;
        }
    }
```

如果我们对`Java`熟悉，那么应该知道，无论在`try`块中是否出现异常，`finally`块中的语句是一定要执行的，但是函数的返回值只可能是`1`或者`2`，绝对不会是`3`。我们通过查看这段`Java`代码对应的字节码指令来理解这一点。

```java
  public int func();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=5, args_size=1
         0: iconst_1
         1: istore_1
         2: iload_1
         3: istore_2
         4: iconst_3
         5: istore_1
         6: iload_2
         7: ireturn
```

这段字节码对应于`try-finally`这条执行轨迹：`iconst_1`将常量`1`压入操作数栈；`istore_1`将栈顶元素弹出并存储于局部变量表`Slot1`处；`iload_1`将局部变量表`Slot1`处的元素压入操作数栈；`istore_2`将操作数栈顶元素弹出并存储于局部变量表`Slot2`处；`iconst_3`将常量`3`压入操作数栈；`istore_1`将操作数栈顶元素弹出并存储于局部变量表`Slot1`处；`iload_2`将局部变量表`Slot2`处的元素压入操作数栈；`ireturn`返回栈顶元素。

第`0`条指令和第`1`条指令实现了`x = 1`，并且`x`的值存储于局部变量表的`Slot1`处；第`2`条指令和第`3`条指令将`x`的值拷贝了一份，并存储在局部变量表的`Slot2`处。第`4`条指令和第`5`条指令实现了`x = 3`；第`6`条指令和第`7`条指令将存储于局部变量表`Slot2`处的`x`的拷贝值返回，由于`Slot2`中的值是执行`finally`块中语句之前`x`的值，因此返回值等于`2`。

基于以上解释，我们再来重新理解`defer`。对于函数`funcA`，当执行至`return`时，变量`i`的值等于`0`，与`Java`类似，`Golang`会将返回值的拷贝值（即变量`i`的值）存储于内存中的某个位置`pos`（对应于局部变量表的`Slot2`处），然后执行`defer`语句，当`defer`语句执行完后，尽管变量`i`的值已增加至`2`，但是返回值依赖于地址`pos`处的值，因此`funcA`返回`0`；对于函数`funcB`，由于`funcB`已经命名了函数的返回值为变量`i`，这意味着函数的返回值的地址即为变量`i`的地址。当执行至`return`时，尽管变量`i`的值为`0`，但是紧接着的`defer`语句使得变量`i`的值增加至`2`，由于`funcA`的返回值的地址为变量`i`的地址，因此`funcB`最后的返回值为`2`。

为了验证我们的解释，运行如下代码。

```go
func main() {
	fmt.Println(*(funcC()))
}

func funcC() *int {
	var i int
	defer func() {
		i++
		fmt.Println("defer2:", i)
	}()
	defer func() {
		i++
		fmt.Println("defer1:", i)
	}()
	return &i
}
```

尽管函数`funcC`的返回值并没有提前声明，但是`funcC`的返回值仍为`2`，这是因为`funcC`的返回值是变量`i`的内存地址，当执行到`return`语句时，变量`i`的内存地址值的拷贝会被存储于内存的某个位置，而该位置的值即是最后的返回值，由于变量`i`的地址在整个过程并未被修改，因此通过地址值的拷贝值我们依旧可以观察到`defer`语句对变量`i`的操作。



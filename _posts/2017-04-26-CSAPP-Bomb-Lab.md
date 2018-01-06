---
title: 'CSAPP: Bomb Lab'
date: 2017-04-26 19:14:51
tags: [CSAPP,Assembly]
toc: true
comments: true
---


`Bomb Lab`是`CS:APP`一书中第二个[实验](http://csapp.cs.cmu.edu/3e/bomblab.pdf)，实验中的`bomb`实际上是一个程序的二进制文件，该程序由一系列`phase`组成，每个`phase`需要我们输入一个字符串，然后该程序会进行校验，如果输入的字符串不满足拆弹要求，那么就会打印`BOOM!!!`。

<!--more-->

完成整个实验的思路是通过`objdump`对`bome`进行反编译（`objdump -d bomb > bomb.txt`），获取所有的汇编代码。提取每个阶段对应的代码并借助`gdb`进行分析，逐一拆弹。

## Phase 1
`phase_1`对应的代码如下所示。

```assembly
0000000000400ee0 <phase_1>:
  400ee0: 48 83 ec 08           sub    $0x8,%rsp
  400ee4: be 00 24 40 00        mov    $0x402400,%esi
  400ee9: e8 4a 04 00 00        callq  401338 <strings_not_equal>
  400eee: 85 c0                 test   %eax,%eax
  400ef0: 74 05                 je     400ef7 <phase_1+0x17>
  400ef2: e8 43 05 00 00        callq  40143a <explode_bomb>
  400ef7: 48 83 c4 08           add    $0x8,%rsp
  400efb: c3                    retq  
```

由`callq`指令可以得知，`phase_1`调用了`strings_not_equal`，并将返回值存储于`%eax`中，`test`指令计算`%eax`的值是否等于`0`，`je`指令决定是否跳转，若`%eax`的值等于`0`，则跳转至`0x400ef7`处，也就是安全区域，拆弹成功，否则不跳转，即执行`explode_bomb`，拆弹失败。通过以上分析，可以得知`phase_1`的关键在于控制`strings_not_equal`的返回值。

在执行`callq strings_not_equal`指令之前，`mov $0x402400,%esi`将常数`0x402400`传递至`%esi`，根据`x86-64`寄存器使用规范，`%esi`的值是`strings_not_equal`函数的第二个参数，而第一个参数则是我们输入的值。因此，问题的关键在于如何得到`%esi`的值。

利用`gdb`对`bomb`进行调试，并在`phase_1`处设置断点，通过`disassemble`对当前函数进行反编译，使用`stepi`对指令进行单步调试，运行至`phase_1`第一条指令的初始状态如下图所示。

![](/img/img-2017-04-26-Image 1.png)

在进入`phase_1`之前，`read_line`函数从终端读取输入（可从`bomb.c`得知该信息，我输入的是`tinylcy`）。接着，通过`stepi`单步执行指令，直至指令`mov $0x402400,%esi`执行完毕，如下图所示。

![](/img/img-2017-04-26-Image 2.png)

此时，`strings_not_equal`函数的第二个参数已经存储于`%esi`，通过`print`打印`%esi`的值，可以得知`%esi`存储的是一个内存地址，这是因为参数类型是字符串类型，因此寄存器存储的是内存地址，而非确切的字符串值，使用`x`打印位于地址`%esi`处的内容，如下图所示。

![](/img/img-2017-04-26-Image 3.png)

由此，我们得到了拆除`pahse_1`炸弹所需要的字符串。

## Phase 2

`phase_2`对应的代码如下所示。

```assembly
0000000000400efc <phase_2>:
  400efc: 55                    push   %rbp
  400efd: 53                    push   %rbx
  400efe: 48 83 ec 28           sub    $0x28,%rsp
  400f02: 48 89 e6              mov    %rsp,%rsi
  400f05: e8 52 05 00 00        callq  40145c <read_six_numbers>
  400f0a: 83 3c 24 01           cmpl   $0x1,(%rsp)
  400f0e: 74 20                 je     400f30 <phase_2+0x34>
  400f10: e8 25 05 00 00        callq  40143a <explode_bomb>
  400f15: eb 19                 jmp    400f30 <phase_2+0x34>
  400f17: 8b 43 fc              mov    -0x4(%rbx),%eax
  400f1a: 01 c0                 add    %eax,%eax
  400f1c: 39 03                 cmp    %eax,(%rbx)
  400f1e: 74 05                 je     400f25 <phase_2+0x29>
  400f20: e8 15 05 00 00        callq  40143a <explode_bomb>
  400f25: 48 83 c3 04           add    $0x4,%rbx
  400f29: 48 39 eb              cmp    %rbp,%rbx
  400f2c: 75 e9                 jne    400f17 <phase_2+0x1b>
  400f2e: eb 0c                 jmp    400f3c <phase_2+0x40>
  400f30: 48 8d 5c 24 04        lea    0x4(%rsp),%rbx
  400f35: 48 8d 6c 24 18        lea    0x18(%rsp),%rbp
  400f3a: eb db                 jmp    400f17 <phase_2+0x1b>
  400f3c: 48 83 c4 28           add    $0x28,%rsp
  400f40: 5b                    pop    %rbx
  400f41: 5d                    pop    %rbp
  400f42: c3                    retq   
```

由`callq read_six_numbers`可知，`phase_2`调用了`read_six_numbers`函数并读取了`6`个数值。根据`cmpl $0x1,(%rsp)`可知，若地址`%rsp`处的值等于`1`，那么进入安全区域，否则就会引爆炸弹。由此可以得知，我们输入的第`1`个数字应该为`1`。然后`phase_2`跳转至`0x400f30`处执行指令`lea 0x4(%rsp),%rbx`，该指令的效果是将`%rsp`的值加`4`并存储于`%rbx`，这意味着`%rbx`的值实际上是下一个数的地址值。`lea 0x18(%rsp),%rbp`用于控制循环的次数，`6`个整型共占用`24`字节，恰好等于`0x18`。接着`phase_2`跳转至`0x400f17`处执行`mov -0x4(%rbx),%eax`指令，该指令的效果是使`%eax`的值等于`M[%rbx-4]`的值，即`M[%rsp]`的值，也就是第`1`个数的值。`add %eax,%eax`使`%eax`的值扩大为原来的`2`倍，`cmp %eax,(%rbx)`将下一个数的值与`%eax`的值作比较，若相等则跳转至安全区域`0x400f25`，否则拆弹失败。`0x400f25`处的指令为`add $0x4,%rbx`，该指令的效果是使`%rbx`存储下一个数的地址，与`%rbp`的值比较并在不相等的情况下跳转至`0x400f17`处循环执行指令。

综上，`phase_2`通过`%rbx`来获取输入的`6`个数字，通过`%eax`来控制比较的数值大小，`%eax`初始化为第`1`个数字的值，并在每次循环后增长至原来的`2`倍，一共`6`次循环。所以`phase_2`的解为`1 2 4 8 16 32`。

## Phase 3

`phase_3`对应的代码如下所示。

```assembly
0000000000400f43 <phase_3>:
  400f43: 48 83 ec 18           sub    $0x18,%rsp
  400f47: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  400f4c: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  400f51: be cf 25 40 00        mov    $0x4025cf,%esi
  400f56: b8 00 00 00 00        mov    $0x0,%eax
  400f5b: e8 90 fc ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  400f60: 83 f8 01              cmp    $0x1,%eax
  400f63: 7f 05                 jg     400f6a <phase_3+0x27>
  400f65: e8 d0 04 00 00        callq  40143a <explode_bomb>
  400f6a: 83 7c 24 08 07        cmpl   $0x7,0x8(%rsp)
  400f6f: 77 3c                 ja     400fad <phase_3+0x6a>
  400f71: 8b 44 24 08           mov    0x8(%rsp),%eax
  400f75: ff 24 c5 70 24 40 00  jmpq   *0x402470(,%rax,8)
  400f7c: b8 cf 00 00 00        mov    $0xcf,%eax
  400f81: eb 3b                 jmp    400fbe <phase_3+0x7b>
  400f83: b8 c3 02 00 00        mov    $0x2c3,%eax
  400f88: eb 34                 jmp    400fbe <phase_3+0x7b>
  400f8a: b8 00 01 00 00        mov    $0x100,%eax
  400f8f: eb 2d                 jmp    400fbe <phase_3+0x7b>
  400f91: b8 85 01 00 00        mov    $0x185,%eax
  400f96: eb 26                 jmp    400fbe <phase_3+0x7b>
  400f98: b8 ce 00 00 00        mov    $0xce,%eax
  400f9d: eb 1f                 jmp    400fbe <phase_3+0x7b>
  400f9f: b8 aa 02 00 00        mov    $0x2aa,%eax
  400fa4: eb 18                 jmp    400fbe <phase_3+0x7b>
  400fa6: b8 47 01 00 00        mov    $0x147,%eax
  400fab: eb 11                 jmp    400fbe <phase_3+0x7b>
  400fad: e8 88 04 00 00        callq  40143a <explode_bomb>
  400fb2: b8 00 00 00 00        mov    $0x0,%eax
  400fb7: eb 05                 jmp    400fbe <phase_3+0x7b>
  400fb9: b8 37 01 00 00        mov    $0x137,%eax
  400fbe: 3b 44 24 0c           cmp    0xc(%rsp),%eax
  400fc2: 74 05                 je     400fc9 <phase_3+0x86>
  400fc4: e8 71 04 00 00        callq  40143a <explode_bomb>
  400fc9: 48 83 c4 18           add    $0x18,%rsp
  400fcd: c3                    retq   
```

前`3`条指令非常普通，并不能吸引我们的注意，能够吸引我们的是`mov $0x4025cf,%esi`这条指令，常数`0x4025cf`应该是个内存地址，打印该内存地址的值，如下图所示。

![](/img/img-2017-04-26-Image 4.png)

很容易把`%d %d`与`callq  400bf0 <__isoc99_sscanf@plt>`这条指令联系起来，由此我们基本知道，`phase_3`会输入`2`个值。`scanf`函数的返回值存储于`%eax`，该值代表输入值的个数，`cmp $0x1,%eax`将`%eax`的值与`1`做比较，如果输入值的个数大于`1`，跳转至安全区域，即指令`cmpl $0x7,0x8(%rsp)`处，否则拆弹失败。`cmpl   $0x7,0x8(%rsp)`将输入的第一个数(以`6`为例)与`7`作比较，如果大于`7`，那么拆弹失败，否则执行指令`mov 0x8(%rsp),%eax`，该指令将第一个数存储于`%eax`中。接着执行指令`jmpq *0x402470(,%rax,8)`，该指令是一个间接跳转指令，通过`gdb`得到执行该指令后的跳转位置，如下图所示。

![](/img/img-2017-04-26-Image 5.png)

`mov $0x2aa,%eax`指令将常数`0x2aa`移至`%eax`，然后执行`jmp 0x400fbe <phase_3+123>`跳转至`0x400fbe`处执行指令`cmp 0xc(%rsp),%eax`，该指令将第二个数与`%eax`做比较，若相等，安全退出，拆弹成功，否则拆弹失败。`0x2aa`的十进制值为`682`，因此`phase_3`输入的两个数应为`6`，`682`。

注意，第一个数并不一定是`6`，只要小于`7`即可。当第一个数的取值改变，那么在获取第二个数时会跳转到不同的分支，因此会得到不同的值。例如当第一个数等于`5`，那么第二个数应为`206`。

## Phase 4

`phase_4`对应的代码如下所示。

```assembly
000000000040100c <phase_4>:
  40100c: 48 83 ec 18           sub    $0x18,%rsp
  401010: 48 8d 4c 24 0c        lea    0xc(%rsp),%rcx
  401015: 48 8d 54 24 08        lea    0x8(%rsp),%rdx
  40101a: be cf 25 40 00        mov    $0x4025cf,%esi
  40101f: b8 00 00 00 00        mov    $0x0,%eax
  401024: e8 c7 fb ff ff        callq  400bf0 <__isoc99_sscanf@plt>
  401029: 83 f8 02              cmp    $0x2,%eax
  40102c: 75 07                 jne    401035 <phase_4+0x29>
  40102e: 83 7c 24 08 0e        cmpl   $0xe,0x8(%rsp)
  401033: 76 05                 jbe    40103a <phase_4+0x2e>
  401035: e8 00 04 00 00        callq  40143a <explode_bomb>
  40103a: ba 0e 00 00 00        mov    $0xe,%edx
  40103f: be 00 00 00 00        mov    $0x0,%esi
  401044: 8b 7c 24 08           mov    0x8(%rsp),%edi
  401048: e8 81 ff ff ff        callq  400fce <func4>
  40104d: 85 c0                 test   %eax,%eax
  40104f: 75 07                 jne    401058 <phase_4+0x4c>
  401051: 83 7c 24 0c 00        cmpl   $0x0,0xc(%rsp)
  401056: 74 05                 je     40105d <phase_4+0x51>
  401058: e8 dd 03 00 00        callq  40143a <explode_bomb>
  40105d: 48 83 c4 18           add    $0x18,%rsp
  401061: c3                    retq   
```

和`phase_3`类似，首先将内存地址为`0x4025cf`的内容打印出来，如下图所示。

![](/img/img-2017-04-26-Image 6.png)

由此得知，`phase_4`的输入应该为`2`个整数。在执行`callq  400bf0 <__isoc99_sscanf@plt>`指令后，返回值存储于`%eax`，然后判断`%eax`的值是否等于`2`，若不等于则会引爆炸弹，否则执行指令`cmpl $0xe,0x8(%rsp)`，该指令将输入的第一个数与常数`0xe`做比较，根据`jbe 40103a <phase_4+0x2e>`，如果输入的第一个数大于`0xe`，那么拆弹失败，否则跳转到`0x40103a`处执行`mov $0xe,%edx`指令。接下来可以看到`phase_4`调用了函数`func4`，而`mov $0xe,%edx`、`mov $0x0,%esi`和`mov 0x8(%rsp),%edi`这三条指令用于设置`func4`的参数，根据`x86-64`寄存器使用规范，第一个、第二个和第三个参数分别存储于寄存器`%edi`、`%esi`和`%edx`。

在查看`func4`对应的代码之前，先观察执行`callq  400fce <func4>`指令之后`phase_4`的操作：`test %eax,%eax`指令检查`%eax`的值是否等于`0`，如果不等于`0`，则会引爆炸弹，否则执行指令`cmpl $0x0,0xc(%rsp)`，该指令将输入的第二个数与`0`做比较，如果相等，那么`phase_4`正常退出，拆弹成功。因此，`phase_4`的第二个输入值即为`0`。

经过以上的分析，可以意识到`phase_4`的核心目标在于要让`func4`执行后，`%eax`的值等于`0`，这取决于输入的第一个数。接着需要分析`func4`执行的操作，其对应代码如下所示。

```assembly
0000000000400fce <func4>:
  400fce: 48 83 ec 08           sub    $0x8,%rsp
  400fd2: 89 d0                 mov    %edx,%eax
  400fd4: 29 f0                 sub    %esi,%eax
  400fd6: 89 c1                 mov    %eax,%ecx
  400fd8: c1 e9 1f              shr    $0x1f,%ecx
  400fdb: 01 c8                 add    %ecx,%eax
  400fdd: d1 f8                 sar    %eax
  400fdf: 8d 0c 30              lea    (%rax,%rsi,1),%ecx
  400fe2: 39 f9                 cmp    %edi,%ecx
  400fe4: 7e 0c                 jle    400ff2 <func4+0x24>
  400fe6: 8d 51 ff              lea    -0x1(%rcx),%edx
  400fe9: e8 e0 ff ff ff        callq  400fce <func4>
  400fee: 01 c0                 add    %eax,%eax
  400ff0: eb 15                 jmp    401007 <func4+0x39>
  400ff2: b8 00 00 00 00        mov    $0x0,%eax
  400ff7: 39 f9                 cmp    %edi,%ecx
  400ff9: 7d 0c                 jge    401007 <func4+0x39>
  400ffb: 8d 71 01              lea    0x1(%rcx),%esi
  400ffe: e8 cb ff ff ff        callq  400fce <func4>
  401003: 8d 44 00 01           lea    0x1(%rax,%rax,1),%eax
  401007: 48 83 c4 08           add    $0x8,%rsp
  40100b: c3                    retq   
```

在分析`func4`之前，不要忘了传递到`func4`的三个参数分别存储于寄存器`%edi`、`%esi`和`%edx`，其值分别为`x`(输入的第一个数)、`0`和`14`。在`0x400fe9`处执行了指令`callq 400fce <func4>`，因此`func4`很可能是个递归函数，我们将`func4`翻译成等价的`C`代码，如下所示。

```c
void func4(int x, int y, int z) {
    int t = z - y;
    int k = t >> 31;
    t = (t + k) >> 1;
    k = t + y;
    if(k <= x) {
        t = 0;
        if(k >= x) {
            return;
        }else {
            y = k + 1;
            func4(x, y, z);
        }
    }else {
        z = k - 1;
        func4(x, y, z);
    }
}
```

`func4`的目的是要让函数退出后`%eax`的值为`0`，而在`0x400ff2`处`mov $0x0,%eax`显示的将`%eax`的值设置为`0`，该指令对应于`C`代码中的`t = 0`。并且，`func4`执行递归的退出条件为`k == x`，其中`x`对应于输入的第一个数，而`k`则可以通过一系列计算得到，由于`y = 0`且`z = 14`，易知`k = 7`，因此输入的第一个数即为`7`。将字符串`7 0`作为`phase_4`的输入，拆弹成功，如下图所示。

![](/img/img-2017-04-26-Image 7.png)

## Phase 5

`phase_5`对应的代码如下所示。

```assembly
0000000000401062 <phase_5>:
  401062: 53                    push   %rbx
  401063: 48 83 ec 20           sub    $0x20,%rsp
  401067: 48 89 fb              mov    %rdi,%rbx
  40106a: 64 48 8b 04 25 28 00  mov    %fs:0x28,%rax
  401071: 00 00 
  401073: 48 89 44 24 18        mov    %rax,0x18(%rsp)
  401078: 31 c0                 xor    %eax,%eax
  40107a: e8 9c 02 00 00        callq  40131b <string_length>
  40107f: 83 f8 06              cmp    $0x6,%eax
  401082: 74 4e                 je     4010d2 <phase_5+0x70>
  401084: e8 b1 03 00 00        callq  40143a <explode_bomb>
  401089: eb 47                 jmp    4010d2 <phase_5+0x70>
  40108b: 0f b6 0c 03           movzbl (%rbx,%rax,1),%ecx
  40108f: 88 0c 24              mov    %cl,(%rsp)
  401092: 48 8b 14 24           mov    (%rsp),%rdx
  401096: 83 e2 0f              and    $0xf,%edx
  401099: 0f b6 92 b0 24 40 00  movzbl 0x4024b0(%rdx),%edx
  4010a0: 88 54 04 10           mov    %dl,0x10(%rsp,%rax,1)
  4010a4: 48 83 c0 01           add    $0x1,%rax
  4010a8: 48 83 f8 06           cmp    $0x6,%rax
  4010ac: 75 dd                 jne    40108b <phase_5+0x29>
  4010ae: c6 44 24 16 00        movb   $0x0,0x16(%rsp)
  4010b3: be 5e 24 40 00        mov    $0x40245e,%esi
  4010b8: 48 8d 7c 24 10        lea    0x10(%rsp),%rdi
  4010bd: e8 76 02 00 00        callq  401338 <strings_not_equal>
  4010c2: 85 c0                 test   %eax,%eax
  4010c4: 74 13                 je     4010d9 <phase_5+0x77>
  4010c6: e8 6f 03 00 00        callq  40143a <explode_bomb>
  4010cb: 0f 1f 44 00 00        nopl   0x0(%rax,%rax,1)
  4010d0: eb 07                 jmp    4010d9 <phase_5+0x77>
  4010d2: b8 00 00 00 00        mov    $0x0,%eax
  4010d7: eb b2                 jmp    40108b <phase_5+0x29>
  4010d9: 48 8b 44 24 18        mov    0x18(%rsp),%rax
  4010de: 64 48 33 04 25 28 00  xor    %fs:0x28,%rax
  4010e5: 00 00 
  4010e7: 74 05                 je     4010ee <phase_5+0x8c>
  4010e9: e8 42 fa ff ff        callq  400b30 <__stack_chk_fail@plt>
  4010ee: 48 83 c4 20           add    $0x20,%rsp
  4010f2: 5b                    pop    %rbx
  4010f3: c3                    retq   
```

根据`x86-64`寄存器使用规范，`%rdi`寄存器存储的是第一个参数的值，由于输入的是字符串，因此`%rdi`存储的应该是输入字符串的起始地址。`0x401067`处的指令`mov %rdi,%rbx`将字符串起始地址保存在`%rbx`中，即`%rbx`为基址寄存器。指令`xor %eax,%eax`的作用是将`%eax`清零，接着调用`string_length`函数获取输入字符串的长度，并将长度值（返回值）存储于`%eax`。指令`cmp $0x6,%eax`将`string_length`的返回值与常数`6`作比较，若不相等则会引爆炸弹，由此可以得知，`phase_5`的输入字符串长度应该等于`6`。

当输入字符串的长度等于`6`，`phase_5`跳转至`0x4010d2`处执行指令`mov $0x0,%eax`，接着又跳转至`0x40108b`处执行指令`movzbl (%rbx,%rax,1),%ecx`，可以发现`0x40108b`至`0x4010ac`构成了一个循环，且在循环退出后在`0x4010bd`处调动了`strings_not_equal`来比较字符串是否相等，若相等，则拆弹成功。其中，由`mov $0x40245e,%esi`指令可知，待比较的字符串存储于地址`0x40245e`处，打印以该地址作为起始地址的字符串，如下图所示。

![](/img/img-2017-04-26-Image 8.png)

待比较的字符串为`flyers`，且长度也为`6`。所以，接下来的关键任务是需要对循环操作进行分析，理解该循环操作对输入字符串做了哪些操作。提取循环操作的代码，如下所示。

```assembly
  40108b: 0f b6 0c 03           movzbl (%rbx,%rax,1),%ecx
  40108f: 88 0c 24              mov    %cl,(%rsp)
  401092: 48 8b 14 24           mov    (%rsp),%rdx
  401096: 83 e2 0f              and    $0xf,%edx
  401099: 0f b6 92 b0 24 40 00  movzbl 0x4024b0(%rdx),%edx
  4010a0: 88 54 04 10           mov    %dl,0x10(%rsp,%rax,1)
  4010a4: 48 83 c0 01           add    $0x1,%rax
  4010a8: 48 83 f8 06           cmp    $0x6,%rax
  4010ac: 75 dd                 jne    40108b <phase_5+0x29>
```

由于`%rbx`存储的是输入字符串的起始地址，`%rax`初始化为`0`，其作用等价于下标，因此`movzbl (%rbx,%rax,1),%ecx`指令的作用是将字符串的第`%rax`个字符存储于`%ecx`，`movzbl`意味做了零扩展。接着，`mov  %cl,(%rsp)`指令取`%ecx`的低`8`位，即一个字符的大小，通过内存间接存储至`%rdx`中。`and  $0xf,%edx`指令将`%edx`的值与常数`0xf`进行位与，由指令`movzbl 0x4024b0(%rdx),%edx`可知，位与后的值将会作为偏移量，以`0x4024b0`为基址，将偏移后的值存储至`%edx`。最后，指令`mov %dl,0x10(%rsp,%rax,1)`以`%edx`低`8`位的值作为新的字符，对原有字符进行替换。

综上，`phase_5`遍历输入字符串的每个字符，将字符的低`4`位作为偏移量，以`0x4024b0`为起始地址，将新地址对应的字符替换原有字符，最终得到`flyers`字符串。打印`0x4024b0`处的内容，如下图所示。

![](/img/img-2017-04-26-Image 9.png)

例如，如果要得到字符`f`，那么偏移量应为`9`，二进制表示为`1001`，通过查找`ASCII`表，可知字符`i`的`ASCII`编码为`01101001`，满足要求。剩余`5`个字符采用同样的策略可以依次求得，最终，`phase_5`的输入字符串为`ionefg`。

## Phase 6

`phase_6`对应的代码非常长，如下所示。

```assembly
00000000004010f4 <phase_6>:
  4010f4: 41 56                 push   %r14
  4010f6: 41 55                 push   %r13
  4010f8: 41 54                 push   %r12
  4010fa: 55                    push   %rbp
  4010fb: 53                    push   %rbx
  4010fc: 48 83 ec 50           sub    $0x50,%rsp
  401100: 49 89 e5              mov    %rsp,%r13
  401103: 48 89 e6              mov    %rsp,%rsi
  401106: e8 51 03 00 00        callq  40145c <read_six_numbers>
  40110b: 49 89 e6              mov    %rsp,%r14               # %r14存储数组起始地址
  40110e: 41 bc 00 00 00 00     mov    $0x0,%r12d              # 将%r12d初始化为0
  
  #################### Section 1:确认数组中所有的元素小于等于6且不存在重复值 ###################
  401114: 4c 89 ed              mov    %r13,%rbp               # %r13和%rbp存储数组某个元素的地址，并不是第1个元素，意识到这点需要结合0x40114d处的指令
  401117: 41 8b 45 00           mov    0x0(%r13),%eax          # %eax存储第%r13个数
  40111b: 83 e8 01              sub    $0x1,%eax               # 将%eax的值减1
  40111e: 83 f8 05              cmp    $0x5,%eax               # 将%eax的值与常数5做比较
  401121: 76 05                 jbe    401128 <phase_6+0x34>
  401123: e8 12 03 00 00        callq  40143a <explode_bomb>
  401128: 41 83 c4 01           add    $0x1,%r12d              # 如果%eax的值小于等于5，%r12d加1
  40112c: 41 83 fc 06           cmp    $0x6,%r12d              # 将%r12d与常数6做比较
  401130: 74 21                 je     401153 <phase_6+0x5f>
  401132: 44 89 e3              mov    %r12d,%ebx              # %ebx起了数组下标的作用
  
  # 用于判断数组6个数是否存在重复值，若存在，引爆炸弹
  401135: 48 63 c3              movslq %ebx,%rax               # 将数组下标存储至%rax
  401138: 8b 04 84              mov    (%rsp,%rax,4),%eax      # 将下一个数存储至%eax
  40113b: 39 45 00              cmp    %eax,0x0(%rbp)          # 将第1个数与%eax的值(当前数)做比较
  40113e: 75 05                 jne    401145 <phase_6+0x51>   # 若相等，引爆炸弹   
  401140: e8 f5 02 00 00        callq  40143a <explode_bomb>
  401145: 83 c3 01              add    $0x1,%ebx               # 数组下标加1            
  401148: 83 fb 05              cmp    $0x5,%ebx               # 判断数组下标是否越界(<=5)
  40114b: 7e e8                 jle    401135 <phase_6+0x41>
  
  40114d: 49 83 c5 04           add    $0x4,%r13               # %r13存储数组下一个数的地址
  401151: eb c1                 jmp    401114 <phase_6+0x20>
  ####################################### Section 1 end ######################################
  
  ################ Section 2：用7减去数组的每个元素，并将相减后的元素替换原有元素 #################
  401153: 48 8d 74 24 18        lea    0x18(%rsp),%rsi         # 0x18(%rsp)是数组的边界地址：0x18 = 24         
  401158: 4c 89 f0              mov    %r14,%rax               # 将数组起始地址存储于%rax
  40115b: b9 07 00 00 00        mov    $0x7,%ecx
  
  401160: 89 ca                 mov    %ecx,%edx               # %edx = 7
  401162: 2b 10                 sub    (%rax),%edx             # %edx = 7 - 数组元素
  401164: 89 10                 mov    %edx,(%rax)             # 用相减后的元素(%edx)替换原有元素
  401166: 48 83 c0 04           add    $0x4,%rax               # %rax存储数组下一个元素的地址
  40116a: 48 39 f0              cmp    %rsi,%rax               # 判断是否越界
  40116d: 75 f1                 jne    401160 <phase_6+0x6c>
  ####################################### Section 2 end ######################################
  
  ########################## Section 3：根据输入数组重排结构体数组 ##############################
  40116f: be 00 00 00 00        mov    $0x0,%esi               # 将%esi初始化为0，作为数组下标
  401174: eb 21                 jmp    401197 <phase_6+0xa3>
  
  401176: 48 8b 52 08           mov    0x8(%rdx),%rdx          # 0x8(%rdx)为下一个元素的地址
  40117a: 83 c0 01              add    $0x1,%eax               
  40117d: 39 c8                 cmp    %ecx,%eax               # %ecx存储了数组当前值(第%esi个元素)
  40117f: 75 f5                 jne    401176 <phase_6+0x82>
  
  401181: eb 05                 jmp    401188 <phase_6+0x94>
  401183: ba d0 32 60 00        mov    $0x6032d0,%edx          # %edx存储结构体数组第1个元素的地址
  401188: 48 89 54 74 20        mov    %rdx,0x20(%rsp,%rsi,2)  # %rsi的初始值为0；该指令的作用是将结构体数组的第%ecx个元素的地址存储在内存的某个位置(以%rsp + 0x20为基地址，%rsi为偏移量)
  40118d: 48 83 c6 04           add    $0x4,%rsi               # 增加偏移量
  401191: 48 83 fe 18           cmp    $0x18,%rsi
  401195: 74 14                 je     4011ab <phase_6+0xb7>
  401197: 8b 0c 34              mov    (%rsp,%rsi,1),%ecx      # %ecx存储数组第%esi个元素
  40119a: 83 f9 01              cmp    $0x1,%ecx               # 将数组第%esi个元素与常数1做比较
  40119d: 7e e4                 jle    401183 <phase_6+0x8f>   # 实际上不会小于1，如果数组的第1个元素等于1，那么跳转至0x401183处
  40119f: b8 01 00 00 00        mov    $0x1,%eax
  4011a4: ba d0 32 60 00        mov    $0x6032d0,%edx          # %edx存储结构体数组第1个元素的地址
  4011a9: eb cb                 jmp    401176 <phase_6+0x82>
  ####################################### Section 3 end ######################################
  
  ######################### Section 4：修改结构体数组元素的next域值 #############################
  4011ab: 48 8b 5c 24 20        mov    0x20(%rsp),%rbx         # %rbx存储地址数组的第1个元素的值
  4011b0: 48 8d 44 24 28        lea    0x28(%rsp),%rax         # %rax存储地址数组的第2个元素的地址
  4011b5: 48 8d 74 24 50        lea    0x50(%rsp),%rsi
  4011ba: 48 89 d9              mov    %rbx,%rcx               # %rcx存储地址数组的第1个元素的值
  
  # 下面用i和i+1来表示元素位置
  4011bd: 48 8b 10              mov    (%rax),%rdx             # %rdx存储地址数组的第i+1个元素的值
  4011c0: 48 89 51 08           mov    %rdx,0x8(%rcx)          # 把第i+1和元素的值存储于第i个结构体元素的next域中，next域的地址为0x8(%rcx)的值
  4011c4: 48 83 c0 08           add    $0x8,%rax
  4011c8: 48 39 f0              cmp    %rsi,%rax
  4011cb: 74 05                 je     4011d2 <phase_6+0xde>
  4011cd: 48 89 d1              mov    %rdx,%rcx
  4011d0: eb eb                 jmp    4011bd <phase_6+0xc9>
  ####################################### Section 4 end ######################################
  
  ######################### Section 5：判断结构体数组是否是递减序列 #############################
  4011d2: 48 c7 42 08 00 00 00  movq   $0x0,0x8(%rdx)
  4011d9: 00 
  4011da: bd 05 00 00 00        mov    $0x5,%ebp
  4011df: 48 8b 43 08           mov    0x8(%rbx),%rax
  4011e3: 8b 00                 mov    (%rax),%eax
  4011e5: 39 03                 cmp    %eax,(%rbx)
  4011e7: 7d 05                 jge    4011ee <phase_6+0xfa>
  4011e9: e8 4c 02 00 00        callq  40143a <explode_bomb>
  4011ee: 48 8b 5b 08           mov    0x8(%rbx),%rbx
  4011f2: 83 ed 01              sub    $0x1,%ebp
  4011f5: 75 e8                 jne    4011df <phase_6+0xeb>
  ####################################### Section 5 end ######################################
  
  4011f7: 48 83 c4 50           add    $0x50,%rsp
  4011fb: 5b                    pop    %rbx
  4011fc: 5d                    pop    %rbp
  4011fd: 41 5c                 pop    %r12
  4011ff: 41 5d                 pop    %r13
  401201: 41 5e                 pop    %r14
  401203: c3                    retq   
```

分析清楚`phase_6`非常需要耐心，我将`phase_6`划分为`5`个`Section`，每个`Section`完成特定的功能，详细的注释直接附到了相关代码。前两个`Section`不难理解：`Section 1`确保输入数组的值的范围在`1 ~ 6`且不存在重复值；`Section 2`用`7`减去输入数组的每个元素，相当于求补。`Section 3`中出现了一个常数地址，使用`gdb`将该地址存储的内容打印出来，如下图所示。

![](/img/img-2017-04-26-Image 11.png)

可以意识到这其实是一个链表数据结构，链表的节点由`3`部分组成：`value 1`、`value 2`和一个地址值(`next`域，指向下一个节点)。`Section 3`根据我们输入的数组，按照数组元素的值将对应结构体数组中的元素的首地址存储到内存的某个位置(`mov %rdx,0x20(%rsp,%rsi,2)`)。例如，假设输入数组为`[3, 4, 5, 6, 1, 2]`，那么`Section 3`首先会将结构体数组的第`3`个元素的地址存储到`0x20(%rsp,%rsi,2)`处，接着将结构体数组的第`4`个元素......依次类推。

`Section 4`根据`Section 3`构建的地址数组，修改结构体数组的`next`域的值，实现单链表的排序操作。`Section 5`进行验证，要求单链表递减排序，若满足要求，那么拆弹成功。

综上，根据已有的结构体数组以及`phase_6`的操作，若要实现单链表的递减排序，应将第`3`个节点放在第`1`位，将第`4`个节点放在第`2`位......最终得到序列：`[3, 4, 5, 6, 1, 2]`。不要忘记`Section 2`中的求补操作，所以`phase_6`的输入序列应该为`[4, 3, 2, 1, 6, 5]`。

## 小结

至此，`Bomb Lab`包含了`6`个`phase`全部拆弹成功。将输入序列存储在文件，然后将文件作为`bomb`二进制文件的参数，运行结果如下图所示。

![](/img/img-2017-04-26-Image 10.png)


---
title: JVM-3.JamVM解释器的实现
date: 2017-3-2 14:46:23
---

## executeJava

主要在 /src/interp/engine/interp.c 和 interp-inlining.h 中。

解释器的实现，对于 JVM 提供的每一个指令，在 interp.c 中都有一个对应的解释代码。其基本形式如下。

每一条指令对应一个代码片断，执行一个 Java 方法的时候，就将这个 Java 方法转化成一个指令数组。然后直接使用 goto 关键字进行方法调用，直到所有的指令的执行完毕。这个 Java 方法也就执行完成了。

## 实现

### 将字节码转换成解释器片断的代码。

interp/direct.c prepare函数 用来实现将 MethodBlock 中的代码翻译成已经编译好的解释代码的入口地址。

src\interp\engine\interp-direct-common.h 中定义了

``` c
/* Initialise pc to the start of the method.  If it
   hasn't been executed before it may need preparing */
// 定义在 interp-direct-common.h 中
// 最终调用 prepare 函数
// 其主要功能就是将 mb->code 由原来的 java bytecode
// 转换成对应的 解释器的代码的入口。
PREPARE_MB(mb);
pc = (CodePntr)mb->code;
```

``` c
typedef struct instruction {
    const void *handler;
    Operand operand;
} Instruction;

typedef union ins_operand {
    uintptr_t u;
    int i;
    struct {
        signed short i1;
        signed short i2;
    } ii;
    struct {
        unsigned short u1;
        unsigned short u2;
    } uu;
    struct {
        unsigned short u1;
        unsigned char u2;
        char i;
    } uui;
    void *pntr;
} Operand;
```

执行完毕之后的 mb->code 存储的格式如下。

``` bash

		++++++++++++
pc -->  |  handler |
		++++++++++++
		|  handler |
		++++++++++++
		|  handler |
		++++++++++++
		|  handler |
		++++++++++++
		|  handler |
		++++++++++++
		|  handler |
		++++++++++++
		|   ...    |
		++++++++++++
		|  handler |
		++++++++++++
		|  return  |
		++++++++++++
```

`handler` 指向的是字节码对应的解释片断的代码入口。代码执行流程如下。

```
call pc->handler
pc++;
call pc->handler
```

直到 handler 是 return 指令时，方法调用结束。

### 

## gdb调试

``` bash
p mb.code

## 查看  mb.code 中的 9 个字节
x/9xb mb.code
```

## handlers 初始化

使用下面的使用可以将 interp.c 进行预编译

``` bash
cd jamvm
gcc -I . -E -P interp/engine/interp.c -o interp.i
```

Jamvm 解释器实现的基本思路是：

JVM 使用 1个字节来定义指令。所以 JVM 能够支持的指令的个数最多是 256 个。而一个 JVM 指令就相当于一个 函数功能。

所以就可以实现一个 JVM 指令和函数入口的对照表。然后所谓执行一个 JAVA 方法的字节码，其实就是执行上面指令所对应的函数。

所以 executeJava 方法的第一条语句：

``` c
/* Definitions specific to the particular
   interpreter variant */
INTERPRETER_DEFINITIONS
```

初始化指令和指令实现代码（这里使用的是标签，而不是函数，这样调用开销大大下降）的一个静态对照表。有了这个对照表，就可以实现进行下一步操作。

## PREPARE_MB(mb) 的实现

其核心是 src/interp/direct.c 中实现

这个方法对 Methods 的 code 进行两遍循环。

其主要功能是将方法的 MethodBlock 中的 code 进行循环，将其转成，一个 Instruction *new_code 数组。并且将 MethodBlock 的状态设置成，已经解析，同时将 MethodBlock.code_size 设置成指令的个数，此后就可以直接调用这些方法。

``` c
// 定义在 jam.h
typedef union ins_operand {
    uintptr_t u;
    int i;
    struct {
        signed short i1;
        signed short i2;
    } ii;
    struct {
        unsigned short u1;
        unsigned short u2;
    } uu;
    struct {
        unsigned short u1;
        unsigned char u2;
        char i;
    } uui;
    void *pntr;
} Operand;
typedef struct instruction {
	// 所有的指令实现都在 executeJava 中，
	// 使用标签标明指令的地址，下面的 handler 就是
	// 指令的标签入口
    const void *handler;
    // 指令的操作数
    Operand operand;
} Instruction;

	// 这个变量表示指令个数
    int ins_count = 0;
    
    for(pass = 0; pass < 2; pass++) {
        int block_quickened = FALSE;
#ifdef INLINING
        int block_start = 0;
#endif
        int cache = 0;
        int pc;

        if(pass == 1)
        	// Instruction 数组
            new_code = sysMalloc((ins_count + 1) * sizeof(Instruction));

        for(ins_count = 0, pc = 0; pc < code_len; ins_count++) {
            int quickened = FALSE;
            Operand operand;
            int ins_cache;
            int opcode;
            
    // 通过 opcode这个是字节码
    //  1. 将操作数提取出来，放置到Operand operand 这个变量中
    // 对于每一个字节码，其操作数的字节数是固定的。
    //  2. 同时更新 pc 这个变量，这个变量始终指向下一个字节码
    switch(opcode) {
        default:
```

## 参考
1. [JamVM 栈顶缓存](http://hllvm.group.iteye.com/group/topic/34814#post-231982)
2. [获取标签地址](http://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html)
3. [C99 goto label地址实现C语言协程](http://www.voidcn.com/article/p-zvzdpqjf-pb.html)
4. [使用 goto + label 的例子](http://blog.csdn.net/weiwangchao_/article/details/7777385)
5. [JamVM的解释器](https://ybin.cc/jvm/jamvm-interpreter/)
6. [Interpreter Implementation Choices](http://realityforge.org/code/virtual-machines/2011/05/19/interpreters.html)
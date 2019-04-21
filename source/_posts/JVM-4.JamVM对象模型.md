---
title: JVM-4.JamVM对象模型
date: 2017-10-15 20:36:41
---

## 对象的表示

### 对象大小的确定

ClassBlock 结构中有一个字段（object_size）存储着对象大小，所以对于每一个类其大小在类被加载的时候就已经确定了。

### 对象的内存布局

``` 
/* 1 word header format
  31                                       210
   -------------------------------------------
  |              block size               |   |
   -------------------------------------------
   ^ has hashcode bit                        ^ alloc bit
    ^ hashcode taken bit                    ^ flc bit
                                           ^ special bit
*/

typedef struct object {
   uintptr_t lock;
   Class *class;
} Object;
```

4字节对象头 + Object 头部 + 对象字段区域

整个堆就是按照上面的布局进行存储

``` 
  ------------> +----------------+
				|     header     |
				+----------------+
				|     lock       |
				+----------------+
				|     class      |
	obj1		+----------------+
				|                |
				|                |
				|                |
				|     data       |
				|                |
				|                |
				|                |
  ------------> +----------------+
				|     header     |
				+----------------+
				|     lock       |
		obj2	+----------------+
				|     class      |
				+----------------+
				|                |
				|     data       |
				|                |
  ------------> +----------------+
				|     header     |
				+----------------+
				|     lock       |
				+----------------+
		obj3	|     class      |
				+----------------+
				|                |
				|     data       |
				|                |
  ------------> +----------------+
				|                |
				|    ......      |
				|                |
				+----------------+
```

## 对象的创建

### JamVM 的 new 指令实现

``` c
DEF_OPC_RW_4(OPC_NEW, OPC_ANEWARRAY, OPC_CHECKCAST, OPC_INSTANCEOF, ({
	// new 指令对应的 对象类 在常量池中的索引
    int idx = pc->operand.uui.u1;
    int opcode = pc->operand.uui.u2;
    int cache = pc->operand.uui.i;
    Class *class;

	// 根据索引加载创建类
    frame->last_pc = pc;
    class = resolveClass(mb->class, idx, TRUE, opcode == OPC_NEW);

    if(exceptionOccurred0(ee))
        goto throwException;
    
    if(opcode == OPC_NEW) {
        ClassBlock *cb = CLASS_CB(class);
        if(cb->access_flags & (ACC_INTERFACE | ACC_ABSTRACT)) {
            signalException(java_lang_InstantiationError, cb->name);
            goto throwException;
        }
    }

	// OPCODE_REWRITE 表示重写 opcode 所对应的 *_QUICK 方法
	// 例如 OPC_NEW ==> OPC_NEW_QUICK
	// 此时  pc 将指向 OPC_NEW_QUICK
    OPCODE_REWRITE((opcode + OPC_NEW_QUICK-OPC_NEW), cache, pc->operand);
    // 调用下面的 OPC_NEW_QUICK 方法
    REDISPATCH
});)

DEF_OPC_210(OPC_NEW_QUICK, {
    Class *class = RESOLVED_CLASS(pc);
    Object *obj;

    frame->last_pc = pc;
    // 调用 allocObject 分配内存，创建对象。
    if((obj = allocObject(class)) == NULL)
        goto throwException;

	// 将对象入栈
    PUSH_0((uintptr_t)obj, 3);
})
```

### 创建一个 Java 对象

一条 new 语句将做两件事，new 指令分配对象内存，invokespecial 指令初始化对象内存

``` java
Dog dog = new Dog("tuantuan", 8);
```

``` bytecode
## 上面的代码编译后生成的代码如下
new my/Dog
dup
ldc "tuantuan"
bipush 8
invokespecial my/Dog/<init>(Ljava/lang/String;I)V
astore_1
```

其中的 `new my/Dog` 这个指令，就会加载 my.Dog 类，然后生成 Dog 类的 Class 对象，然后生成这个类的一个对象。

#### 分配内存

在 Jamvm 中创建对象的方法及JVM堆内存管理的代码主要在 src/alloc.c 中。

其中创建对象实例的方法是 `allocObject`

``` c
Object *allocObject(Class *class) {
    ClassBlock *cb = CLASS_CB(class);
    // 调用 gcMalloc 在堆内存上分配对象。
    // 对象分配好之后 调用 memset(ret_addr, 0, n-HEADER_SIZE);
    // 将这块内存区域进行默认初始化，
    // 全部设置成 0, 这也就是类的字段，即使不显示初始化
    // 也会初始化成 0, null 等等，其实就是这里的显示初始化的原因
    Object *ob = gcMalloc(cb->object_size);

    if(ob != NULL) {
        ob->class = class;
		// ...
    }

    return ob;
}
```

#### 初始化对象

对象的内存分配好之后，初始化这块内存区域。

使用 invokespecial 指令调用类的 <init> 方法，类的构造函数将生成名称为 <init> 的方法。这类方法就用在的对象内存的初始化。

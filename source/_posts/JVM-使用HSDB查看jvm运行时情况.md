---
title: JVM-使用HSDB查看jvm运行时情况
date: 2017-1-17 17:59:22
---

## 0. 准备实验环境

### 0.1 源码

``` java
public class Test {

	public static void main(String[] args) {
		int a = 1;
		int b = 1;
		
		Test t = new Test();
		
		int c = t.add(a, b);
		
		System.out.println(c);
	}
	
	public int add(int a, int b){
		return a + b;
	}
}
```

使用以下命令进行编译

``` bash
## 使用 -g 参数可以保留调试符号信息，便于调试
javac -g Test.java
```

### 0.2 调试

``` bash
## 进入 jdb
## 启动 jdb 的时候可以使用 -XX:-UseCompressedOops	参数
## 表示禁用对象压缩，这样便于观察对象的内存布局
jdb -XX:+UseSerialGC -Xmx10m
## 设置断点
jdb> stop in Test.add
jdb> run Test
```

## 1. 运行时栈数据

启动 HSDB 

``` bash
java -cp .;"%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB
```

使用 `jps` 命令获得上面已经启动的 Test 程序的 pid, 然后将 hsdb attatch 到 pid 上。

观察主线程的栈。可以了解到栈桢的信息。

``` c
// ------------------------------ Asm interpreter ----------------------------------------
// Layout of asm interpreter frame:
//    [expression stack      ] * <- sp
//    [monitors              ]   \
//     ...                        | monitor block size
//    [monitors              ]   /
//    [monitor block size    ]
//    [byte code index/pointr]                   = bcx()                bcx_offset
//    [pointer to locals     ]                   = locals()             locals_offset
//    [constant pool cache   ]                   = cache()              cache_offset
//    [methodData            ]                   = mdp()                mdx_offset
//    [methodOop             ]                   = method()             method_offset
//    [last sp               ]                   = last_sp()            last_sp_offset
//    [old stack pointer     ]                     (sender_sp)          sender_sp_offset
//    [old frame pointer     ]   <- fp           = link()
//    [return pc             ]
//    [oop temp              ]                     (only for native calls)
//    [locals and parameters ]
//                               <- sender sp
// ------------------------------ Asm interpreter ----------------------------------------
```

## Klass

``` bash
//  Klass layout:
//    [header        ] klassOop
//    [klass pointer ] klassOop
//    [C++ vtbl ptr  ] (contained in Klass_vtbl)
//    [layout_helper ]
//    [super_check_offset   ] for fast subtype checks
//    [secondary_super_cache] for fast subtype checks
//    [secondary_supers     ] array of 2ndary supertypes
//    [primary_supers 0]
//    [primary_supers 1]
//    [primary_supers 2]
//    ...
//    [primary_supers 7]
//    [java_mirror   ]
//    [super         ]
//    [name          ] -- null
//    [first subklass] -- null
//    [next_sibling  ] link to chain additional subklasses   0x0020002000000000
//    [modifier_flags]
//    [access_flags  ]
//    [verify_count  ] - not in product
//    [alloc_count   ]
//    [last_biased_lock_bulk_revocation_time] (64 bits)
//    [prototype_header]
//    [biased_lock_revocation_count]
```

## $. 参考
1. [HSDB介绍](http://www.cnblogs.com/cyhe/archive/2016/12/03/6129436.html)
2. [借HSDB来探索HotSpot VM的运行时数据](http://www.cnblogs.com/princessd8251/articles/5333675.html)
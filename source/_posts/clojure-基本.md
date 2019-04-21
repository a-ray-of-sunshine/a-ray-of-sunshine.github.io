---
title: clojure-基本
date: 2017-3-31 10:45:18
---

## clojure 安装使用

安装过程

``` bash
## 安装 lein
lein self-install

## 创建一个 clojure 工程
lein new app myclojure

## 编译命名空间
lein compile myclojure.core

## 运行命名空间
lein run myclojure.core

## 运行指定的方法
lein run -m myclojure.demo/say
```

## clojure 编译

在一个命名空间中的函数将编译成该命令空间下的内部类。

函数将继承自 clojure.lang.RestFn 类， 并 override 一个对应参数个数的 invoke 方法。

例如：
``` clojure
(defn -main
  "I don't do a whole lot ... yet."
  [& args]
  (println "Hello, World!"))
```
函数 -main 接受一个 序列 作为参数，所以编译后的代码中重写
``` java
// clojure.lang.RestFn
protected Object doInvoke(Object args) {
	return null;
}
```

一般还会生成一个 invokeStatic 方法，这个方法包涵 函数 的核心实现逻辑。 doInvoke 将参数准备好之后，就直接调用这个方法。

如果在函数在的实现中使用的其它函数，则会有一个 clojure.lang.Var 类型的变量来引用这个函数对象。例如上面的 -main 函数中使用了 函数 clojure.core/println 进行输出操作。

则，函数编译后的类中，将会一个对应的 Var字段, 代表这个函数的引用, 其初始化的方法，如下：
``` java
// 使用 RT 的静态方法，var 进行命名空间的查找。
// 每一个 Var 对象都有一个 root 变量
// 这个变量存储的就是该 Var 所引用的 RestFn 对象
// 所以可以通过 var 调用 RestFn。
// 从而执行函数所表达的功能。
Var var = RT.var("clojure.core", "println");
```

``` clojure
(defn println
  "Same as print followed by (newline)"
  {:added "1.0"
   :static true}
  [& more]
    (binding [*print-readably* nil]
      (apply prn more)))
```

``` bytecode
;; init 
;; 在 init 函数中初始化 const__1303
ldc_w "clojure.core"
ldc_w "println"
invokestatic clojure/lang/RT/var(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Var;
checkcast clojure/lang/Var
putstatic clojure/core__init/const__1303 Lclojure/lang/Var;
;; 上面的字节码片断相当于下面的代码
;; var 方法创建命令空间 "clojure.core", 
;; 并创建一个 var 对象。将其放置到命名空间中，然后返回
Var const__1303 = clojure.lang.RT.var("clojure.core", "println");

;; const__1306 的初始化
invokestatic clojure/lang/RT/map([Ljava/lang/Object;)Lclojure/lang/IPersistentMap;
checkcast clojure/lang/AFn
putstatic clojure/core__init/const__1306 Lclojure/lang/AFn;
;; 上面的字节码片断相当于下面的代码 
;; 其中对象数组中存储着 clojure.core/println 函数的元信息
;; 例如 {:added "1.0"
;;      :static true}
;; RT.map 方法返回一个 PersistentArrayMap 对象
;; 这个对象继承自 AFn
AFn const__1306 = clojure.lang.RT.map(new Object{//....});

;; core__init.class/load 

;; const__1303 （Var）指向 clojure/core$println 对象。
getstatic clojure/core__init/const__1303 Lclojure/lang/Var;
dup

;; const__1306 指向
getstatic clojure/core__init/const__1306 Lclojure/lang/AFn;
checkcast clojure/lang/IPersistentMap
invokevirtual clojure/lang/Var/setMeta(Lclojure/lang/IPersistentMap;)V
dup

;; 创建类 clojure/core$println 的对象。
new clojure/core$println
dup
invokespecial clojure/core$println/<init>()V
;; 调用上面 const__1303 这个 Var 的 bindRoot 方法。
;; 将创建好的 clojure/core$println 对象设置为其 root 对象。
invokevirtual clojure/lang/Var/bindRoot(Ljava/lang/Object;)V

;; 上面的字节码片断相当于下面的代码
Var const__1303;
AFn const__1306;
const__1303.setMeta(const__1306);
const__1303.bindRoot(new clojure.core$println());
```

``` bytecode
;; clojure.core__init.class/__init13 方法。
;; const__1303 的创建
;; !!! clojure/lang/RT/var 这个方法调用最终将 !!!
;; !!! 创建一个 "clojure.core" 的命令空间 
;; !!! 并将 "println" 注册到这个命令空间中
;; 此后， Namespace.mappings 对象中 将有一个
;; key为 "println" 的对象。
;; 同时创建的每一个命令空间将被放置到 Namespace.namespaces
;; 这个静态字段中，并且其 key 为 "clojure.core"
ldc_w "clojure.core"
ldc_w "println"
invokestatic clojure/lang/RT/var(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Var;
checkcast clojure/lang/Var
putstatic clojure/core__init/const__1303 Lclojure/lang/Var;

;; const__1306 的创建
bipush 14
anewarray java/lang/Object
dup
iconst_0
aconst_null
ldc_w "arglists"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
iconst_1
iconst_1
anewarray java/lang/Object
dup
iconst_0
aconst_null
ldc_w "&"
invokestatic clojure/lang/Symbol/intern(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Symbol;
aconst_null
ldc_w "more"
invokestatic clojure/lang/Symbol/intern(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Symbol;
invokestatic clojure/lang/Tuple/create(Ljava/lang/Object;Ljava/lang/Object;)Lclojure/lang/IPersistentVector;
aastore
invokestatic java/util/Arrays/asList([Ljava/lang/Object;)Ljava/util/List;
invokestatic clojure/lang/PersistentList/create(Ljava/util/List;)Lclojure/lang/IPersistentList;
aastore
dup
iconst_2
aconst_null
ldc_w "doc"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
iconst_3
ldc_w "Same as print followed by (newline)"
aastore
dup
iconst_4
aconst_null
ldc_w "added"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
iconst_5
ldc_w "1.0"
aastore
dup
bipush 6
aconst_null
ldc_w "static"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
bipush 7
getstatic java/lang/Boolean/TRUE Ljava/lang/Boolean;
aastore
dup
bipush 8
aconst_null
ldc_w "line"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
bipush 9
sipush 3631
invokestatic java/lang/Integer/valueOf(I)Ljava/lang/Integer;
aastore
dup
bipush 10
aconst_null
ldc_w "column"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
bipush 11
iconst_1
invokestatic java/lang/Integer/valueOf(I)Ljava/lang/Integer;
aastore
dup
bipush 12
aconst_null
ldc_w "file"
invokestatic clojure/lang/RT/keyword(Ljava/lang/String;Ljava/lang/String;)Lclojure/lang/Keyword;
aastore
dup
bipush 13
ldc_w "clojure/core.clj"
aastore
invokestatic clojure/lang/RT/map([Ljava/lang/Object;)Lclojure/lang/IPersistentMap;
checkcast clojure/lang/AFn
putstatic clojure/core__init/const__1306 Lclojure/lang/AFn;
```

``` bytecode
;; const__1303 和 const__1306 初始化
;; load 函数。
getstatic clojure/core__init/const__1303 Lclojure/lang/Var;
dup
getstatic clojure/core__init/const__1306 Lclojure/lang/AFn;
checkcast clojure/lang/IPersistentMap
invokevirtual clojure/lang/Var/setMeta(Lclojure/lang/IPersistentMap;)V
dup
new clojure/core$println
dup
invokespecial clojure/core$println/<init>()V
invokevirtual clojure/lang/Var/bindRoot(Ljava/lang/Object;)V
```

所以 core__init.class 的主要作用就是创建函数对象的实例。

## clojure 运行时的准备

clojure 运行时环境的准备由 clojure.lang.RT 类来完成，这个类会首先创建一个默认的命名空间 "clojure.core"

``` java
static public final Namespace CLOJURE_NS = Namespace.findOrCreate(Symbol.intern("clojure.core"));
```

``` bash
## Namespace 类

## Namespace 类的 namespaces 字段，将存储所有 clojure 命名空间
## 的 Symbol 和 其对应的 Namespace 对象的引用
## 每一个被加载的命名空间将都会注册到这个对象上
final static ConcurrentHashMap<Symbol, Namespace> namespaces = new ConcurrentHashMap<Symbol, Namespace>();

## 具体的 Namesapce
## 这个对象是 clojure.lang.PersistentArrayMap， 其 key 是 Symbol，
## 也就是函数的名称，
## value 则是对应的是一个 clojure.lang.Var 对象
## 这个 Var 对象持有一个 root 对象，通常这个 root 对象会被绑定到
## clojure.lang.RestFn 类的子类，也就是具体的函数类。
## 这样就可以通过 命名空间和函数名称来获得 Var 对象的 root 对象
## 从而可以调用函数。
transient final AtomicReference<IPersistentMap> mappings = new AtomicReference<IPersistentMap>();
```

## 变量

### 局部变量

``` clojure
;; 创建局部变量
;; 并且这个变量的作用域只在 let 函数范围内
(defn defvar []
	(let [a 123]
		(println a)))
```

上面的函数相当于下面的 java 代码

``` java
void defvar(){

	{
	// 相当于 let 域
		int a = 123;
		System.out.println(a);
	}

	// 已经超出了 let 域
	// 下面的调用会编译通过
	// 因为符号 a 找不到
	System.out.println(a);
}
```

### 全局变量

而使用 (def a), 则是在当前的命名空间中创建了一个 Var 变量。从而这个方法可以通过 命名空间+变量名称的 方式被任何一个函数使用。

通过(def) 语句定义的变量相当于类的字段。

命名空间的 __init.class 类，会把这个 Var 对象注册到当前命名空间中。

当前命名空间和其它命名空间中的函数都可以通过 命名空间+变量名 的方法访问该变量。

修改全局变量，仍然使用 (def) 语句

(def x new_value)

这个语句将调用 Var 变量 x 的 bindRoot 方法将新值绑定到 x.

### 线程绑定

``` clojure
;; 定义一个可以进行动态绑定的变量
(def ^:dynamic x)

(defn defvar []
	;; binding 方法，将会以 x 变量作为 key
	;; 待绑定的值作为 value, 
	;; 放置到一个 ThreadLocal 变量中
	;; (binding) 的 body 语句在调用之后
	;; 会将此次绑定 pop出当前线程。
	(binding [x (java.util.Date.)]
	(binding [x "hello binding."]
		(println x))
		(println x))
	(println x))
```

实现线程绑定的核心是  clojure.lang.Var.pushThreadBindings 和 clojure.lang.Var.popThreadBindings

### Ref 加锁的引用 

使用 (def x (ref 123)) 创建一个 clojure.lang.Ref.Ref 对象
每一个 ref 对象都有一个锁，对其进行操作时会默认进行加锁。

使用 @x 可以获得变量的值，

``` clojure
(defn defvar []
	(def x (ref 890))
	(println @x x))
```

### Atom 变量

``` clojure
(def my-atom (atom 1))
(reset! my-atom 2)
(println @my-atom)
```

atom 是 java.util.concurrent.atomic.AtomicReference 的包装，便于在 clojure 中使用

### Agent



## $. 参考
1. [clojure docs](https://clojuredocs.org/clojure.core)
2. [clojure](https://clojure.org/)
3. [clojure.github](http://clojure.github.io/)
4. [leiningen](https://leiningen.org/)
5. [clojure-doc](http://clojure-doc.org/)
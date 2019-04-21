---
title: JVM-java对象模型的实现
date: 2016-8-31 14:15:48
---

JVM 是如何实现 Java 语言的 对象系统呢?

使用 klass-oop 对象模型

[深入探究JVM | klass-oop对象模型研究](http://www.sczyh30.com/posts/Java/jvm-klass-oop/)

[HotSpotVM 对象机制实现浅析](http://blog.csdn.net/kisimple/article/details/44632443)

[CompressedOops](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)

[Strongtalk oops](https://github.com/talksmall/Strongtalk/tree/master/vm/oops)

JVM 使用 oop-klass 模型来实现 java 的对象模型。

整个 oop-klass 模型 类层次的定义在 
hotspot\src\share\vm\oops\oopsHierarchy.hpp 文件中

## oopsHierarchy

* oop

	``` c++
	typedef class oopDesc*                            oop;
	typedef class   instanceOopDesc*            instanceOop;
	typedef class   methodOopDesc*                    methodOop;
	typedef class   constMethodOopDesc*            constMethodOop;
	typedef class   methodDataOopDesc*            methodDataOop;
	typedef class   arrayOopDesc*                    arrayOop;
	typedef class     objArrayOopDesc*            objArrayOop;
	typedef class     typeArrayOopDesc*            typeArrayOop;
	typedef class   constantPoolOopDesc*            constantPoolOop;
	typedef class   constantPoolCacheOopDesc*   constantPoolCacheOop;
	typedef class   klassOopDesc*                    klassOop;
	typedef class   markOopDesc*                    markOop;
	typedef class   compiledICHolderOopDesc*    compiledICHolderOop;
	```
	
* klass

	``` c++
	class Klass;
	class   instanceKlass;
	class     instanceMirrorKlass;
	class     instanceRefKlass;
	class   methodKlass;
	class   constMethodKlass;
	class   methodDataKlass;
	class   klassKlass;
	class     instanceKlassKlass;
	class     arrayKlassKlass;
	class       objArrayKlassKlass;
	class       typeArrayKlassKlass;
	class   arrayKlass;
	class     objArrayKlass;
	class     typeArrayKlass;
	class   constantPoolKlass;
	class   constantPoolCacheKlass;
	class   compiledICHolderKlass;
	```
	
oop的声明 hotspot\src\share\vm\oops\oop.hpp，其实现文件在 oop.cpp 和 oop.inline.hpp 文件中。

klass的声明 hotspot\src\share\vm\oops\klass.hpp，其实现文件 klass.cpp 中。

oop中实例数据的存取：
``` c++
// 获得数据区域基址
inline void*     oopDesc::field_base(int offset)        const { return (void*)&((char*)this)[offset]; }

// 依据索引 offset 获得字段存储位置
inline jint*     oopDesc::int_field_addr(int offset)    const { return (jint*)    field_base(offset); }

// 数据获取
inline jint oopDesc::int_field(int offset) const                    { return *int_field_addr(offset);        }
// 数据更新
inline void oopDesc::int_field_put(int offset, jint contents)       { *int_field_addr(offset) = contents;    }

// 加锁取数据
inline jint oopDesc::int_field_acquire(int offset) const                    { return OrderAccess::load_acquire(int_field_addr(offset));      }
// 存储数据
inline void oopDesc::release_int_field_put(int offset, jint contents)       { OrderAccess::release_store(int_field_addr(offset), contents);  }
```

```
	oop ---> +-----------+ ---}
			 |	 _mark   |    |
			 +-----------+    |---> header
			 | _metadata |    |
		{--- +-----------+ ---}
		|	 | 0x238AB801|---> +-----------+
		|	 +-----------+     |   field1  |
		|	 |	  ...	 | 	   +-----------+
		|	 +-----------+
data<---|	 |	  ...	 | 
		|	 +-----------+
		|	 | 0x338AB801|---> +-----------+
		|	 +-----------+     |   fieldN  |
		|	 |	  ...	 |     +-----------+
		{--- +-----------+
```

可以看到所有的 Java 对象的字段，最终是以字段的地址的列表存储在 oop 中，然后通过 offset （可以认为是 索引）获得字段所在的地址。然后使用该地址来访问对象的字段。这个 offset 应该和 字段在java 文件中的声明顺序相对应。

现在，比较有趣的就是如何针对不同的 java 类，分配不同的内存，因为不同的 java 类，拥有的字段也不同。所以 data 区域的大小自然不同。

``` java
class C1{
	private int i;
}

class C2{
	private int i;
	private double d;
}
```

当执行：
``` java
C1 c1 = new C1();
C2 c2 = new C2();
```

c1 所对应的 oop 该如何分配内存呢？下面是猜测实现：

c2 和 c1 明显所占内存不同。其实也简单。当 C1 和 C2 被载入 JVM 的时候，会创建其所对应的 klass 类的对应，这个对象就是用来描述 C1 和 C2 这种 java 类的，自然这个类当然知道，C1 和 C2 内部所含有字段个数，以及所需要的内存大小。

``` c++
// oop
oop c1Oop;
oop c2Oop;

// c1 的 Klass
// C1 加载的时候，已经初始化好了
Klass c1Klass;

// c2 的 Klass
// C2 加载的时候，已经初始化好了
Klass c2Klass;

// 通过动态分配内存
void* c1ptr = malloc(c1Klass.size());
// 关键实现，直接强转。
c1Oop = (opp)c1ptr;

void* c2ptr = malloc(c2Klass.size());
c2Oop = (oop)c2ptr;
```

通过强制类型转换，解决了两个问题：

* 不同的 JAVA 类大小不同，所以对应的 oop 大小也应该不同，需要动态分配
* 保证的类型安全

下面来验证上面的推理：
oops/instanceKlass.cpp
instanceKlass 是 instanceOop 类相对应。
``` c++
// 分配一个当前 instanceKlass 类型的 实例 instanceOop
instanceOop instanceKlass::allocate_instance(TRAPS) {
  assert(!oop_is_instanceMirror(), "wrong allocation path");
  bool has_finalizer_flag = has_finalizer(); // Query before possible GC
  
  // 调用 size_helper 方法计算 oop 对象的大小
  // size_helper 定义在 Klass 中，Klass 表示 Java 类型，自然
  // 这个类可以确定 Java 类 的大小。
  int size = size_helper();  // Query before forming handle.

  // KlassHandle 一个快捷类，帮助句柄之间的转换
  KlassHandle h_k(THREAD, as_klassOop());

  instanceOop i;

  // 使用 CollectedHeap::obj_allocate 分配内存空间
  // 然后，直接进行类型转换，转换成 instanceOop 类型。
  i = (instanceOop)CollectedHeap::obj_allocate(h_k, size, CHECK_NULL);
  if (has_finalizer_flag && !RegisterFinalizersAtInit) {
    i = register_finalizer(i, CHECK_NULL);
  }
  return i;
}
```

可以看到这个方法使用 CollectedHeap::obj_allocate 来分配 oop 对象。

``` c++
oop CollectedHeap::obj_allocate(KlassHandle klass, int size, TRAPS) {
  debug_only(check_for_valid_allocation_state());
  assert(!Universe::heap()->is_gc_active(), "Allocation during gc not allowed");
  assert(size >= 0, "int won't convert to size_t");

  // 在堆上分配内存
  HeapWord* obj = common_mem_allocate_init(size, false, CHECK_NULL);
  post_allocation_setup_obj(klass, obj, size);
  NOT_PRODUCT(Universe::heap()->check_for_bad_heap_word_value(obj, size));
  return (oop)obj;
}
```

* JVM_NewInstance

``` c++
JVM_ENTRY(jobject, JVM_NewInstance(JNIEnv *env, jclass cls))
  JVMWrapper("JVM_NewInstance");
  Handle mirror(THREAD, JNIHandles::resolve_non_null(cls));

  methodOop resolved_constructor = java_lang_Class::resolved_constructor(mirror());
  if (resolved_constructor == NULL) {
    klassOop k = java_lang_Class::as_klassOop(mirror());
    // The java.lang.Class object caches a resolved constructor if all the checks
    // below were done successfully and a constructor was found.

    // Do class based checks
    if (java_lang_Class::is_primitive(mirror())) {
      const char* msg = "";
      if      (mirror == Universe::bool_mirror())   msg = "java/lang/Boolean";
      else if (mirror == Universe::char_mirror())   msg = "java/lang/Character";
      else if (mirror == Universe::float_mirror())  msg = "java/lang/Float";
      else if (mirror == Universe::double_mirror()) msg = "java/lang/Double";
      else if (mirror == Universe::byte_mirror())   msg = "java/lang/Byte";
      else if (mirror == Universe::short_mirror())  msg = "java/lang/Short";
      else if (mirror == Universe::int_mirror())    msg = "java/lang/Integer";
      else if (mirror == Universe::long_mirror())   msg = "java/lang/Long";
      THROW_MSG_0(vmSymbols::java_lang_NullPointerException(), msg);
    }

    // Check whether we are allowed to instantiate this class
    Klass::cast(k)->check_valid_for_instantiation(false, CHECK_NULL); // Array classes get caught here
    instanceKlassHandle klass(THREAD, k);
    // Make sure class is initialized (also so all methods are rewritten)
    klass->initialize(CHECK_NULL);

    // Lookup default constructor
    resolved_constructor = klass->find_method(vmSymbols::object_initializer_name(), vmSymbols::void_method_signature());
    if (resolved_constructor == NULL) {
      ResourceMark rm(THREAD);
      THROW_MSG_0(vmSymbols::java_lang_InstantiationException(), klass->external_name());
    }

    // Cache result in java.lang.Class object. Does not have to be MT safe.
    java_lang_Class::set_resolved_constructor(mirror(), resolved_constructor);
  }

  assert(resolved_constructor != NULL, "sanity check");
  methodHandle constructor = methodHandle(THREAD, resolved_constructor);

  // We have an initialized instanceKlass with a default constructor
  instanceKlassHandle klass(THREAD, java_lang_Class::as_klassOop(JNIHandles::resolve_non_null(cls)));
  assert(klass->is_initialized() || klass->is_being_initialized(), "sanity check");

  // Do security check
  klassOop caller_klass = NULL;
  if (UsePrivilegedStack) {
    caller_klass = thread->security_get_caller_class(2);

    if (!Reflection::verify_class_access(caller_klass, klass(), false) ||
        !Reflection::verify_field_access(caller_klass,
                                         klass(),
                                         klass(),
                                         constructor->access_flags(),
                                         false,
                                         true)) {
      ResourceMark rm(THREAD);
      THROW_MSG_0(vmSymbols::java_lang_IllegalAccessException(), klass->external_name());
    }
  }

  // Allocate object and call constructor
  Handle receiver = klass->allocate_instance_handle(CHECK_NULL);
  JavaCalls::call_default_constructor(thread, constructor, receiver, CHECK_NULL);

  jobject res = JNIHandles::make_local(env, receiver());
  if (JvmtiExport::should_post_vm_object_alloc()) {
    JvmtiExport::post_vm_object_alloc(JavaThread::current(), receiver());
  }
  return res;
JVM_END

// 上面实现的核心代码如下

JVM_ENTRY(jobject, JVM_NewInstance(JNIEnv *env, jclass cls))
  JVMWrapper("JVM_NewInstance");

  // 1. 解析类 cls 的构造函数
  Handle mirror(THREAD, JNIHandles::resolve_non_null(cls));
  methodOop resolved_constructor = java_lang_Class::resolved_constructor(mirror());
  methodHandle constructor = methodHandle(THREAD, resolved_constructor);

  // We have an initialized instanceKlass with a default constructor
  // 2. 解析类 cls 的 Klass 
  instanceKlassHandle klass(THREAD, java_lang_Class::as_klassOop(JNIHandles::resolve_non_null(cls)));

  // Allocate object and call constructor
  // 3. 使用类的 klass 分配内存
  Handle receiver = klass->allocate_instance_handle(CHECK_NULL);
  // 4. 使用 constructor 来初始化内存（这里是 c++ 代码调用 java 对象的 构造函数）
  //    使用 JavaCalls::call_default_constructor 调用 java 代码的构造函数
  JavaCalls::call_default_constructor(thread, constructor, receiver, CHECK_NULL);

  // 5. 返回创建好的对象
  jobject res = JNIHandles::make_local(env, receiver());
  return res;
JVM_END
```
JVM_NewInstance 使用 klass->allocate_instance_handle 来分配内存，其实现代码如下：

``` c++
// instanceKlass.hpp
// 可见在这时还是使用 allocate_instance 来分配内存。
instanceHandle allocate_instance_handle(TRAPS)      { return instanceHandle(THREAD, allocate_instance(THREAD)); }
```

## 对象分配

使用 java 语言的 new 关键字分配对象的实现过程：

参考：<<HotSpot实战>> 3.3 创建对象

hotspot\src\share\vm\interpreter\interpreterRuntime.cpp
InterpreterRuntime::_new

interpreterRuntime.cpp中有很多 java 关键实现，

例如：monitorenter 指令的实现，InterpreterRuntime::monitorenter

monitorexit 指令：InterpreterRuntime::monitorexit

## java创建对象的方式

* new

new 关键字的实现，参考：
hotspot\src\share\vm\interpreter\interpreterRuntime.cpp
InterpreterRuntime::_new

``` c++
IRT_ENTRY(void, InterpreterRuntime::_new(JavaThread* thread, constantPoolOopDesc* pool, int index))
  klassOop k_oop = pool->klass_at(index, CHECK);
  instanceKlassHandle klass (THREAD, k_oop);

  // Make sure we are not instantiating an abstract klass
  klass->check_valid_for_instantiation(true, CHECK);

  // Make sure klass is initialized
  klass->initialize(CHECK);

  // At this point the class may not be fully initialized
  // because of recursive initialization. If it is fully
  // initialized & has_finalized is not set, we rewrite
  // it into its fast version (Note: no locking is needed
  // here since this is an atomic byte write and can be
  // done more than once).
  //
  // Note: In case of classes with has_finalized we don't
  //       rewrite since that saves us an extra check in
  //       the fast version which then would call the
  //       slow version anyway (and do a call back into
  //       Java).
  //       If we have a breakpoint, then we don't rewrite
  //       because the _breakpoint bytecode would be lost.
  oop obj = klass->allocate_instance(CHECK);
  thread->set_vm_result(obj);
IRT_END
```

调用 klass->allocate_instance(CHECK); 来创建对象。

* Class.newInstance


``` c++
// jvm.cpp
JVM_ENTRY(jobject, JVM_NewInstanceFromConstructor(JNIEnv *env, jobject c, jobjectArray args0))
  JVMWrapper("JVM_NewInstanceFromConstructor");
  oop constructor_mirror = JNIHandles::resolve(c);
  objArrayHandle args(THREAD, objArrayOop(JNIHandles::resolve(args0)));
  oop result = Reflection::invoke_constructor(constructor_mirror, args, CHECK_NULL);
  jobject res = JNIHandles::make_local(env, result);
  if (JvmtiExport::should_post_vm_object_alloc()) {
    JvmtiExport::post_vm_object_alloc(JavaThread::current(), result);
  }
  return res;
JVM_END

// 	hotspot\src\share\vm\runtime\reflection.cpp
oop Reflection::invoke_constructor(oop constructor_mirror, objArrayHandle args, TRAPS) {
  oop mirror             = java_lang_reflect_Constructor::clazz(constructor_mirror);
  int slot               = java_lang_reflect_Constructor::slot(constructor_mirror);
  bool override          = java_lang_reflect_Constructor::override(constructor_mirror) != 0;
  objArrayHandle ptypes(THREAD, objArrayOop(java_lang_reflect_Constructor::parameter_types(constructor_mirror)));

  instanceKlassHandle klass(THREAD, java_lang_Class::as_klassOop(mirror));
  methodOop m = klass->method_with_idnum(slot);
  if (m == NULL) {
    THROW_MSG_0(vmSymbols::java_lang_InternalError(), "invoke");
  }
  methodHandle method(THREAD, m);
  assert(method->name() == vmSymbols::object_initializer_name(), "invalid constructor");

  // Make sure klass gets initialize
  klass->initialize(CHECK_NULL);

  // Create new instance (the receiver)
  klass->check_valid_for_instantiation(false, CHECK_NULL);
  Handle receiver = klass->allocate_instance_handle(CHECK_NULL);

  // Ignore result from call and return receiver
  // invoke 方法最终使用
  // JavaCalls::call(&result, method, &java_args, THREAD);
  // 调用使用 java代码 实现的构造函数
  invoke(klass, method, receiver, override, ptypes, T_VOID, args, false, CHECK_NULL);
  return receiver();
}
```

调用 klass->allocate_instance_handle 创建新对象实例。 allocate_instance_handle 内部使用 allocate_instance 来创建对象。

所以，对于一个 java 对象，最终是由其所对应的类型的 Klass 的 allocate_instance 方法来完成内存分配的。

## $. 参考
1. [Dangerous Code: How to be Unsafe with Java Classes & Objects in Memory](http://zeroturnaround.com/rebellabs/dangerous-code-how-to-be-unsafe-with-java-classes-objects-in-memory/)
2. [Getting Started with HotSpot and OpenJDK](https://www.infoq.com/articles/Introduction-to-HotSpot)
3. [源码分析：Java对象的内存分配](http://www.cnblogs.com/iceAeterNa/p/4877741.html)
4. [HotSpotVM 对象机制实现浅析](http://blog.csdn.net/kisimple/article/details/44632443)
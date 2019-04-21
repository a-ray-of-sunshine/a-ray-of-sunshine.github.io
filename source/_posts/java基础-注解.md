---
title: java基础-注解
date: 2017-1-9 10:23:37
---

## 注解的类型

> An annotation type declaration is a special kind of interface declaration. To distinguish an annotation type declaration from an ordinary interface declaration, the keyword interface is preceded by an at-sign (@).
>
> **Note that the at-sign (@) and the keyword interface are two distinct tokens.** Technically it is possible to separate them with whitespace, but this is discouraged as a matter of style.

由此可知声明一个注解其实就是声明一个接口。

``` java
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
	
	String value() default "testdefault";
	
	int test();

}
```

使用命令 `javap -v MyAnnotation.class` 对上面的 class 文件进行反编译：`public interface MyAnnotation extends java.lang.annotation.Annotation`

由此可知，注解是一个隐式继承自 `java.lang.annotation.Annotation` 的接口。

## 注解在class文件中的存储

源代码中的注解在编译的时候，将其编译成对应 Element 的一个属性。

``` java
@MyAnnotation(value1 = "test", test=9527)
public @SuppressWarnings("rawtypes") class MyClass{
	
	public void test(){
		System.out.println("test");
	}
}
```

例如对于类 `MyClass` 来说，它有多个属性，其中 public 就是其一个属性，表示这个类的访问权限。同样地注解 @MyAnnotation 也是其一个属性。语法意义是相同的。所以 `public` 和 `@MyAnnotation` 的位置可以任意对调。

编译后的二进制结构，如下：

java源代码被编译成二进制的 class 文件，其中的各个 Elements 被编译成不同的二进制结构。例如： ClassFile, field_info, and method_info structure 这些结构中都有一个 attributes table。

其中注解将作为一种 attribute 被存储在上面三种结构的 attributes table 中。

注解的attribute的结构如下：

``` js
RuntimeVisibleAnnotations_attribute {
	// 属性名称在常量池中的索引
	// 其值是： "RuntimeVisibleAnnotations"
    u2         attribute_name_index;
    // 属性的长度
    u4         attribute_length;
    // 注解的个数
    u2         num_annotations;
    // 注解数组
    annotation annotations[num_annotations];
}

annotation {
	// 常量池中的索引，指向当前这个注解结构所
	// 代表的注解的类型。应该就是注解的类名。
    u2 type_index;
    // 表示下面键值对数组的个数
    u2 num_element_value_pairs;
    // 当前注解的所有键值对。
    {   // 键名索引
    	u2            element_name_index;
    	// 键值
        element_value value;
    } element_value_pairs[num_element_value_pairs];
}

element_value {
	// 值类型。
    u1 tag;
    
    // union 类型的数据，其具体的含义取决于
    // tag 的值。
    union {
    	// 1. 表示常量池索引  
    	// if the tag item is one of B, C, D, F, I, J, S, Z, or s.
        u2 const_value_index;

		// 2. 表示枚举常量
		// tag == e
        {   u2 type_name_index; // 枚举类型索引
            u2 const_name_index;// 枚举值索引
        } enum_const_value;

		// 3. 表示 class 类型
		// tag == c
        u2 class_info_index;

		// 4. 表示是一个注解
		// tag == @ 一个嵌套的注解
        annotation annotation_value;

		// 5. 表示是一个数组
		// tag == [
        {   u2            num_values;
            element_value values[num_values];
        } array_value;
    } value;
}
```

关于注解的静态结构的解析，其源码在`sun.reflect.annotation.AnnotationParser.parseMemberValue`。

## 注解的动态结构

通过注解的名称可以获得注解的 Class 对象。同时，注解的值，就是键值对也可以获得到，键名就是注解中声明的方法名称。

此时就可以通过动态代理创建注解的实例了。

``` java
// sun.reflect.annotation.AnnotationParser
public static Annotation annotationForMap(
    Class<? extends Annotation> type, Map<String, Object> memberValues)
{
    return (Annotation) Proxy.newProxyInstance(
        type.getClassLoader(), new Class[] { type },
        new AnnotationInvocationHandler(type, memberValues));
}

// AnnotationInvocationHandler.invoke 方法
public Object invoke(Object proxy, Method method, Object[] args) {
    String member = method.getName();
    Class<?>[] paramTypes = method.getParameterTypes();

    // Handle Object and Annotation methods
    // 处理 Object 类和 Annotation 接口的中的方法
    if (member.equals("equals") && paramTypes.length == 1 &&
        paramTypes[0] == Object.class)
        return equalsImpl(args[0]);
    assert paramTypes.length == 0;
    if (member.equals("toString"))
        return toStringImpl();
    if (member.equals("hashCode"))
        return hashCodeImpl();
    if (member.equals("annotationType"))
        return type;

    // Handle annotation member accessors
    // 处理 注解中定义的成员。
    Object result = memberValues.get(member);

    if (result == null)
        throw new IncompleteAnnotationException(type, member);

    if (result instanceof ExceptionProxy)
        throw ((ExceptionProxy) result).generateException();

    if (result.getClass().isArray() && Array.getLength(result) != 0)
        result = cloneArray(result);

    return result;
}
```

## 注解的生命周期

JDK提供了一个 meta-annotation 来标识一个注解的生命周期

`@Retention` 其具体的值可以在以下几种：

* SOURCE 

	这种类型的注解，编译器直接将其丢弃，在 class 文件中不会存储。

* CLASS

	编译器将为注解所在的 element 生成一个 RuntimeInvisibleAnnotations 的属性。这个属性的结构和 RuntimeVisibleAnnotations 的结构完全相同。
	
	> The RuntimeInvisibleAnnotations attribute is similar to the RuntimeVisibleAnnotations attribute, except that the annotations represented by a RuntimeInvisibleAnnotations attribute must not be made available for return by reflective APIs
	
	也就是说，这种注解存储在 class 文件中，但无法通过反射的 API 来获取到。

* RUNTIME

	编译器为这种类型的注解生成一个 RuntimeVisibleAnnotations 的属性。将其存储在 class 文件中。可以通过反射的 API 获取到。

## 总结

当创建一个注解时，就是创建一个隐式继承自 `java.lang.annotation.Annotation` 的接口。

当在Class， Methods，Filed 上使用注解的时候，注解的信息在编译的时候，将保存成对应的 Element 的一个 Attribute。

当使用反射 API 获取注解的时候，反射 API 内部，将通过编译时保存的 Attribute。创建一个基于 注解接口 的动态代理对象。可以通过这个代理对象获得注解上的值。

注解是全局的，静态的元数据。

## $. 参考
1. [Annotation Types](http://docs.oracle.com/javase/specs/jls/se7/html/jls-9.html#jls-9.6)
2. [java注解是怎么实现的](https://www.zhihu.com/question/24401191)
3. [Java annotation的实例是什么类的](http://rednaxelafx.iteye.com/blog/1148983)
4. [C# attribute和Java annotation](http://rednaxelafx.iteye.com/blog/464889)
5. [The RuntimeVisibleAnnotations attribute](http://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html#jvms-4.7.16)
6. [java Tutorials - Annotations](http://docs.oracle.com/javase/tutorial/java/annotations/index.html)
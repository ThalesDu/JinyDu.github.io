---
layout: post
title: Objc Runtime - 类对象
date: 2017-02-18
categories: blog
tags: [iOS,Objc-Runtime]
---
本节讨论的问题：
* 类对象的结构
* 编译之后类对象变成了什么样？
* id类型是什么；
* 运行期系统如何知道某个对象的类型？
* id，NSObject *, id<NSObject>三者的区别和联系。

## is a指针

Objc的对象类型并非在编译期就绑定好了，而是要在运行期查找。一般情况下，我们会指明接受信息的对象类型，这样做如果出现该对象无法响应
的消息，编译器可以产生警告信息⚠️。
而id类型的对象则不然，编译器假定他们响应所有的消息。
编译器无法得知某个对象到底能够响应多少种`select`，因为在运行时可以动态的添加。但是，即使使用了动态增加的技术，编译器还是希望在某个头文件中找到原型方法的定义，据此可以了解完整的方法签名。并生成派发消息所需要的正确代码。

`运行期的检视对象类型` 这一个操作也叫做`类信息查询（内省）`，这个特性内置在`Fundation`框架中的`NSObject`协议中-凡是继承自`NSObject`或`NSProxy`都遵循这个协议。

每个Objc对象都是指向某个内存地址的指针。

```objc
NSString *str = @"This is String";//<1>
id str = @"This is String";      //<2>
```
对于<1>、<2>的区别，只是编译器会检查<1>的方法是否存在。
若想要将对象分配到栈上：

```objc
String str = @"This is String"
//error :interface type cannot be statically allocated
```

对于id的本质呢是一个指向`objc_object`这个`struct`的指针，
在`<objc/runtime>`中定义了这个结构。

``` objc
typedef struct objc_class *Class;
typedef struct objc_object *id;

struct objc_object {
    Class isa                   OBJC_ISA_AVAILABILITY;
    ..
};
```
所以对于每一个`id`对象，其指向的结构体的首个成员就是`Class`变量，这个变量定义了对象所属与哪个类。通常称作`is a`指针。e.g.刚才的的例子中，`str`是一个`is a NSString`,

``` objc
typedef struct objc_class *Class;
struct objc_class : objc_object {
// inheir isa
    Class superclass;
    cache_t cache;
    class_data_bits_t bits;

    class_rw_t *data() { 
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }
};
```
`objc_class` 这个结构中存放了类的`元数据（meta data）`，而他从`objc_object`中继承的过来的`isa`指针,我们稍后再解析，我们主要聚焦`date()`这个方法上，这个方法返回一个`class_rw_t`结构体的指针。源码如下：

```objc
// objc-runtime-new.h
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;
}
```
 而`class_rw_t`结构又包含`class_ro_t`的结构体的指针变量，在`class_ro_t`中包含了，诸如：`method_list_t`一个类的方法列表，`protocol_list_t`协议列表,`ivar_list_t`实例变量列表，`property_list_t`属性列表。源码如下：
 
 ```objc
 // objc-runtime-new.h
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif
    const uint8_t * ivarLayout;
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
 ```

## 推演

到此我们可以得出一个结论，每一个实例对象都从`NSObject`中继承了一个叫做`isa`的指针变量。指向一个`obj_class` 的结构体。结构体中可以的：

* `superClass`父类；
* `cache`缓存
* `method_list_t`方法列表
* `protocol_list_t`协议列表
* `ivar_list_t`实例变量列表
* `property_list_t`属性列表。

根据上面所说的我们写出下面的伪代码：

```objc
struct __class_t{
    struct __class_t *isa;
    struct __class_t *superclass;
    void *cache;
    struct __class_ro_t *data;
}
struct __class_ro_t{
    unsigned int flags;
    unsigned int instanceStart;
    unsigned int instanceSize;
    unsigned int reserved;
    const unsigned char *ivarLayout;
    const char *name;
    const struct _method_list_t *baseMethods;
    const struct _objc_protocol_list *baseProtocols;
    const struct _ivar_list_t *ivars;
    const unsigned char *weakIvarLayout;
    const struct _prop_list_t *properties;
}

```
假设有2个`Person`的实例，`personA`和`personB`，这2个实例都有一个isa指针了;指向的是同一个`__class_t`的结构体。这个`__class_t`的结构体在**编译**的时候产生，包含了对象具有的：`method_list_t`,`protocol_list_t`、`ivar_list_t`, `property_list_t`这些信息。
调用放啊的时候，比如`[personA hello]`,通过`personA `的`isa`变量在`__class_t`结构体中找对应的方法实现，`[personB hello]` 也是如此的过程。（消息的传递机制）。

## 元类（MetaClass）

现在还存在2问题：

* 类如何去响应类方法；
* 在`__class_t`中还存在一个`isa`的这个变量的用途是什么呢？

理解类方法的需要首先理解`metaClass`的含义，`meta-`这个词意思是`关于……的……`,如`meta-data`是关于数据的数据。`meta-class`是关于类的类。

理解第二个概念：类也是实例对象。在 Java 这个面向对象的语言中，personA 和 personB 都是 Person 这个类的实例对象，而 Person 这个类也是一个实例对象，是 Class 这个类的实例对象。对于另外的一个类 Animal ，animalA和animalB都是Animal的实例。而Animal类**也是**Class类的实例。

在Objc中，也有类似的概念：

* Person 这个类是`OBJC_METACLASS_Person`这个元类的实例对象。
* Animal 这个类事`OBJC_METACLASS_Animal`这个元类的实例对象。

区别就是在 Java 中 Class 类已经被预先创建。而在 Objc中元类`OBJC_METACLASS_Person` 和 `OBJC_METACLASS_Animal` 是在编译期由编译器创建；并且也转化为一个`__class_t`的结构体类型。
而这个`__class_t`的结构体变量也被称作元类对象`MetaClass Object`；在其中也可以查到`method_list_t`,`ivar_list_t`等等；而这里的 method 就是类方法。

梳理一下：
存在某个类的实例`Instance`它包含一个`isa`指针,这个指针指向它的类结构`Class`,可以查询到他的实例方法列表。`Class`这个结构体也包含一个`isa`指针，指向`MetaClass`，可以查询类方法

**类对象 `Class Object` 表明了这个类的所具有的实例变量，实例方法、遵循的协议等等信息**<br>
**元类对象 `MetaClass Object` 表明这个类所具有的类方法等信息。**
![objctree-w600](/assets/resources/objCRuntime/objctree.png)


所以isa不会是nil的；只要有一个 id 类型的对象，runtime都可以访问首地址偏移`isa`拿到该对象的信息。
绿色的线时响应消息的过程。如果没有找到会沿着`superClass`一直向上找。
##使用Clang观察rewrite之后文件。
在工程中调用 `clang -rewrite-objc person.m`, 会在工程中生成一个`person.cpp`的文件,在最后可以这些信息。

## NSObject*  |  id 之间的区别以及联系

写到这里有一个疑问：id既然可以表示所有 Objc 类的对象，NSObject又是Objc中类的基类 (一般情况下)，那他们有什么区别呢。

我们引入3个东西来比较：

```objc
id foo1;
NSObject *foo2;
id<NSObject> *foo3;
```

* id:
     它可以指向的类型不仅限于NSObject
    id 我们经常看到运用的地方莫过于
    @property (nonatomic, weak, nullable) id delegate;
    `id`对象对于编译器来说只意味着一个指向对象的指针，具体指向什么对象编译器并不知道，所以向id对象发送任何消息编译器都不会报错。
    
    ```objc
    @interface Foo:Objcet
    @end
    @implementation Foo
    @end
    
    id foo = [[Foo alloc] init];
    [foo say];
    ```
其中alloc方法返回的就是一个 id 对象所以无论发什么消息都不会报错。
* NSObject*
    而id指向的对象不都是NSObject的子类，e.g.`NSProxy`。
    **当我们指定一个类是NSObject的实例的时候，如果我们调用的方法没有在`NSObjcet`中定义编译器就会果断报错**。
    
* id<NSObjcet>
    id<NSObject> 相当于告诉编译器，你不需要知道我是什么对象，你只需要知道我遵循了`<NSObject>`协议，编译器就能保证赋值给id<NSObject>的对象都是遵循`<NSObject>`。而对于`NSObject`、`NSProxy`都遵循了`<NSObject>`协议。保证了接口的统一性。
###id & instancetype
注意：引用Clang中的一句话。

>"instancetype is a contextual keyword that is only permitted in the result type >of an Objective-C method"
>
>http://clang.llvm.org/docs/LanguageExtensions.html Clang Language Extensions

如上所说 `instancetype` 只是一个上下文环境的关键字。编译器有了它就可以有足够的类型信息检查某个类是否可以响应某些方法；








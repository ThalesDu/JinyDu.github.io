---
layout: post
title: Objc Runtime（2） - 消息的传递机制
date: 2017-02-18
categories: blog
tags: [iOS,Objc-Runtime]
---

本节我们讨论的问题：

* 消息的发送；
* 方法缓存；
* 消息的转发机制；


## 消息的发送
在 Objc 中 调用一个类的方法，是形如`[personA say];`这样的形式被称为发消息，可以看作`[receiver message]`，`personA`消息的接受者，而`say`是消息。实际上呢，`[receiver message]`这样的代码，会转换成`objc_msgSend(receive,@select(message));`的形式去调用。
可以通过 `clang -rewrite-objc main.m`查看：

```objc
//转换前
[personA say];
//转换后
((void (*)(id, SEL))(void *)objc_msgSend)((id)personA, sel_registerName("say"));
```

在`message.h`中可以看到`objc_msgSend`的定义

```objc
// message.h
#if !OBJC_OLD_DISPATCH_PROTOTYPES
OBJC_EXPORT void objc_msgSend(void /* id self, SEL op, ... */ )
   __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
OBJC_EXPORT void objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )
   __OSX_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
#else
```
可以看到`objc_msgSend`前面两个参数一个是消息的接受者`id`类型，一个是`SEL`,之后是个可变参数`...`。
Runtime维护着一张表，将相同字符的方法名Map到唯一一个`SEL`,通过`sel_registerName(char *name);`可以查到一个字符串所对应的选择子。
Apple提供了一个语法糖（编译器保留字）`@select()`方便调用此函数。 

在 Objc中有着SEL的定义：

``` objc
/// An opaque type that represents a method selector.
typedef struct objc_selector *SEL;
```
这个`objc_selector`是个什么类型呢？在Runtime源码中找不到定义，注释也写了这个是一个不透明的type，经过如下验证：

```objc
printf("%s",@select(isEqual:));
//console output : isEqual;
```
我们将`SEl`可以理解为一个字符串
还有一个关键字我们需要提及,那就是`IMP`，`IMP`的定义如下：

``` objc
//objc.h
/// A pointer to the function of a method implementation. 
#if !OBJC_OLD_DISPATCH_PROTOTYPES
typedef void (*IMP)(void /* id, SEL, ... */ ); 
#else
typedef id (*IMP)(id, SEL, ...); 
#endif
```
我们可以看到IMP实际上是一个函数指针，记录着函数执行的入口地址；objc 方法调用实际最后被转换成了 C 的函数调用而入口就是这个IMP；

在 Objc.h 中我们找到了`struct objc_method`以及`struct objc_method_list`的定义

```objc
struct objc_method {
    SEL method_name                                          
    char *method_types                                       
    IMP method_imp                                           
}                                                            

struct objc_method_list {
    struct objc_method_list *obsolete                        

    int method_count                                         
#ifdef __LP64__
    int space                                                
#endif
    /* variable length structure */
    struct objc_method method_list[1]                        
}                                                            
```
可以看到`objc_method`将一个`SEL method_name`Map到一个`IMP method_imp`而在`objc_class`-->`class_rw_t`-->`class_ro_t`->中的`method_list_t * baseMethodList`
存储了method列表；

##objc_msgSend 的具体实现；
`objc_msgSend`具体实现由汇编写成，在不同平台由不同的实现，objc-msg-arm.s、objc-msg-arm64.s、objc-msg-i386.s、objc-msg-simulator-i386.s、objc-msg-simulator-x86_64.s、objc-msg-x86_64.s，这里摘取的是 objc-msg-arm.s

``` .s
ENTRY _objc_msgSend
MESSENGER_START
	
cbz	r0, LNilReceiver_f  //check r0 is nil? nil run LNilReceiver_f : continue

ldr	r9, [r0]		// r9 = self->isa
GetClassFromIsa			// r9 = class
CacheLookup NORMAL
// cache hit, IMP in r12, eq already set for nonstret forwarding
MESSENGER_END_FAST
bx	r12			// call imp

CacheLookup2 NORMAL
// cache miss
ldr	r9, [r0]		// r9 = self->isa
GetClassFromIsa			// r9 = class
MESSENGER_END_SLOW
b	__objc_msgSend_uncached

```
从上述代码中可以看到，objc_msgSend（就arm平台而言）的消息分发分为以下几个步骤：

* 检查receiver是否为nil，也就是objc_msgSend的第一个参数self，也就是要调用的那个方法所属对象
* 从缓存里寻找，找到了则分发，否则
* 执行__objc_msgSend_uncached；
* 缓存的具体实现稍后在细说。

``` .s
STATIC_ENTRY __objc_msgSend_uncached

// THIS IS NOT A CALLABLE C FUNCTION
// Out-of-band r9 is the class to search
	
MethodTableLookup NORMAL	// returns IMP in r12
bx	r12

END_ENTRY __objc_msgSend_uncached
	
```
```.s
/////////////////////////////////////////////////////////////////////
//
// MethodTableLookup	NORMAL|STRET
//
// Locate the implementation for a selector in a class's method lists.
//
// Takes: 
//	  $0 = NORMAL, STRET
//	  r0 or r1 (STRET) = receiver
//	  r1 or r2 (STRET) = selector
//	  r9 = class to search in
//
// On exit: IMP in r12, eq/ne set for forwarding
//
/////////////////////////////////////////////////////////////////////
	
	.macro MethodTableLookup
	
	stmfd	sp!, {r0-r3,r7,lr}
	add	r7, sp, #16
	sub	sp, #8			// align stack
	FP_SAVE

.if $0 == NORMAL # receiver already in r0selector already in r1

.else
	mov 	r0, r1			# receiver
	mov 	r1, r2			# selector
.endif
	mov	r2, r9			# class to search

	blx	__class_lookupMethodAndLoadCache3
	mov	r12, r0			# r12 = IMP
	
.if $0 == NORMAL
	cmp	r12, r12		# set eq for nonstret forwarding
.else
	tst	r12, r12		# set ne for stret forwarding
.endif

	FP_RESTORE
	add	sp, #8			# align stack
	ldmfd	sp!, {r0-r3,r7,lr}

.endmacro
```
* 利用objc-class.mm中_class_lookupMethodAndLoadCache3方法去寻找selector;
* _class_lookupMethodAndLoadCache3
    * 从本class的method list寻找selector，如果找到，填充到缓存中，并返回selector，否则
    * 寻找父类的method list，并依次往上寻找，直到找到selector，填充到缓存中，并返回selector，否则
    * 调用_class_resolveMethod，如果可以动态resolve为一个selector，不缓存，方法返回，否则
    * 转发这个selector，否则
* 成功返回一个IMP否则报错，抛出异常。

而`_class_lookupMethodAndLoadCache3`方法反悔了一个`lookUpImpOrForward()`的一个C函数,而这个函数是整个 Runtime 消息机制中非常重要的一环，代码如下：

```objc
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    Class curClass;
    IMP imp = nil;
    Method meth;
    bool triedResolver = NO;

    runtimeLock.assertUnlocked();

    // Optimistic cache lookup
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }

    if (!cls->isRealized()) {
        rwlock_writer_t lock(runtimeLock);
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        _class_initialize (_class_getNonMetaClass(cls, inst));
        // If sel == initialize, _class_initialize will send +initialize and 
        // then the messenger will send +initialize again after this 
        // procedure finishes. Of course, if this is not being called 
        // from the messenger then it won't happen. 2778172
    }

    // The lock is held to make method-lookup + cache-fill atomic 
    // with respect to method addition. Otherwise, a category could 
    // be added but ignored indefinitely because the cache was re-filled 
    // with the old value after the cache flush on behalf of the category.
 retry:
    runtimeLock.read();

    // Try this class's cache.

    imp = cache_getImp(cls, sel);
    if (imp) goto done;

    // Try this class's method lists.

    meth = getMethodNoSuper_nolock(cls, sel);
    if (meth) {
        log_and_fill_cache(cls, meth->imp, sel, inst, cls);
        imp = meth->imp;
        goto done;
    }

    // Try superclass caches and method lists.

    curClass = cls;
    while ((curClass = curClass->superclass)) {
        // Superclass cache.
        imp = cache_getImp(curClass, sel);
        if (imp) {
            if (imp != (IMP)_objc_msgForward_impcache) {
                // Found the method in a superclass. Cache it in this class.
                log_and_fill_cache(cls, imp, sel, inst, curClass);
                goto done;
            }
            else {
                // Found a forward:: entry in a superclass.
                // Stop searching, but don't cache yet; call method 
                // resolver for this class first.
                break;
            }
        }

        // Superclass method list.
        meth = getMethodNoSuper_nolock(curClass, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
            imp = meth->imp;
            goto done;
        }
    }

    // No implementation found. Try method resolver once.

    if (resolver  &&  !triedResolver) {
        runtimeLock.unlockRead();
        _class_resolveMethod(cls, sel, inst);
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES;
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.

    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlockRead();

    return imp;
}
```
`lookUpImpOrForward`主要做的以下几个工作：

* `if (!cls->isRealized())`来判断是否调用relizeClass函数，
* 根据 `cls->isInitialized` 来判断是不是初始化了过了，也就是类被首次使用的时候`initialize`方法是需要被调用一次的。
* ignoreSelector 跳过一些特定的方法。
* 接下来去缓存找，找到返回 IMP，找不到接着去调用 log_and_fill_cache 函数，会去 class 的方法列表查找，找到会加入缓存列表然后返回 IMP。
* 自己的列表找不到，就去基类的方法列表中寻找。
* 都找不到则 `_class_resolveMethod`被调用，进入消息动态处理，转发阶段。

## 消息的动态处理、转发
如果一个方法 `id` 没有找到会发生什么？通常程序会Crash 并抛出 `unrecognized selector send to instance`的异常。但是在抛出异常之前有3次拯救的机会。

1. 在`__class_resloveMethod(Class cls, SEL sel, id inst)` 通过`cls->isMetaClass()`判断要添加的方法是类方法还是实例方法。调用`_class_resolveInstanceMethod(cls, sel, inst);` 或`_class_resolveInstanceMethod(cls, sel, inst);`来sendMethod `[cls resolveInstanceMethod:sel]`或者`[cls resolveClassMethod:sel]`如果 `[cls resolveInstanceMethod:sel]`或者`[cls resolveClassMethod:sel]`返回了YES；则重启`methodSend`过程.
2. resolveInstanceMethod:sel]`或者`[cls resolveClassMethod:sel]`返回了NO,则进入到下一个阶段：消息转发：(Message Forwarding)，这里需要`receiver`实现`-forwardingTargetForSelector:`方法， runtime会调用这个方法将消息转发给其他对象。

```objc 
- (id)forwardingTargetForSelector:(SEL)aSelector
{
    if(aSelector == @selector(foo:)){
        return alternateObject;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```
3. `- (id)forwardingTargetForSelector:(SEL)aSelector`方法默认返回nil，紧接着`-methodSignatureForSelector` 方法得到调用，该方法被`CoreFundation`替换，不过会返回一个`NSMethodSignature`对象，**封装了原始函数的的调用参数，返回值类型等**。紧接着这个`NSMethodSignature`对象又被封装成了一个`NSInvocation`对象，然后 `-forwordInvocation:`方法被调用。这个`NSInvocation`对象被传递给该方法，然后抛出异常，`-forwordInvocation:`默认的实现如下：

```objc
// NSObject.mm
// 默认实现
- (void)forwardInvocation:(NSInvocation *)invocation {
    [self doesNotRecognizeSelector:(invocation ? [invocation selector] : 0)];
}

// Replaced by CF (throws an NSException)
- (void)doesNotRecognizeSelector:(SEL)sel {
    _objc_fatal("-[%s %s]: unrecognized selector sent to instance %p", 
                object_getClassName(self), sel_getName(sel), self);
}
```







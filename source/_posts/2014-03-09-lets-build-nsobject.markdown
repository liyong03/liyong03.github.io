---
layout: post
title: "动手创建 NSObject"
date: 2014-03-09 00:16:54 +0800
comments: true
categories: ios programming
---
本文译自[这里](https://www.mikeash.com/pyblog/friday-qa-2013-01-25-lets-build-nsobject.html)，转载请注明！

在 Cocoa 编程时创建和使用的（几乎）所有类都是以 NSObject 为根基，但它背后都做了什么、是怎么做的呢？今天，我将从零开始创建 NSObject。

##根类(root class)的组成
准确地说，根类到底做了什么？以 Objective-C 自身的术语描述，有一个准确的要求：根类的第一个实例变量必须是 isa，它指向一个实例(object)的类(class)，在分发消息的时候，它被用来准确找出该实例的类。从严格的编程语言观点来看，这是根类必须要做的。

当然根类只提供这些是不够的，NSObject 提供了更多的。它提供的功能可以分为一下三类：

1. **内存管理**：标准的内存管理方法如 *retain* 和 *release* 都是在 NSObject 中实现的，*alloc* 方法也是如此。
2. **内省**：NSObject 提供了很多 Objective-C 运行时方法的包装，例如 *class*、*respondsToSelector:* 和 *isKindOfClass:*。
3. **内置实用方法**：有很多每一个实例都需要实现的方法，例如 *isEqual:* 和 *description*。为了保证每一个实例都有这些方法，NSObject 提供了这些方法的默认实现。

##代码
我接下来将要在 MAObject 中实现 NSObject 的功能，源代码在 Github: <https://github.com/mikeash/MAObject>

这个代码没有使用 ARC，虽然 ARC 是个好东西，而且应该尽可能使用它，不过在实现根类的时候使用就不太合适，因为根类需要实现内存管理，如果使用 ARC 就把内存管理交给编译器了！

###实例变量
MAObject 有两个实例变量，第一个是 isa 指针，第二个是实例的引用计数[^1]：

[^1]:NSObject 的实现有些不同，具体可以参考实现源码：<http://opensource.apple.com/source/objc4/objc4-532.2/runtime/NSObject.mm>

``` objc
    @implementation MAObject {
        Class isa;
        volatile int32_t retainCount;
    }
```

我们将使用 OSAtomic.h 中的方法管理引用计数保证线程安全，这就是为什么这里没有用 NSUInteger 或者类似的声明，而是用了个看起来不寻常的方式。

实际上 NSObject 是把引用计数保存在外面。有一个全局的表（table），标中是实例地址和引用计数映射。这样可以节省内存，因为不在表中的实例的引用计数默认就是1。尽管有这些好处，但是这样实现起来有点复杂并且性能有些慢，所以在这里我就不实现这种方法了。

###内存管理
MSObject 需要做的第一件事情就是创建实例，主要在 +alloc 中实现完成。（这里我略过了已经弃用的 +allocWithZone: 方法，这个方法实际上会忽略参数，完成 +alloc 同样的工作。）

子类一般很少会重写 +alloc，而是依赖根类来分配内存。这就意味着 MAObject 不仅要为自己分配内存，还要为子类分配内存，完成这个就要利用**类方法中的 self 的值实际上就是消息将会被分发到的类**这一优势。例如代码为 **[SomeSubClass alloc]**，那么 self 就是指向 SomeSubClass。这个类将会被用来使用运行时方法确定需要的内存大小和正确设置 isa 。同时引用计数的值也会被设为1，与创建一个实例的行为相符：

``` objc
    + (id)alloc
    {
        MAObject *obj = calloc(1, class_getInstanceSize(self));
        obj->isa = self;
        obj->retainCount = 1;
        return obj;
    }
```

**retain** 方法很简单，就是使用 **OSAtomicIncrement32** 方法将引用计数加1，然后返回 self：
``` objc
    - (id)retain
    {
        OSAtomicIncrement32(&retainCount);
        return self;
    }
```

**release** 方法干的事情多一点，首先将引用计数减1，如果引用计数降为0了，那么实例需要被销毁，所以要调用 **dealloc**：

``` objc
    - (oneway void)release
    {
        uint32_t newCount = OSAtomicDecrement32(&retainCount);
        if(newCount == 0)
            [self dealloc];
    }
```

**autorelease** 的实现实际上调用 **NSAutoreleasePool** 将 self 加入当前的 autorelease pool 。Autorelease pool 现在时运行时的一部分，所以这样做不是很直接，但是 autorelease pool 的 API 是私有的，所以这样实现也是目前最好的办法了：

``` objc
    - (id)autorelease
    {
        [NSAutoreleasePool addObject: self];
        return self;
    }
```

**retainCount** 方法简单返回引用计数值：

``` objc
    - (NSUInteger)retainCount
    {
        return retainCount;
    }
```

最后是 **dealloc** 方法，通常类的 dealloc 方法清除所有的实例变量，然后调用父类的 dealloc 。所以根类实际上需要释放占用的内存。这里通过调用 free 来完成：

``` objc
    - (void)dealloc
    {
        free(self);
    }
```

还有一些辅助的方法。NSObject 为了一致性提供了一个什么也没做的 init 方法，所以子类通常会调用 [super init]：

``` objc
    - (id)init
    {
        return self;
    }
```

还有一个 new 方法，它只是包装了一下 alloc 和 init ：

``` objc
    + (id)new
    {
        return [[self alloc] init];
    }
```

还有个空的 finalize 方法。NSObject 把它作为垃圾回收的一部分实现了。不过 MAObject 开始就不支持垃圾回收，不过我在这里加上它只是因为 NSObject 有它：

``` objc
    - (void)finalize
    {
    }
```

###内省
很多内省的方式只是运行时方法的包装，因为这没太大意思，所以我会简单介绍一下运行时方法背后的工作原理。

最简单的内省方法就是 **class** ，它只是返回 isa ：

``` objc
    - (Class)class
    {
        return isa;
    }
```

从技术上来讲，这样的实现在 isa 是一个标记指针(tagged pointers)[^2] 时就会发生错误，更合理的实现应该是调用 object_getClass 方法，它在标记指针时也可以正常工作。

[^2]:详情请参考<http://en.wikipedia.org/wiki/Tagged_pointer>和<https://www.mikeash.com/pyblog/friday-qa-2012-07-27-lets-build-tagged-pointers.html>

实例方法 superclass 和使用类来调用 superclass 的行为一样，我们也是这么实现的：

``` objc
    - (Class)superclass
    {
        return [[self class] superclass];
    }
```

有很多类方法也是内省的一部分，+class 直接返回 self ，实际上它时一个类实例(class object)。这里有些奇怪，但实际上 NSObject 就是这么工作的。[object class] 返回的是实例的类，[MyClass class] 返回的指针指向 MyClass 本身，看起来不一致，实际上 MyClass 也有一个类，他就是 MyClass 的元类(meta class) 。下面是实现：

``` objc
    + (Class)class
    {
        return self;
    }
```

+superclass 方法顾名思义，它是通过调用 class_getSuperclass 实现的，该方法会解析运行时的类结构来找出指向父类的指针：

``` objc
    + (Class)superclass
    {
        return class_getSuperclass(self);
    }
```

还有一些方法用来查询某个实例的类是否与指定的类相匹配，最简单的一个就是 isMemberOfClass: ，该方法进行严格匹配，会忽略子类，实现如下：

``` objc
    - (BOOL)isMemberOfClass: (Class)aClass
    {
        return isa == aClass;
    }
```

isKindOfClass: 方法会检查子类，所以 [subclassInstance isKindOfClass: [Superclass class]] 会返回 YES 。该方法的返回值与 +isSubclassOfClass: 的返回值完全一样，我们也是通过调用它来实现的：

``` objc
    - (BOOL)isKindOfClass: (Class)aClass
    {
        return [isa isSubclassOfClass: aClass];
    }
```

+isSubclassOfClass: 方法有点意思，它会从 self 开始向上递归，在每一层比较目标类。如果找到相匹配的，就返回 YES ，如果一直找到类继承机构的顶层也没有匹配的，就返回 NO ：

``` objc
    + (BOOL)isSubclassOfClass: (Class)aClass
    {
        for(Class candidate = self; candidate != nil; candidate = [candidate superclass])
            if (candidate == aClass)
                return YES;
        return NO;
    }
```
你可能已经注意到了这里不是很高效，如果你对一个处在很深的继承关系中的类调用该方法，它在返回 NO 之前将会进行很多次循环。因此 isKindOfClass: 会比通常的消息发送慢，在某种情况下会是性能瓶颈，这也是需要避免使用这类方法的原因之一。

**respondsToSelector:** 方法只是调用运行时方法 **class_respondsToSelector** ，该方法在类的方法表中依此查找看是否有匹配项：

``` objc
    - (BOOL)respondsToSelector: (SEL)aSelector
    {
        return class_respondsToSelector(isa, aSelector);
    }
```

还有另外一个类方法 **instancesRespondToSelector:** 和上面的方法几乎一样，不过唯一的区别是它的实现传入的是 self 而不是 isa ，在上下文环境中 self 应该是元类(meta class)：

``` objc
    + (BOOL)instancesRespondToSelector: (SEL)aSelector
    {
        return class_respondsToSelector(self, aSelector);
    }
```

与此类似，也有两个 **conformsToProtocol:** 方法，一个是实例方法，另一个是类方法。他们也都是对运行时方法的包装，在这里是去遍历所有类遵循的协议(protocol)表，坚持给定的协议是否在其中：

``` objc
    - (BOOL)conformsToProtocol: (Protocol *)aProtocol
    {
        return class_conformsToProtocol(isa, aProtocol);
    }

    + (BOOL)conformsToProtocol: (Protocol *)protocol
    {
        return class_conformsToProtocol(self, protocol);
    }
```

下一个方法是 **methodForSelector:** ，类方法中类型的方法是 **instanceMethodForSelector:** 。他们俩都会调用运行时方法 **class_getMethodImplementation** ，该方法会查找类的函数表，然后返回响应的 IMP ：

``` objc
- (IMP)methodForSelector: (SEL)aSelector
    {
        return class_getMethodImplementation(isa, aSelector);
    }

    + (IMP)instanceMethodForSelector: (SEL)aSelector
    {
        return class_getMethodImplementation(self, aSelector);
    }
```

有趣的一点是 **class_getMethodImplementation** 总是返回一个 IMP ，即使参数是一个不存在的 selector 。当类没有实现某个方法时，它会返回一个特殊的转发 IMP ，这个 IMP 包装好了调用 **forwardInvocation:** 的消息参数。

方法 **methodSignatureForSelector:** 只是对类方法的包装：

``` objc
    - (NSMethodSignature *)methodSignatureForSelector: (SEL)aSelector
    {
        return [isa instanceMethodSignatureForSelector: aSelector];
    }
```

而这个类方法也是对运行时方法的包装。它首先获取输入 selector 的 Method 。如果不能得到，那么就表明类没有实现该方法，那么返回 nil 。否则就获取方法类型的 C 字符串表达，并且包装在 **NSMethodSignature** 中：

``` objc
    + (NSMethodSignature *)instanceMethodSignatureForSelector: (SEL)aSelector
    {
        Method method = class_getInstanceMethod(self, aSelector);
        if(!method)
            return nil;

        const char *types = method_getTypeEncoding(method);
        return [NSMethodSignature signatureWithObjCTypes: types];
    }
```

最后是 **performSelector:** 方法，还有两个类似方法用 **withObject:** 去接收参数。他们不是严格意义上的内省方法，但他们都是对底层运行方法的包装。他们只是获取 selector 的 IMP ，强制转换成合适的函数指针类型，然后调用：

``` objc
    - (id)performSelector: (SEL)aSelector
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL))imp)(self, aSelector);
    }

    - (id)performSelector: (SEL)aSelector withObject: (id)object
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL, id))imp)(self, aSelector, object);
    }

    - (id)performSelector: (SEL)aSelector withObject: (id)object1 withObject: (id)object2
    {
        IMP imp = [self methodForSelector: aSelector];
        return ((id (*)(id, SEL, id, id))imp)(self, aSelector, object1, object2);
    }
```

###内置实用方法
MAObject 提供了很多方法的默认实现，我们从 **isEqual:** 和 **hash** 两个方法开始，因为它们都是用实例的指针进行唯一性判断：

``` objc
    - (BOOL)isEqual: (id)object
    {
        return self == object;
    }

    - (NSUInteger)hash
    {
        return (NSUInteger)self;
    }
```

任何子类想要实现更复杂的相等判断就要重写这些方法，但是如果子类想要只有和自身相等的情况下就可以使用这些方法。

另外一个方便的方法是 **description** ，我们也有一个默认实现。这个方法只是生成一个类似 **<MAObject: 0xdeadbeef>** 的字符串，包含了实例的类和指针：

``` objc
    - (NSString *)description
    {
        return [NSString stringWithFormat: @"<%@: %p>", [self class], self];
    }
```

类方法 **description** 只需要返回类的名字即可，所以调用运行时方法获取类名然后返回即可：

``` objc
    + (NSString *)description
    {
        return [NSString stringWithUTF8String: class_getName(self)];
    }
```

**doesNotRecognizeSelector:** 是一个知道的人比较少的实用方法。它会通过抛异常来让类看起来不会响应某些方法，这在我们创建一些子类必须重写的方法时很有用：

``` objc
    - (void)subclassesMustOverride
    {
        // pretend we don't actually implement this here
        [self doesNotRecognizeSelector: _cmd];
    }
```

代码很简单，唯一有些技巧的地方就是正确给出方法的名称，我们想为实例输出类似 **-[Class method]** 的东西，同时类方法前面需要显示 **+** ，类似 **+[Class classMethod]** 。为了分辨是哪种情况，我们就需要检查 isa 是不是元类，如果是元类，那么 self 就是类，需要显示 **+** 。否则 self 就是实例，需要用 **-** 。代码其他部分就是用来抛异常：

``` objc
    - (void)doesNotRecognizeSelector: (SEL)aSelector
    {
        char *methodTypeString = class_isMetaClass(isa) ? "+" : "-";
        [NSException raise: NSInvalidArgumentException format: @"%s[%@ %@]: unrecognized selector sent to instance %p", methodTypeString, [[self class] description], NSStringFromSelector(aSelector), self];
    }
```

最后，还有很多显而易见的方法（比如 self 方法），很多让子类安全调用 **super** 的方法（类似空的 **+initialize** 方法），很多重写点（例如 **copy** 就会抛异常）。这些都没太多意思，但都包括在了 MAObject 中：

``` objc
    - (id)self
    {
        return self;
    }

    - (BOOL)isProxy
    {
        return NO;
    }

    + (void)load
    {
    }

    + (void)initialize
    {
    }

    - (id)copy
    {
        [self doesNotRecognizeSelector: _cmd];
        return nil;
    }

    - (id)mutableCopy
    {
        [self doesNotRecognizeSelector: _cmd];
        return nil;
    }

    - (id)forwardingTargetForSelector: (SEL)aSelector
    {
        return nil;
    }

    - (void)forwardInvocation: (NSInvocation *)anInvocation
    {
        [self doesNotRecognizeSelector: [anInvocation selector]];
    }

    + (BOOL)resolveClassMethod:(SEL)sel
    {
        return NO;
    }

    + (BOOL)resolveInstanceMethod:(SEL)sel
    {
        return NO;
    }
```

##总结
NSObject 其实就是一大堆不同函数的组合，没什么奇怪之处。它的主要功能就是分配和管理内存，你可据此创建实例。同时它也提供了一堆每个方法都应该支持的有用的重写点，也包装了很多有用的运行时 API 。

这里我忽略了 NSObject 的一个块内容：key-value coding ，这块复杂到我需要另写[一篇文章](https://www.mikeash.com/pyblog/friday-qa-2013-02-08-lets-build-key-value-coding.html)了。

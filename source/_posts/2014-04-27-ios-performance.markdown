---
layout: post
title: "iOS App 性能备忘"
date: 2014-04-27 11:04:22 +0800
comments: true
categories: [programming, ios]
categories: 
---

本文译自[这里](https://github.com/danielamitay/iOS-App-Performance-Cheatsheet).

# iOS App 性能备忘

**本备忘收集了很多可以提高 iOS 中 Objective-C 代码性能的代码片段和配置**

这些文档中大部分代码片段和配置包括：将平时随手使用会用到的优先考虑灵活性而不是性能的高级 API，替换为功能相同的底层 API；一些会影响到绘制性能的类属性配置。对于 app 性能来说，好的架构和恰当的多线程总是很重要的，但是有时需要具体问题具体对待。

### 目录

**iOS App 性能备忘** 按照 Objective-C framework 组织为不同章节的 Markdown 文件：

- [**Foundation**](Foundation.md)
	- [NSDateFormatter](Foundation.md#nsdateformatter)
	- [NSFileManager](Foundation.md#nsfilemanager)
	- [NSObjCRuntime](Foundation.md#nsobjcruntime)
	- [NSString](Foundation.md#nsstring)
- [**UIKit**](UIKit.md)
	- [UIImage](UIKit.md#uiimage)
	- [UIView](UIKit.md#uiview)
- [**QuartzCore**](QuartzCore.md)
	- [CALayer](QuartzCore.md#calayer)


# Foundation

- [NSDateFormatter](#nsdateformatter)
- [NSFileManager](#nsfilemanager)
- [NSObjCRuntime](#nsobjcruntime)
- [NSString](#nsstring)

---

### NSDateFormatter

`NSDateFormatter` 不是唯一一个创建的开销就很昂贵的类，但是它却是常用的、开销大到 Apple 会特别建议应该缓存和重复使用实例的一个。

> Creating a date formatter is not a cheap operation. If you are likely to use a formatter frequently, it is typically more efficient to cache a single instance than to create and dispose of multiple instances. One approach is to use a static variable.

[Source](https://developer.apple.com/library/ios/documentation/cocoa/Conceptual/DataFormatting/Articles/dfDateFormatting10_4.html)

一种通用的缓存 `NSDateFormatter` 的方法是使用 `-[NSThread threadDictionary]`（因为 `NSDateFormatter` 不是线程安全的）：

```objective-c
+ (NSDateFormatter *)cachedDateFormatter {
	NSMutableDictionary *threadDictionary = [[NSThread currentThread] threadDictionary];
	NSDateFormatter *dateFormatter = [threadDictionary objectForKey:@"cachedDateFormatter"];
    if (dateFormatter == nil) {
        dateFormatter = [[NSDateFormatter alloc] init];
        [dateFormatter setLocale:[NSLocale currentLocale]];
        [dateFormatter setDateFormat: @"YYYY-MM-dd HH:mm:ss"];
        [threadDictionary setObject:dateFormatter forKey:@"cachedDateFormatter"];
    }
    return dateFormatter;
}
```

#####- (NSDate *)dateFromString:(NSString *)string

这可能是最常见的 iOS 性能瓶颈。经过多方努力寻找，下面是 [ISO8601](http://en.wikipedia.org/wiki/ISO_8601) 转成 `NSDate` 的 `NSDateFormatter` 的最著名替代品。

######strptime 

```objective-c
//#include <time.h>

time_t t;
struct tm tm;
strptime([iso8601String cStringUsingEncoding:NSUTF8StringEncoding], "%Y-%m-%dT%H:%M:%S%z", &tm);
tm.tm_isdst = -1;
t = mktime(&tm);
[NSDate dateWithTimeIntervalSince1970:t + [[NSTimeZone localTimeZone] secondsFromGMT]];
```

[Source](http://sam.roon.io/how-to-drastically-improve-your-app-with-an-afternoon-and-instruments)

######sqlite3

```objective-c
//#import "sqlite3.h"

sqlite3 *db = NULL;
sqlite3_open(":memory:", &db);
sqlite3_stmt *statement = NULL;
sqlite3_prepare_v2(db, "SELECT strftime('%s', ?);", -1, &statement, NULL);
sqlite3_bind_text(statement, 1, [iso8601String UTF8String], -1, SQLITE_STATIC);
sqlite3_step(statement);
int64_t value = sqlite3_column_int64(statement, 0);
NSDate *date = [NSDate dateWithTimeIntervalSince1970:value];
sqlite3_clear_bindings(statement);
sqlite3_reset(statement);
```

[Source](http://vombat.tumblr.com/post/60530544401/date-parsing-performance-on-ios-nsdateformatter-vs)


### NSFileManager

#####- (NSDictionary *)attributesOfItemAtPath:(NSString *)filePath error:(NSError *)error

当试图获取磁盘中一个文件的属性信息时，使用 `–[NSFileManager attributesOfItemAtPath:error:]` 会浪费大量时间读取你可能根本不需要的附加属性。这时你可以使用 `stat` 代替 `NSFileManager`，直接获取文件属性：

```objective-c
//#import <sys/stat.h>

struct stat statbuf;
const char *cpath = [filePath fileSystemRepresentation];
if (cpath && stat(cpath, &statbuf) == 0) {
    NSNumber *fileSize = [NSNumber numberWithUnsignedLongLong:statbuf.st_size];
    NSDate *modificationDate = [NSDate dateWithTimeIntervalSince1970:statbuf.st_mtime];
    NSDate *creationDate = [NSDate dateWithTimeIntervalSince1970:statbuf.st_ctime];
    // etc
}
```

### NSObjCRuntime

#####NSLog(NSString *format, ...)

`NSLog()` 写消息到 Apple 的系统日志。当通过 Xcode 变异运行程序时，被写出的日志会展现在调试终端，同事也会写到设备产品终端日志中。此外，系统会在主线程序列化 `NSLog()` 的内容。即使是最新的 iOS 设备，`NSLog()` 输出调试信息所花的时间也是无法忽略的。所以在产品环境中推荐尽可能少的使用 `NSLog()`。

> Calling `NSLog` makes a new calendar for each line logged. Avoid calling `NSLog` excessively.

[Source](https://developer.apple.com/videos/wwdc/2012/?id=235)

下面是通常会用到的宏定义，它会根据 debug/production 来选择执行 `NSLog()`：

```objective-c
#ifdef DEBUG
// Only log when attached to the debugger
#    define DLog(...) NSLog(__VA_ARGS__)
#else
#    define DLog(...) /* */
#endif
// Always log, even in production
#define ALog(...) NSLog(__VA_ARGS__)
```

[Source](http://iphoneincubator.com/blog/debugging/the-evolution-of-a-replacement-for-nslog)

### NSString

#####+ (instancetype)stringWithFormat:(NSString *)format,, ...

创建 `NSString` 不是特别昂贵，但是当在紧凑循环（比如作为字典的键值）中使用时， `+[NSString stringWithFormat:]` 的性能可以通过使用类似 `asprintf` 的 C 函数显著提高。

```objective-c
NSString *firstName = @"Daniel";
NSString *lastName = @"Amitay";
char *buffer;
asprintf(&buffer, "Full name: %s %s", [firstName UTF8String], [lastName UTF8String]);
NSString *fullName = [NSString stringWithCString:buffer encoding:NSUTF8StringEncoding];
free(buffer);
```

#####- (instancetype)initWithFormat:(NSString *)format, ...

参考 [`+[NSString stringWithFormat:]`](#-instancetypestringwithformatnsstring-format-)

# UIKit

- [UIImage](#uiimage)
- [UIView](#uiview)

---

### UIImage

#####+ (UIImage *)imageNamed:(NSString *)fileName

如果 boundle 中得某个图片只显示一次，推荐使用 `+ (UIImage *)imageWithContentsOfFile:(NSString *)path`，系统就不会缓存该图片。参考 Apple 的文档：

> If you have an image file that will only be displayed once and wish to ensure that it does not get added to the system’s cache, you should instead create your image using imageWithContentsOfFile:. This will keep your single-use image out of the system image cache, potentially improving the memory use characteristics of your app.

[Source](https://developer.apple.com/library/ios/documentation/uikit/reference/UIImage_Class/Reference/Reference.html#//apple_ref/occ/clm/UIImage/imageNamed:)

### UIView

#####@property(nonatomic) BOOL clearsContextBeforeDrawing

将 `UIView` 的属性 `clearsContextBeforeDrawing` 设置为 `NO` 在多数情况下可以提高绘制性能，尤其是在你自己用绘制代码实现了一个定制 view 的时候。

> If you set the value of this property to NO, you are responsible for ensuring the contents of the view are drawn properly in your drawRect: method. If your drawing code is already heavily optimized, setting this property is NO can improve performance, especially during scrolling when only a portion of the view might need to be redrawn.

[Source](https://developer.apple.com/library/ios/documentation/uikit/reference/uiview_class/uiview/uiview.html#//apple_ref/occ/instp/UIView/clearsContextBeforeDrawing)

> By default, UIKit clears a view’s current context buffer prior to calling its drawRect: method to update that same area. If you are responding to scrolling events in your view, clearing this region repeatedly during scrolling updates can be expensive. To disable the behavior, you can change the value in the clearsContextBeforeDrawing property to NO.

[Source](https://developer.apple.com/library/ios/documentation/2ddrawing/conceptual/drawingprintingios/DrawingTips/DrawingTips.html)

#####@property(nonatomic) CGRect frame

当设置一个 `UIView` 的 frame 属性时，应该保证坐标值和像素位置对齐，否则将会触发反锯齿降低性能，也有可能引起图形界面的边界模糊（译者注：尤其是涉及到绘制文字时将会引起文字模糊不清，非 retina 设备特别明显）。一种简单直接的办法就是使用 `CGRectIntegral()` 自动将 `CGRect` 的值四舍五入到整数。对于像素密度大于1的设备，可以将坐标值近似为 `1.0f / screen.scale` 整数倍。

# QuartzCore

- [CALayer](#calayer)

---

### CALayer

#####@property BOOL allowsGroupOpacity

在 iOS7 中，这个属性表示 layer 的 sublayer 是否继承父 layer 的透明度，主要用途是当在动画中改变一个 layer 的透明度时（会引起子 view 的透明度显示出来）。但是如果你不需要这种绘制类型，可以关闭这个属性来提高性能。

> When true, and the layer's opacity property is less than one, the layer is allowed to composite itself as a group separate from its parent. This gives the correct results when the layer contains multiple opaque components, but may reduce performance. 
> 
> The default value of the property is read from the boolean UIViewGroupOpacity property in the main bundle's Info.plist. If no value is found in the Info.plist the default value is YES for applications linked against the iOS 7 SDK or later and NO for applications linked against an earlier SDK.

上述引用来源已不存在，可以参考 `CALayer.h`。

> (Default on iOS 7 and later) Inherit the opacity of the superlayer. This option allows for more sophisticated rendering in the simulator but can have a noticeable impact on performance.

[Source](https://developer.apple.com/library/ios/documentation/general/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/uid/TP40009252-SW6)

#####@property BOOL drawsAsynchronously

`drawsAsynchronously` 属性会导致 layer 的 `CGContext` 延迟到后台线程绘制。这个属性对于频繁绘制的 leyer 有很大的好处。

> When this property is set to YES, the graphics context used to draw the layer’s contents queues drawing commands and executes them on a background thread rather than executing them synchronously. Performing these commands asynchronously can improve performance in some apps. However, you should always measure the actual performance benefits before enabling this capability.

[Source](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Reference/CALayer_class/Introduction/Introduction.html#//apple_ref/occ/instp/CALayer/drawsAsynchronously)

> Any drawing that you do in your delegate’s drawLayer:inContext: method or your view’s drawRect: method normally occurs synchronously on your app’s main thread. In some situations, though, drawing your content synchronously might not offer the best performance. If you notice that your animations are not performing well, you might try enabling the drawsAsynchronously property on your layer to move those operations to a background thread. If you do so, make sure your drawing code is thread safe.

[Source](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreAnimation_guide/ImprovingAnimationPerformance/ImprovingAnimationPerformance.html)

#####@property CGPathRef shadowPath

如果要操作 `CALayer` 的阴影属性，推荐设置 layer 的 `shadowPath` 属性，系统将会缓存阴影减少不必要的重绘。但当改变 layer 的 bounds 时，一定要重设 `shadowPath`。

```objective-c
CALayer *layer = view.layer;
layer.shadowOpacity = 0.5f;
layer.shadowRadius = 10.0f;
layer.shadowOffset = CGSizeMake(0.0f, 10.0f);
UIBezierPath *bezierPath = [UIBezierPath bezierPathWithRect:layer.bounds];
layer.shadowPath = bezierPath.CGPath;
```

> Letting Core Animation determine the shape of a shadow can be expensive and impact your app’s performance. Rather than letting Core Animation determine the shape of the shadow, specify the shadow shape explicitly using the shadowPath property of CALayer. When you specify a path object for this property, Core Animation uses that shape to draw and cache the shadow effect. For layers whose shape never changes or rarely changes, this greatly improves performance by reducing the amount of rendering done by Core Animation.

[Source](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreAnimation_guide/ImprovingAnimationPerformance/ImprovingAnimationPerformance.html)

#####@property BOOL shouldRasterize

如果 layer 只需要绘制依此，那么可以设置 `CALayer` 的属性 `shouldRasterize` 为 `YES`。但是如果该 layer 让然会被移动、缩放或者变形，那么将 `shouldRasterize` 设置为 `YES` 会损伤绘制性能，因为系统每次绘制完后会尝试再次重绘。

> When the value of this property is YES, the layer is rendered as a bitmap in its local coordinate space and then composited to the destination with any other content. Shadow effects and any filters in the filters property are rasterized and included in the bitmap.

[Source](https://developer.apple.com/library/mac/documentation/GraphicsImaging/Reference/CALayer_class/Introduction/Introduction.html#//apple_ref/occ/instp/CALayer/shouldRasterize)

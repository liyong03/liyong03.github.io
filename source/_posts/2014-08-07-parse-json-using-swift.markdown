---
layout: post
title: "使用 Swift 解析 JSON"
date: 2014-08-07 22:59:53 +0800
comments: true
categories: swift ios programming
---

本文翻译自[这篇文章](http://chris.eidhof.nl/posts/json-parsing-in-swift.html)，本文中所有的代码都放在[Github](https://gist.github.com/chriseidhof/4c071de50461a802874e)。

我将在本文中概述一个使用 Swift 完成的处理 JSON 的解析库。一个 JSON 的例子如下：

``` json
    var json : [String: AnyObject] = [
      "stat": "ok",
      "blogs": [
        "blog": [
          [
            "id" : 73,
            "name" : "Bloxus test",
            "needspassword" : true,
            "url" : "http://remote.bloxus.com/"
          ],
          [
            "id" : 74,
            "name" : "Manila Test",
            "needspassword" : false,
            "url" : "http://flickrtest1.userland.com/"
          ]
        ]
      ]
    ]
```

最具挑战的部分就是如何将该数据转换成如下 Swift 结构体的数组：

``` objc
    struct Blog {
        let id: Int
        let name: String
        let needsPassword : Bool
        let url: NSURL
    }
```

我们首先来看最终的解析函数，它包含两个运算法：`>>=` 和 `<*>` 。这两个运算符或许看起来很陌生，但是解析整个 JSON 结构就是这么简单。本文其他部分将会解释这些库代码。下面的解析代码是这样工作的：如果 JSON 是不合法的（比如 name 不存在或者 id 不是整型）最终结果将是 `nil` 。我们不需要反射（reflection）和 KVO ，仅仅需要几个函数和一些聪明的组合方式：

``` objc
    func parseBlog(blog: AnyObject) -> Blog? {
        return asDict(blog) >>= {
            mkBlog <*> int($0,"id")
                   <*> string($0,"name")
                   <*> bool($0,"needspassword")
                   <*> (string($0, "url") >>= toURL)
        }
    }
    let parsed : [Blog]? = dictionary(json, "blogs") >>= {
        array($0, "blog") >>= {
            join($0.map(parseBlog))
        }
    }
```

上面的代码到底做了什么呢？我们来仔细看看这些最重要的函数。首先来看看 dictionary 函数，它接受一个 `String` 到 `AnyObject` 的字典，返回另一个具有指定 key 的字典：

``` objc
    func dictionary(input: [String: AnyObject], key: String) ->  [String: AnyObject]? {
        return input[key] >>= { $0 as? [String:AnyObject] }
    }
```

例如在前面的 JSON 例子中，我们期望 key = "blogs" 包含一个字典。如果字典存在，上述函数返回该字典，否则返回 nil 。我们可以对 `Array`、`String`、`Integer` 写出同样的方法（下面只是生命，完整代码见 Github）：

``` objc
    func array(input: [String:AnyObject], key: String) ->  [AnyObject]?
    func string(input: [String:AnyObject], key: String) -> String?
    func int(input: [NSObject:AnyObject], key: String) -> Int?
```

现在，我们来看一下 JSON 例子的完整结构。它本身就是一个字典，包含一个 key 为 "blogs" 的另一个字典。该字典包含一个 key 为 "blog" 的 `Array` 。我们可以用下面的代码表达上述结构：

``` objc
    if let blogsDict = dictionary(parsedJSON, "blogs") {
        if let blogsArray = array(blogsDict, "blog") {
             // Do something with the blogs array
        }
    }
```

我么可以实现一个 >>= 操作来代替，接受一个 `optional` 参数，当该参数不为 `nil` 的时候，对其使用一个函数。该操作符使用 `flatten` 函数，`flatten` 函数将嵌套的 `optional` 展开：

``` objc
    operator infix >>= {}
    @infix func >>= <U,T>(optional : T?, f : T -> U?) -> U? {
        return flatten(optional.map(f))
    }
    func flatten<A>(x: A??) -> A? {
        if let y = x { return y }
        return nil
    }
```

另一个被频繁使用的是 `<*>` 操作符。例如下面的代码是用来解析单个 blog 的：

``` objc
    mkBlog <*> int(dict,"id")
           <*> string(dict,"name")
           <*> bool(dict,"needspassword")
           <*> (string(dict, "url") >>= toURL)
```

当所有的 `optional` 参数都是 `non-nil` 的时候该函数才能正常运行，上面的代码转化成：

``` objc
    mkBlog(int(dict,"id"), string(dict,"name"), bool(dict,"needspassword"), (string(dict, "url") >>= toURL))
```

所以，我们来看看操作符 `<*>` 的定义。它接受两个 `optional` 的参数，左边的参数是一个函数。如果两个参数都不是 `nil` ，将会对右边的参数使用左边的函数参数：

``` objc
    operator infix <*> { associativity left precedence 150 }
    func <*><A, B>(f: (A -> B)?, x: A?) -> B? {
        if let f1 = f {
            if let x1 = x {
                return f1(x1)
            }
        }
        return nil
    }
```

现在你有可能想知道 `mkBlog` 是做什么的吧。它是一个 [curried](http://en.wikipedia.org/wiki/Currying) 函数用来包装我们的初始化函数。首先，我们有一个 (Int,String,Bool,NSURL) -> Blog 类型的函数。然后 `curry` 函数将其类型转化为 `Int -> String -> Bool -> NSURL -> Blog` ：

``` objc
    let mkBlog = curry {id, name, needsPassword, url in 
       Blog(id: id, name: name, needsPassword: needsPassword, url: url) 
    }
```

我们将 `mkBlog` 和 `<*>` 一起使用，我们来看第一行：

``` objc
    // mkBlog : Int -> String -> Bool -> NSURL -> Blog
    // int(dict,"id") : Int?
    let step1 = mkBlog <*> int(dict,"id")
```

可以看到，用 `<*>` 将他们两个连起来，将会返回一个新的类型：`(String -> Bool -> NSURL -> Blog)?` ，然后和 `string` 函数结合：

``` objc
    let step2 = step1 <*> string(dict,"name")
```

我们得到：`(Bool -> NSURL -> Blog)?` ，一直这样结合，最后将会得到类型为 `Blog?` 的值。

希望你现在能明白整个代码是如何在一起工作的了。通过创建一些辅助函数和运算符，我们可以让解析强类型的 JSON 数据变得非常容易。如果不用 `optional` 类型，那么我们将会使用完全不同的类型，并且包含一些错误信息，但这将是另外的 blog 的话题了。
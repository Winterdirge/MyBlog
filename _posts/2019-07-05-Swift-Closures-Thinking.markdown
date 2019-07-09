---
layout:     post
title:      "Swift Closures随想"
subtitle:   "Swift闭包"
date:       2019-03-08
author:     "YangGuang"
header-style: text
tags:
    - Swift
    - iOS
    - 移动端
---

>本文不会对Closures的基本语法进行解读，如果对于Closures语法有所疑惑，请参考[Apple官方文档](https://docs.swift.org/swift-book/LanguageGuide/Closures.html)
本文主要针对Closures在项目中的使用情况进行分析，主要涉及**为何使用、何时使用、如何使用**等几个方面进行分析
如有不足之处欢迎批评指正，谢谢！

*In programming languages, closures (also called lexical closures or function closures) are techniques for implementing lexically scoped named binding in languages with first-class functions. Operationally, a closure is a record storing a function together with an environment: a mapping associating each free variable of the function (variables that are used locally, but defined in an enclosing scope) with the value or reference to which the name was bound when the closure was created. — [Wikipedia](https://en.wikipedia.org/wiki/Closure_%28computer_programming%29#Notes)*

# 是啥

官方文档给出这样一句话：*Closures are self-contained blocks of functionality that can be passed around and used in your code. Closures in Swift are similar to blocks in C and Objective-C and to lambdas in other programming languages.*
大意就是：*闭包是一个独立的功能块，可以在代码中传递和使用。 Swift中的闭包类似于C和Objective-C中的block以及其他编程语言中的lambdas。*
说白了和函数比较类似，对它们的调用本质上都是执行了一段代码，它们都可以有参数，都可以有返回值，只是表现的方式不同。Closures有它自己的特色。

# 为啥要用

说白了，用Closures就是为了方便、简洁、优雅。
其实Closures并没有赋予我们额外的能力，可以用Closures实现的功能，也可以用其他方法来实现。不过Closures确实可以使代码更加的清晰、紧凑以及可读，众所周知，清晰明了的代码会包含更少的bug，测试起来也更加方便。
Closures只是一种让函数访问本地状态、变量等更加方便的途径而已。
我们不必创建一个可以使用局部变量的类，而只需在现场定义函数，它就可以隐式访问当前可见的每个变量。使用传统OOP语言定义成员方法时，从某种意义上来说，这些成员方法就相当于闭包，因为它们可以访问“此类中可见的所有成员”。

# 啥时候用

在具体的项目中，Closures主要用于回调以及代理中。
例如，让你设计一个网络请求的类，你如何把请求的结果返回给调用者？因为网络请求是一个异步的过程，你不能直接通过函数的返回值直接传递结果，用户也不可能一直等着函数返回，那么该怎么办呢？
一个办法是使用通知。当请求结束时，被调用方会发送网络请求结束通知，带上网络请求的结果，这种方法需要你注册成为观察者，然后编写收到通知后要执行的代码。试想一下，我们如果有n多个网络请求，就需要在结果返回时去判断该结果属于哪一个网络请求，然后才能对结果进行相应处理。一种方法是使用代理，实现网络请求的代理方法，网络请求结束时会调用代理方法，传递网络请求结果，这种方法和上面相比少了一步注册成为观察者的过程，但是同样需要在代理方法中对结果进行判断。这两种方法都需要将请求和结果分开来写，阅读起来不是很直观。
另一种方法就是使用Closures。既然我们不知道网络请求何时返回结果，但是我们知道请求返回时我们要做什么，也就是说网络请求结束后我们一定会进行这些操作，那么我们有没有什么办法直接告诉被调方，在你请求网络结束后，进行我需要的操作。说白了就是你跑完你的代码以后，能不能接着跑我的代码，这个问题的关键在于怎么能让对方知道我要让它跑什么代码，怎么能让它知道呢？当然是直接告诉它啊，也就是直接把你想执行的代码告诉它，这种‘告诉’可以通过参数传递过去。什么？把要执行的方法传过去？没错，就是直接传过去，在C语言中我们可以直接传递一个函数指针，OC中可以使用selector或者block，Swift理所当然就是用Closures了。什么？你没听懂？没关系，时间久了，用的多了自然而然就差不多了。
Closures的使用时机需要具体问题具体分析，这就需要有比较丰富经验，不能单纯为了使用而使用，有时候过多的使用Closures会使代码更加难懂，比如说Closures嵌套Closures的情况。

# 怎么用

OC中block：

```objectivec
//As a local variable:
returnType (^blockName)(parameterTypes) = ^returnType(parameters) {...};

//As a property:
@property (nonatomic, copy, nullability) returnType (^blockName)(parameterTypes);

//As a method parameter:
- (void)someMethodThatTakesABlock:(returnType (^nullability)(parameterTypes))blockName;

//As an argument to a method call:
[someObject someMethodThatTakesABlock:^returnType (parameters) {...}];

//As a parameter to a C function:
void SomeFunctionThatTakesABlock(returnType (^blockName)(parameterTypes));

//As a typedef:
typedef returnType (^TypeName)(parameterTypes);
TypeName blockName = ^returnType(parameters) {...};
```

Swift Closures：

```swift
//As a variable:
var closureName: (ParameterTypes) -> ReturnType

//As an optional variable:
var closureName: ((ParameterTypes) -> ReturnType)?

//As a type alias:
typealias ClosureType = (ParameterTypes) -> ReturnType

//As a constant:
let closureName: ClosureType = { ... }

//As a parameter to another function:
funcName(parameter: (ParameterTypes) -> ReturnType)

//Note: if the passed-in closure is going to outlive the scope of the method, 
//e.g. if you are saving it to a property, it needs to be annotated with @escaping.
//As an argument to a function call:
funcName({ (ParameterTypes) -> ReturnType in statements })

//As a function parameter:
array.sorted(by: { (item1: Int, item2: Int) -> Bool in return item1 < item2 })

//As a function parameter with implied types:
array.sorted(by: { (item1, item2) -> Bool in return item1 < item2 })

//As a function parameter with implied return type:
array.sorted(by: { (item1, item2) in return item1 < item2 })

//As the last function parameter:
array.sorted { (item1, item2) in return item1 < item2 }

//As the last parameter, using shorthand argument names:
array.sorted { return $0 < $1 }

//As the last parameter, with an implied return value:
array.sorted { $0 < $1 }

//As the last parameter, as a reference to an existing function:
array.sorted(by: <)

//As a function parameter with explicit capture semantics:
array.sorted(by: { [unowned self] (item1: Int, item2: Int) -> Bool in return item1 < item2 })

//As a function parameter with explicit capture semantics and inferred parameters / return type:
array.sorted(by: { [unowned self] in return $0 < $1 })
```

OC Block和Swift Closures之间的语法差别从上面可以清晰的看出了，在这里就不总结了。

# 下面说一说我认为比较有意思的东西

对于调用方，Closures中的参数是被调方传递给你的，如果被调方传给你的参数还是Closures，那么这个Closures其实已经是被调方实现好了的，只需要你传递Closures所需的参数即可。下面通过例子简单看一下
```swift
typealias closure1 = (_ intValue: Int) -> Void
typealias closure2 = (_ closure: closure1) -> Void
typealias closure3 = (_ closure: closure2) -> Void
```

定义了三个Closures，closure1有一个Int类型参数，closure2有一个closure1类型参数，closure3有一个closure2类型参数。

```swift
func testClosure1(closure1: closure1) -> Void {
    //do something...
    if let closure1 = closure1 {
        closure1(1)
    }
}

testClosure1 { (i) in
    print(i)
}
```
上面这种使用方法在开发过程中应该会经常用到，通常是用作回调，需要注意的是这里closure1中的参数是被调用方传入的，调用方只需要传入具体实现就行，也就是说，被调方用自己传入自己的参数调用你的实现，就是通常意义上的回调，双方约定好参数类型，被调方提供参数，调用方提供实现。

```swift
func testClosure2(closure2: closure2) -> Void {
    var closure1: closure1 = { i in
        print(i)
    }
    closure2(closure1)

    //或者使用以下写法
    closure2 { i in //这其实是block1的具体实现，参数类型是int
        print(i)
    }
}

testClosure2 { (closure1) in
    closure1(5)
}
```

上面这种方式开发中也会用到，closure2中的参数是closure1，此时被调用方传入的参数其实是类型为closure1的参数，这是调用方拿到的就是closure1这个closure，这个closure已经由被调用方定义好了，传入参数即可使用。（可能有点绕，内部提供实现，外部提供参数）
下面这个可能更绕

```swift
func testClosure3(closure3: closure3) -> Void {
    var closure22: closure2 = { closure in
        closure(11)
    }
    closure3(closure22)
}
testClosure3 { closure in
    closure({ i in
        print(i)
    })
}
```

3重closure，这是就需要调用方传入closure1的具体实现，被调方在此基础上传入参数实现closure2，最后再将这个closure2当作参数传递出去，日常开发中反正我是没用过，如果大家发现有这么用的那么请告诉我，让我也一起学习下。至于更多重的closures，请大家自行脑补吧。

# To Be Continued

说白了不论是Closure还是其他的语言特性，都是为了更好的与机器交流而已，编程语言使我们可以与机器交流，像中文英文可以让我们相互交流而已。而在编程语言的特性，就好比某些网络通用语或者谚语，往往可以使用很简短的几句代码或几句话，实现很复杂的功能。这也是编程语言发展的一个趋势。
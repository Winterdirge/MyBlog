---
layout:     post
title:      "Swift 5.0 新特性"
subtitle:   "Swift"
date:       2019-03-28
author:     "YangGuang"
header-img: "assets/images/base/post-bg-js-version.jpg"
tags:
    - 移动端开发
    - iOS
    - 翻译
---

# Raw strings
相关资料 [SE-0200](https://github.com/apple/swift-evolution/blob/master/proposals/0200-raw-string-escaping.md)
原始字符串，说白了就是所见即所得，输入的字符串长什么样子，输出的就长什么样子(某些字符串插值的情况先不讨论)
市面上大部分的编程语言，声明使用字符串的时候用的都是引号`""`，但是当字符串中出现`""`的时候，字符串中的`""`可能被当作该字符串的结束标志，从而导致奇奇怪怪的错误。
因此，先驱者们就引入了转义字符`\`(escape character)，当字符串中需要某些特殊字符的时候，就需要在这些特殊字符前面加上转义字符，方便编译器对字符串进行处理。但是如果需要输入转义字符呢？那就需要在`\`前面再加一个`\`。这样的字符串虽然可以让编译器识别，但是对于代码的编写者和阅读者来说，过程还是有些痛苦的。
为了解决这个问题，Swift就引入了Raw string，使用方法是在你的字符串前后加上`一个或多个#`号
```
let rain = #"The "rain" in "Spain" falls mainly on the Spaniards."#
```
可以看到，无论字符串中有多少`""`都可以很好识别
有的人可能要问了，要是字符串中有`"#`符号呢？注意了，我们前面说的是`一个或多个`，没错，这时候我们需要做的只需要增加`#`的数量的就行了，不过开头和结尾的`#`数量要保持一致
```
let str = ##"My dog said "woof"#gooddog"##
```
那我们的字符串插值怎么办？加上一个`#`即可
```
let answer = 42
//before
let dontpanic = "The answer is \(answer)."
//now
let dontpanic = #"The answer is \#(answer)."#
```
同样的Raw strings也支持Swift的多行字符串
```
let multiline = #"""
The answer to life,
the universe,
and everything is \#(answer).
"""#
```
有了Raw string我们就可以写一些比较复杂的字符串了，特别是某些正则表达式，因为正则表达式中总是包含很多`\`
```
//before
let regex1 = "\\\\[A-Z]+[A-Za-z]+\\.[a-z]+"
//now
let regex2 = #"\\[A-Z]+[A-Za-z]+\.[a-z]+"#
```
# A standard Result type
相关资料 [SE-0235](https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md)
```
public enum Result<Success, Failure> where Failure : Error {
    case success(Success)
    case failure(Failure)
    public func map<NewSuccess>(_ transform: (Success) -> NewSuccess) -> Result<NewSuccess, Failure>
    public func mapError<NewFailure>(_ transform: (Failure) -> NewFailure) -> Result<Success, NewFailure> where NewFailure : Error
    public func flatMap<NewSuccess>(_ transform: (Success) -> Result<NewSuccess, Failure>) -> Result<NewSuccess, Failure>
    public func flatMapError<NewFailure>(_ transform: (Failure) -> Result<Success, NewFailure>) -> Result<Success, NewFailure> where NewFailure : Error
    public func get() throws -> Success
    public init(catching body: () throws -> Success)
}
```
最初见到这个类型是在Kingfisher开源库中，第一次看到时感觉也没什么，不就是新定义了一个Result枚举，然后把success和failure包装了起来，后来细细研究了一下，发现事情并没有那么简单。
Result提供了一个通用的范型，基本可以囊括所有的成功和失败类型。也就是说，它对外提供了一个统一的结果类型，用来处理返回结果。
```
//eg
enum NetworkError: Error {
    case badURL
}
//成功返回Int，错误返回NetworkError类型，用Result类型包装起来
func fetchUnreadCount1(from urlString: String, completionHandler: @escaping (Result<Int, NetworkError>) -> Void)  {
    guard let url = URL(string: urlString) else {
        completionHandler(.failure(.badURL))
        return
    }

    // complicated networking code here
    print("Fetching \(url.absoluteString)...")
    completionHandler(.success(5))
}
//处理返回结果，标准的switch case来处理枚举
fetchUnreadCount1(from: "https://www.hackingwithswift.com") { result in
    switch result {
    case .success(let count):
        print("\(count) unread messages.")
    case .failure(let error):
        print(error.localizedDescription)
    }
}
```
Result提供了`get`方法，它会返回成功的结果，或者抛出错误结果。
```
//get方法
public func get() throws -> Success {
    switch self {
    case let .success(success):
        return success
    case let .failure(failure):
        throw failure
    }
}
fetchUnreadCount1(from: "https://www.hackingwithswift.com") { result in
    if let count = try? result.get() {
        print("\(count) unread messages.")
    }
}
```
另外，还提供了一种使用会抛出错误的closure来初始化Result类型的方法，该closure执行后，成功的结果会被存储在`.success`中，抛出的错误会被存储在`.failure`
```
public init(catching body: () throws -> Success) {
    do {
        self = .success(try body())
    } catch {
        self = .failure(error)
    }
}
//eg
let result = Result { try String(contentsOfFile: someFile) }
```
其次，如果定义的错误类型不满足需求的话，Result还提供了对结果的转换方法，`map(), flatMap(), mapError(), flatMapError()`
```
enum FactorError: Error {
    case belowMinimum
    case isPrime
}
func generateRandomNumber(maximum: Int) -> Result<Int, FactorError> {
    if maximum < 0 {
        // creating a range below 0 will crash, so refuse
        return .failure(.belowMinimum)
    } else {
        let number = Int.random(in: 0...maximum)
        return .success(number)
    }
}
let result1 = generateRandomNumber(maximum: 11)
//如果result1是.success，下面的map方法， 将
//Result<Int, FactorError>转换为Result<String, FatorError>
//如果是.failure，不做处理，仅仅将failure返回
let stringNumber = result1.map { "The random number is: \($0)." }
```
另一种情况是将上个结果的返回值转换为另一个结果
```
func calculateFactors(for number: Int) -> Result<Int, FactorError> {
    let factors = (1...number).filter { number % $0 == 0 }

    if factors.count == 2 {
        return .failure(.isPrime)
    } else {
        return .success(factors.count)
    }
}
let result2 = generateRandomNumber(maximum: 10)
//计算result2 .success结果的因子，然后返回另一个Result
let mapResult = result2.map { calculateFactors(for: $0) }
```
细心的朋友们可能会发现，我们将.success转换为另一个Result，也就是说mapResult的类型是`Result<Result<Int, Factor>, Factor>`，而不是`Result<Int, Factor>`，那如果结果需要转换很多次呢？那岂不是要嵌套很多层？没错，很多朋友可能已经想到了，那就是Swift中的`flatMap`方法，Result也对其进行了实现
```
public func flatMap<NewSuccess>(_ transform: (Success) -> Result<NewSuccess, Failure>) -> Result<NewSuccess, Failure> {
    switch self {
    case let .success(success):
        return transform(success)
    case let .failure(failure):
        return .failure(failure)
    }
}
public func flatMapError<NewFailure>(_ transform: (Failure) -> Result<Success, NewFailure>) -> Result<Success, NewFailure> {
    switch self {
    case let .success(success):
        return .success(success)
    case let .failure(failure):
        return transform(failure)
    }
}
//此时flatMapResult的类型为Result<Int, FactorError>
let flatMapResult = result2.flatMap { calculateFactors(for: $0) }
```
# Customizing string interpolation

相关资料[SE-0228](https://github.com/apple/swift-evolution/blob/master/proposals/0228-fix-expressiblebystringinterpolation.md)
新的可自定义的字符串插值系统，更加的高效灵活，增加了以前版本不可能实现的全新功能，它可以让我们控制对象在字符串中的显示方式。
```
//eg
struct User {
    var name: String
    var age: Int
}
extension String.StringInterpolation {
    mutating func appendInterpolation(_ value: User) {
        appendInterpolation("My name is \(value.name) and I'm \(value.age)")
    }
}
let user = User(name: "Sunshine", age: 18)
print("User details: \(user)")
//before: User(name: "Sunshine", age: 18)
//after: My name is Sunshine and I'm 18
```
有人可能会说，直接实现`CustomStringConvertible`不就行了，非得这么复杂，不仅需要扩展`String.StringInterpolation`，还需要实现`appendInterpolation`方法。没错，对于简单的插值来说，重写一下`description`就行。
下面就来看一下，新的插值的全新特性。只要你开心，你可以自定义有很多个参数的插值方法。
```
extensionn String.StringInterpolation {
    mutating func appendInterpolation(_ number: Int, style: NumberFormatter.Style) {
        let formatter = NumberFormatter()
        formatter.numberStyle = style
        if let result = formatter.string(from: number as NSNumber) {
            appendLiteral(result)
        }
    }

    mutating func appendInterpolation(repeat str: String, _ count: Int) {
        for _ in 0 ..< count {
            appendLiteral(str)
        }
    }

    mutating func appendInterpolation(_ values: [String], empty defaultValue: @autoclosure () -> String) {
        if values.count == 0 {
            appendLiteral(defaultValue())
        } else {
            appendLiteral(values.joined(separator: ", "))
        }
    }
}
//we can use like this
print("The lucky number is \(8, style: .spellOut)")
//res: The lucky number is eight.

print("This T-shirt is \(repeat: "buling", 2)")
//res: This T-shirt is buling buling

let nums = ["1", "2", "3"]
print("List of nums: \(nums, empty: "No one").")
//res: List of nums: 1, 2, 3.
```
我们也可以定义自己的interpolation类型，需要注意一下几点
* 需要遵循`ExpressibleByStringLiteral`, `ExpressibleByStringInterpolation`, `CustomStringConvertible`协议
* 在自定义类型中，需要创建一个`StringInterpolation`的结构体遵循`StringInterpolationProtocol`
* 实现`appendLiteral`方法，以及实现一个或多个`appendInterpolation`方法
* 自定义类型需要有两个初始化方法，允许直接从字符串，或字符串插值创建对象

```
//copy from Whats-New-In-Swift-5-0
struct HTMLComponent: ExpressibleByStringLiteral, ExpressibleByStringInterpolation, CustomStringConvertible {
    struct StringInterpolation: StringInterpolationProtocol {
        // start with an empty string
        var output = ""

        // allocate enough space to hold twice the amount of literal text
        init(literalCapacity: Int, interpolationCount: Int) {
            output.reserveCapacity(literalCapacity * 2)
        }

        // a hard-coded piece of text – just add it
        mutating func appendLiteral(_ literal: String) {
            print("Appending \(literal)")
            output.append(literal)
        }

        // a Twitter username – add it as a link
        mutating func appendInterpolation(twitter: String) {
            print("Appending \(twitter)")
            output.append("<a href=\"https://twitter/\(twitter)\">@\(twitter)</a>")
        }

        // an email address – add it using mailto
        mutating func appendInterpolation(email: String) {
            print("Appending \(email)")
            output.append("<a href=\"mailto:\(email)\">\(email)</a>")
        }
    }

    // the finished text for this whole component
    let description: String

    // create an instance from a literal string
    init(stringLiteral value: String) {
        description = value
    }

    // create an instance from an interpolated string
    init(stringInterpolation: StringInterpolation) {
        description = stringInterpolation.output
    }
}
//usage
let text: HTMLComponent = "You should follow me on Twitter \(twitter: "twostraws"), or you can email me at \(email: "paul@hackingwithswift.com")."
//You should follow me on Twitter <a href="https://twitter/twostraws">@twostraws</a>, or you can email me at <a href="mailto:paul@hackingwithswift.com">paul@hackingwithswift.com</a>.
```

当然，它的玩法还有很多，需要我们慢慢探索。

# Handling future enum cases

相关资料 [SE-0192](https://github.com/apple/swift-evolution/blob/master/proposals/0192-non-exhaustive-enums.md)
处理未来可能改变的枚举类型
Swift的安全特性需要`switch`语句必须是详尽的完全的，必须涵盖所有的情况，但是将来添加新的case时，例如系统框架改变等，就会导致问题。
现在我们可以通过新增的`@unknown`属性来处理
* 对于所有其他case，应该运行此默认case，因为我不想单独处理它们
* 我想单独处理所有case，但如果将来出现任何case，请使用此选项，而不是导致错误

```
enum PasswordError: Error {
    case short
    case obvious
    case simple
}

func showNew(error: PasswordError) {
    switch error {
    case .short:
        print("Your password was too short.")
    case .obvious:
        print("Your password was too obvious.")
    @unknown default:
        print("Your password wasn't suitable."）
    }
}
//cause warning: Switch must be exhaustive.
```
# Transforming and unwrapping dictionary values

相关资料[SE-0218](https://github.com/apple/swift-evolution/blob/master/proposals/0218-introduce-compact-map-values.md)
`Dictionary`增加了`compactMapValues()`方法，该方法将数组中的`compactMap()`（转换我的值，展开结果，放弃空值）与字典的`mapValues()`（保持键不变，转换我的值）结合起来。
```
//eg
//返回value是Int的新字典
let times = [
    "Hudson": "38",
    "Clarke": "42",
    "Robinson": "35",
    "Hartis": "DNF"
]
//two ways
let finishers1 = times.compactMapValues { Int($0) }
let finishers2 = times.compactMapValues { Init.init }
```
或者可以使用该方法拆包可选值，过滤`nil`值，而不用做任何的类型转换。
```
let people = [
    "Paul": 38,
    "Sophie": 8,
    "Charlotte": 5,
    "William": nil
]
let knownAges = people.compactMapValues { $0 }
```
# Checking for integer multiples

相关资料[SE-0225](https://github.com/apple/swift-evolution/blob/master/proposals/0225-binaryinteger-iseven-isodd-ismultiple.md)
新增`isMultiple(of:)`方法，可是是我们更加清晰方便的判断一个数是不是另一个数的倍数，而不是每次都使用`%`
```
//判断偶数
let rowNumber = 4
if rowNumber.isMultiple(of: 2) {
    print("Even")
} else {
    print("Odd")
}
```
# Flattening nested optionals resulting from try?

相关资料[SE-0230](https://github.com/apple/swift-evolution/blob/master/proposals/0230-flatten-optional-try.md)
修改了`try?`的工作方式，让嵌套的可选类型变成正常的可选类型，即让多重可选值变成一重，这使得它的工作方式与可选链和条件类型转换相同。
```
struct User {
    var id: Int

    init?(id: Int) {
        if id < 1 {
            return nil
        }

        self.id = id
    }

    func getMessages() throws -> String {
        // complicated code here
        return "No messages"
    }
}

let user = User(id: 1)
let messages = try? user?.getMessages()
//before swift 5.0: String?? 即Optional(Optional(String))
//now: String? 即Optional(String)
```
# Dynamically callable types

[SE-0216](https://github.com/apple/swift-evolution/blob/master/proposals/0216-dynamic-callable.md)
增加了`@dynamicCallable`属性，它将类型标记为可直接调用。
它是语法糖，而不是任何类型的编译器魔法，将方法`random(numberOfZeroes：3)`标记为`@dynamicCallable`之后，可通过下面方式调用
`random.dynamicallyCall(withKeywordArguments：["numberOfZeroes"：3])`。
`@dynamicCallable`是Swift 4.2的`@dynamicMemberLookup`的扩展，为了使Swift代码更容易与Python和JavaScript等动态语言一起工作。
要将此功能添加到您自己的类型，需要添加`@dynamicCallable`属性并实现下面方法
```
func dynamicCall(withArguments args: [Int]) -> Double
func dynamicCall(withKeywordArguments args: KeyValuePairs <String，Int>) -> Double 
```
当你调用没有参数标签的类型（例如a（b，c））时会使用第一个，而当你提供标签时使用第二个（例如a（b：cat，c：dog））。

`@dynamicCallable`对于接受和返回的数据类型非常灵活，使您可以从Swift的所有类型安全性中受益，同时还有一些高级用途。因此，对于第一种方法（无参数标签），您可以使用符合`ExpressibleByArrayLiteral`的任何内容，例如数组，数组切片和集合，对于第二种方法（使用参数标签），您可以使用符合`ExpressibleByDictionaryLiteral`的任何内容，例如字典和键值对。

除了接受各种输入外，您还可以为输出提供多个重载，一个可能返回字符串，一个是整数，依此类推。只要Swift能够解决使用哪一个，你就可以混合搭配你想要的一切。

我们来看一个例子。这是一个生成介于0和某个最大值之间数字的结构，具体取决于传入的输入：

```
import Foundation

@dynamicCallable
struct RandomNumberGenerator1 {
    func dynamicallyCall(withKeywordArguments args: KeyValuePairs<String, Int>) -> Double {
        let numberOfZeroes = Double(args.first?.value ?? 0)
        let maximum = pow(10, numberOfZeroes)
        return Double.random(in: 0...maximum)
    }
}
```
可以使用任意数量的参数调用该方法，或者为零，因此读取第一个值并为其设置默认值。

我们现在可以创建一个RandomNumberGenerator1实例并像函数一样调用：
```
let random1 = RandomNumberGenerator1()
let result1 = random1(numberOfZeroes: 0)
```
或者
```
@dynamicCallable
struct RandomNumberGenerator2 {
    func dynamicallyCall(withArguments args: [Int]) -> Double {
        let numberOfZeroes = Double(args[0])
        let maximum = pow(10, numberOfZeroes)
        return Double.random(in: 0...maximum)
    }
}
let random2 = RandomNumberGenerator2()
let result2 = random2(0)
```
使用@dynamicCallable时需要注意：

* 可以将它应用于结构，枚举，类和协议
* 如果你实现`withKeywordArguments:`并且没有实现`withArguments:`，你的类型仍然可以在没有参数标签的情况下调用，你只需要将键设置为空字符串就行
* 如果`withKeywordArguments:`或`withArguments:`被标记为throw，则调用该类型也会throw
* 您不能将`@dynamicCallable`添加到扩展，只能添加在类型的主要定义中
* 可以为类型添加其他方法和属性，并正常使用它们。

另外，它不支持方法解析，这意味着我们必须直接调用类型（例如random(numberOfZeroes: 5)，而不是调用类型上的特定方法（例如random.generate(numberOfZeroes: 5)

# THE END
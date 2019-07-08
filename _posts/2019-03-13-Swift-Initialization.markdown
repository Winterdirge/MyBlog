---
layout:     post
title:      "Swift Initialization"
subtitle:   "Swift初始化方法"
date:       2019-03-13
author:     "YangGuang"
header-style: text
tags:
    - Swift
---
 
 
 >本文基本上是官方文档的简单翻译，有限添加了个人理解，如有错漏之处，请不吝指出，谢谢。[原文链接](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html#ID219)

初始化方法为class struct enum中的`stored property`赋值
初始化操作需要定义初始化方法，初始化方法是可以创建特定类型实例的特殊方法，和OC不同的是Swift初始化方法并没有返回值，
它们的职责都是确保新的实例在第一次使用时已经被正确的初始化

# Setting Inital Values for Stored Properties

Struct，Class的实例创建时必须为所有的存储属性赋值（Stored Property）
#### Initilizers

初始化方法使用关键字`init`
最简单的初始化方法没有参数

```swift
init() {}
```

#### Default Property Values默认属性值

可以在初始化方法中为属性设置默认值，也可以在属性定义时指定默认值

```swift
struct A {
	var a = 1
}
//这种情况下会生成两个默认初始化方法，分别是init()和init(a: Int)
//如果是类的话只会生成一个默认的初始化方法init()
```

# Customizing Initialization自定义初始化

自定义初始化方法：
可以通过输入参数、可选属性、指定常量值来进行自定义的初始化操作

#### Initialztion Parameters初始化参数

可以提供初始化参数作为初始化方法的一部分，用来定义初始化过程中所用到的value的类型和名称，初始化参数的语法与函数相同（`初始化方法没有返回值，但是函数可能会有`）

```swift
struct A {
    var a: Double
    init(a: Double) {
        self.a = a //如果外部参数和属性名称相同的话，需要使用self.来表明是为实例的同名属性赋值，否则会引发编译错误
    }
}
```
#### Parameter Names and Argument Labels

参数名和参数标签
和函数和方法参数一样，初始化方法的参数可以有用于initalizer's body的参数名，以及被调用时的参数标签。
然而，初始化方法没有一个可辨别的方法名（`函数可以通过不同的函数名来判断，初始化方法都是init`），所以初始化方法中参数的名字和类型对判断需要调用哪个方法起着至关重要的作用，因此如果我们不提供参数标签，Swift会为初始化程序中的每个参数提供自动参数标签（`此处标签名与参数名相同`）。

```Swift
struct Color {
    let red, green, blue: Double
    init(red: Double, green: Double, blue: Double) {
        //因为参数名和属性名称一样，初始化时需要使用self
        self.red = red
        self.green = green
        self.blue = blue
    }
    init(white: Double) {
        //参数名与属性名不同，可以不使用self
        red = white
        green = white
        blue = white
    } 
}
//此处的red，green，blue即为自动生成的与init方法参数名称相同的参数标签
let magenta = Color(red: 1.0, green: 0.0, blue: 1.0)
```

#### Initializer Parameters Without Argument Labels

不带参数标签的初始化参数
如果不想使用参数标签，需要使用下划线（`_`）代替具体的参数标签。

```swift
struct Celsius {
    var temperatureInCelsius: Double
    init(fromFahrenheit fahrenheit: Double) {
        temperatureInCelsius = (fahrenheit - 32.0) / 1.8
    }
    init(fromKelvin kelvin: Double) {
        temperatureInCelsius = kelvin - 273.15
    }
    init(_ celsius: Double) {
        temperatureInCelsius = celsius
    }
}
//与上个Color例子的调用方式相比较，缺少了参数标签celsius，这就是下划线具体作用
let bodyTemperature = Celsius(37.0)
// bodyTemperature.temperatureInCelsius is 37.0
```

#### Optional Property Types

可选参数类型
如果你的自定义类型有一个在逻辑上允许为空（no value）的property，无论是因为它无法在初始化时被设置，还是因为它在之后允许被设置为空，这时你可以将它设置为`Optional`（可选类型，有时候又叫可空类型），这种类型会被自动初始化为`nil`，表示这个属性在初始化时故意没有值（`也就是说这些属性在初始化方法中不初始化也不会报错，其他属性不初始化会报编译错误`）。

```swift
class SurveyQuestion {
    var text: String
    var response: String?
    //如果response不是可选类型，init方法不初始化response就会报错
    init(text: String) {
        self.text = text
    }
    func ask() {
        print(text)
    }
}
let cheeseQuestion = SurveyQuestion(text: "Do you like cheese?")
cheeseQuestion.ask()
// Prints "Do you like cheese?"
cheeseQuestion.response = "Yes, I do like cheese."
```

#### Assigning Constant Properties During Initialization

初始化是指定常量值
你可以在初始化的任何时候为常量设定值，只要在初始化结束后它有一个确定的值即可。一旦常量被初始化，它将无法再次更改(`也就是说，常量只能在初始化方法中设置一次，以后无法再次修改`)。

```swift
class SurveyQuestion {
    let text: String
    var response: String?
    init(text: String) {
        self.text = text
    }
    func ask() {
        print(text)
    }
}
let beetsQuestion = SurveyQuestion(text: "How about beets?")
beetsQuestion.ask()
// Prints "How about beets?"
beetsQuestion.response = "I also like beets. (But not with cheese.)"
```

# Default Initializers

默认初始化方法
如果Struct或者Class为所有property提供了默认值，并且没有提供任何初始化方法，Swift将会为它们生成默认你的初始化方法，该方法会生成一个所有属性都是默认值的实例。

```swift
class ShoppingListItem {
    var name: String?
    var quantity = 1
    var purchased = false
}
var item = ShoppingListItem()
```

#### Memberwise Initializers for Structure Types

结构体类型如果没有定义初始化方法，将会生成一个参数标签为它的每个属性的初始化方法。（`和默认初始化方法不同，即使属性没有默认值，也会生成初始化方法`）

```swift
struct Size {
    var width = 0.0, height = 0.0
}
//自动生成的初始化方法，以每个property为标签
let twoByTwo = Size(width: 2.0, height: 2.0)
```

# Initializer Delegation for Value Types

值类型的初始化方法委派
初始化方法可以调用其他初始化方法来进行实例的部分初始化，这个过程叫做初始化委派，避免了不同初始化方法中的重复代码。
值类型和类类型的委派方式不同，因为值类型无法继承，所以它们的委派过程相对简单，它们只能委派到另一个它们自己已经实现的初始化方法。类类型由于继承的原因，它必须确保它继承而来的属性被合理的初始化。
对于值类型，你需要使用`self.init`来调用相同值类型的其他初始化方法，而且你只能在初始化方法中调用`self.init`。（`初始化实例的时候使用ClassName.init方法`）
如果为值类型实现了一个自定义的初始化方法，你将无法在调用默认的初始化方法（如果是结构体则无法访问memberwise initializer）。此约束可以防止有人使用自动的初始化方法绕过一个更复杂的初始化方法，进而导致某些属性没有被正确的初始化。

```swift
struct Size {
    var width = 0.0, height = 0.0
}
struct Point {
    var x = 0.0, y = 0.0
}
struct Rect {
    var origin = Point()
    var size = Size()
    //虽然origin和size都有默认值，但是不会生成默认的无参数的init方法，只能手动声明一个
    init() {}
    init(origin: Point, size: Size) {
        self.origin = origin
        self.size = size
    }
    init(center: Point, size: Size) {
        let originX = center.x - (size.width / 2)
        let originY = center.y - (size.height / 2)
        //self.init只能在初始化方法中调用，在其他方法中不能调用
        self.init(origin: Point(x: originX, y: originY), size: size)
    }
}
let basicRect = Rect()
// basicRect's origin is (0.0, 0.0) and its size is (0.0, 0.0)
let originRect = Rect(origin: Point(x: 2.0, y: 2.0), size: Size(width: 5.0, height: 5.0))
// originRect's origin is (2.0, 2.0) and its size is (5.0, 5.0)
let centerRect = Rect(center: Point(x: 4.0, y: 4.0), size: Size(width: 3.0, height: 3.0))
// centerRect's origin is (2.5, 2.5) and its size is (3.0, 3.0)
```

# Class Inheritance and Initialization

一个类的所有属性（包括继承而来的属性）都需要被初始化。
Swift为类定义了两种初始化方法，指定初始化方法（designated initializers）和便捷初始化方法（convenience initializers）

#### Designated Initializers and Convenience Initializers

指定初始化方法是类的首要初始化方法，它会初始化类的所有属性，然后调用父类的初始化方法以继续父类链的初始化过程。
类只有少量的指定初始化方法，一般来说一个类通常只有一个。指定初始化方法是初始化发生的起始点，初始化方法通过该点继续父类的初始化进程。
每个类必须至少有一个指定的初始化方法。某些情况下，通常会从父类继承一个或多个指定的初始化方法。
便捷初始化方法是一个类的次要的，辅助性的初始化方法。你可以定义一个便捷初始化方法，用此方法来调用该类中的指定初始化方法，将指定初始化方法中的某些参数设为默认值。
如果你的类不需要便捷初始化方法，你就无需提供它们。只要通用初始化类型的快捷方式可以节省时间，或者让类的初始化在内容是更清晰，那么就创建便捷初始化方法。

#### Syntax for Designated and Convenience Initializers

```swift
init(parameters) {
    statements
}
convenience init(parameters) {
    statements
}
```

#### Initializer Delegation for Class Types

为了简化指定和便捷初始化方法之间的关系，Swift定义了下面3条规则：
- *指定初始化方法必须调用它直接父类的指定初始化方法*
- *便捷初始化方法必须调用调用同类中的其他初始化方法*
- *便捷初始化方法最总必须要调用指定初始化方法*

一个简单的记忆就是：
**指定初始化方法必须向上委派（父类）**
**便捷初始化方法必须横向委派（当前类）**

官方有几幅图画的很好，大家可以随便看看：

![](/assets/images/2019/initializerDelegation01.png)

![](/assets/images/2019/initializerDelegation02.png)

#### Two-Phase Initialization

两步初始化
第一步，每个存储属性被指定一个初始值，一旦每个存储属性被初始化，第二部就开始了，这是每个类都可以更深层次的定义它们的属性，在新的实例准备好使用之前。
两步初始化阻止属性在初始化之前被访问，同时防止了属性被另一个初始化方法设为一个不同的值。
Swift编译器通过四步安全检查来保证两阶段初始化正常完成。
**Safety check 1**
指定初始化方法必须保证类中自己声明的属性初始化完成，然后再委派到父类的初始化方法。
**Safety check 2**
指定初始化方法必须委派到父类的初始化方法（`此处的初始化方法应该是指定初始化方法`），然后才能修改从父类继承的属性。
**Safety check 3**
便捷初始化方法先调用其他的初始化方法，然后才能修改类中的其他属性（`最终会调用指定初始化方法，如果先设置属性，会被指定初始化方法设定的值覆盖`）。
**Safety check 4**
初始化方法不能调用实例方法（初始化的时候还没有生成类的实例对象），不能读取实例的任何属性，不能将self当作值去使用（`不能引用self，但可以使用self.来初始化属性的值`）

只有第一阶段完成以后，类才是有效。
**Phase 1**
- 调用类的指定或便捷初始化方法
- 初始化类的一个新实例的内存，但是内存并未初始化
- 指定初始化方法保证所有存储属性都被初始化，这些存储属性的内存现在已经初始化
- 指定初始化方法调用父类初始化方法去初始化从父类继承而来的属性
- 继续初始化直到继承链的顶端
- 到达继承链的顶端以后，类的所有属性都已经初始化完成，实例的内存已经完全初始化，第一阶段完成。

**Phase 2**
- 从继承链的顶端开始，每一个指定初始化方法都有机会去深层次的自定义实例，这时候初始化方法以及可以访问`self`，可以改变`self`的属性，调用`self`的实例方法。（`说白了就是只要初始化完了类的所有属性，该实例就已经初始化完成了，你就可以调用self的属性和方法了`）
- 最终，便捷初始化方法可以去自定义实例，使用`self`。（`和上面对应起来，便捷初始化方法必须先要调用其他初始化方法，然后才能对属性进行操作`）

![](/assets/images/2019/twoPhaseInitialization01.png)

![](/assets/images/2019/twoPhaseInitialization02.png)

#### Initializer Inheritance and Overriding

初始化方法的继承和重写
和OC不同，Swift的子类默认不会继承父类的初始化方法。这么做是为了防止一个非常复杂的子类初始化时，调用了从父类继承的初始化方法，从而导致子类的属性没有完全的、正确的初始化。
当子类的初始化方法和父类的指定初始化方法一致时，你已经提供了父类初始化方法的重写，你必须使用`override`关键字在初始化方法面前标明。（`即使重写了默认初始化方法也需要标明，不过编译器已经很智能的帮你添加了`）
与重写属性、方法、下标一样，`override`修饰符提示Swift检查父类是否具有要覆盖的指定初始化方法，并验证参数已按预期指定。
>Note：
当重写父类的指定初始化方法是，总是需要`override`修饰符，即使子类实现的初始化方法是便捷初始化方法。

相反，如果你子类的初始化方法和父类的便捷初始化方法一样，子类无法直接调用父类的便捷初始化方法，因此你的子类（从严格意义上来说）并没有提供父类初始化方法的重写。因此，你无需使用`override`修饰符。
```swift
//没提供初始化方法，编译器生成默认的初始化方法
class Vehicle {
    //存储属性
    var numberOfWheels = 0
    //计算属性
    var description: String {
        return "\(numberOfWheels) wheel(s)"
    }
}
//使用默认初始化方法
let vehicle = Vehicle()
print("Vehicle: \(vehicle.description)")
// Vehicle: 0 wheel(s)

class Bicycle: Vehicle {
    
    //指定初始化方法，因为和父类相同，所以要用override
    override init() {
        super.init()
        //初始化之后才可以修改父类继承的属性
        numberOfWheels = 2
    }
}

let bicycle = Bicycle()
print("Bicycle: \(bicycle.description)")
// Bicycle: 2 wheel(s)
```

如果子类在第二阶段不需要在进行自定义操作，并且父类有一个0参数的指定初始化方法，在子类的属性初始化完成以后，你可以省略`super.init()`方法的调用。

```swift
class Hoverboard: Vehicle {
    var color: String
    init(color: String) {
        self.color = color
        //此处省略了super.init的调用
        // super.init() implicitly called here
    }
    override var description: String {
        return "\(super.description) in a beautiful \(color)"
    }
}
let hoverboard = Hoverboard(color: "silver")
print("Hoverboard: \(hoverboard.description)")
// Hoverboard: 0 wheel(s) in a beautiful silver
```

>NOTE
子类可以改变从父类继承的变量，但是无法修改继承而来的常量。

#### Automatic Initializer Inheritance

子类默认不会继承父类的初始化方法，但是特定条件下，父类的初始化方法会被自动继承。

**Rule 1**
- 如果子类没有定义任何`指定初始化方法`，它将自动继承父类所有的指定初始化方法。

**Rule 2**
- 如果子类提供了父类指定初始化方法的所有实现，（无论是通过Rule 1继承而来，`还是通过提供自定义实现作为其定义的一部分`），那么它会自动继承父类的便捷初始化方法。
>NOTE
子类可以实现父类的指定初始化方法作为自身的便捷初始化方法，也满足Rule 2

#### Designated and Convenience Initializers in Action

```swift
class Food {
    var name: String
    //指定初始化方法
    init(name: String) {
        self.name = name
    }
    //便捷初始化方法
    convenience init() {
        //直接调用指定初始化方法，需要使用self进行调用，这里的self只能调用初始化方法，
        //以及为属性设定值，无法调用实例方法，次句之后才可以调用实例方法
        self.init(name: "[Unnamed]")
    }
}
let namedMeat = Food(name: "Bacon")
// namedMeat's name is "Bacon"

let mysteryMeat = Food()
// mysteryMeat's name is "[Unnamed]"
```

![initializersExample01](/assets/images/2019/initializersExample01.png)

类没有默认的成员初始化方法，所以需要提供一个指定初始化方法。Food类没有父类，所有初始化自身属性以后无需在调用super.init()

```swift
class RecipeIngredient: Food {
    var quantity: Int
    //指定初始化方法，会调用父类指定初始化方法
    init(name: String, quantity: Int) {
        self.quantity = quantity
        super.init(name: name)
    }
    //虽然是便捷初始化方法，但是和父类指定初始化方法相同，必须标明override
    override convenience init(name: String) {
        self.init(name: name, quantity: 1)
    }
}
let oneMysteryItem = RecipeIngredient()
//name: "[Unnamed]" quantity: 1
let oneBacon = RecipeIngredient(name: "Bacon")
//name: "Bacon" quantity: 1
let sixEggs = RecipeIngredient(name: "Eggs", quantity: 6)
//name: "Eggs" quantity: 9
```

![initializersExample02](/assets/images/2019/initializersExample02.png)

尽管RecipeInngredient提供了init(name: String)便捷初始化方法，但是相当于提供了父类指定初始化方法的实现，因此RecipeIngredient自动继承父类的所有便捷初始化方法。
RecipeIngredient继承Food的`convenience init()`，但是调用该方法时的`init
(name:)`，它会委派到自己`override`的`init(name:)`方法

```swift
class ShoppingListItem: RecipeIngredient {
    var purchased = false
    var description: String {
        var output = "\(quantity) x \(name)"
        output += purchased ? " ✔" : " ✘"
        return output
    }
}

var breakfastList = [
    ShoppingListItem(),
    ShoppingListItem(name: "Bacon"),
    ShoppingListItem(name: "Eggs", quantity: 6),
]
breakfastList[0].name = "Orange juice"
breakfastList[0].purchased = true
for item in breakfastList {
    print(item.description)
}
// 1 x Orange juice ✔
// 1 x Bacon ✘
// 6 x Eggs ✘
```

>NOTE
ShoppingListItem没有定义初始化方法为purchased提供初始值，因为购物清单上的item开始是总是未购买状态。因为未定义任何初始化方法，它将从父类继承所有的指定和便捷初始化方法。

![initializersExample03](/assets/images/2019/initializersExample03.png)


# Failable Initializers

使用`init?`来定义可能失败的初始化方法
>NOTE
你可以用相同的名字定义可失败和不可失败的初始化方法
可失败的初始化方法初始化了一个可选的值类型，当初始化返回`nil`时，就表明初始化失败。
严格来说，初始化方法不返回值，它的作用是保证`self`的完全初始化，尽管你`return nil`来触发初始化失败，但是初始化成功时并不需要返回值。

```swift
let wholeNumber: Double = 12345.0
let pi = 3.14159

if let valueMaintained = Int(exactly: wholeNumber) {
    print("\(wholeNumber) conversion to Int maintains value of \(valueMaintained)")
}
// Prints "12345.0 conversion to Int maintains value of 12345"

let valueChanged = Int(exactly: pi)
// valueChanged is of type Int?, not Int

if valueChanged == nil {
    print("\(pi) conversion to Int does not maintain value")
}
// Prints "3.14159 conversion to Int does not maintain value"
```

```swift
struct Animal {
    let species: String
    init?(species: String) {
        if species.isEmpty { return nil }
        self.species = species
    }
}
let someCreature = Animal(species: "Giraffe")
// someCreature is of type Animal?, not Animal

if let giraffe = someCreature {
    print("An animal was initialized with a species of \(giraffe.species)")
}
// Prints "An animal was initialized with a species of Giraffe"

let anonymousCreature = Animal(species: "")
// anonymousCreature is of type Animal?, not Animal

if anonymousCreature == nil {
    print("The anonymous creature could not be initialized")
}
// Prints "The anonymous creature could not be initialized"
```

#### Failable Initializers for Enumerations

你可以使用一个可失败的初始化方法根据一个或多个参数来选择一个合适的枚举，如果提供的参数不匹配合适的枚举值，那么返回失败。

```swift
enum TemperatureUnit {
    case kelvin, celsius, fahrenheit
    init?(symbol: Character) {
        switch symbol {
        case "K":
            self = .kelvin
        case "C":
            self = .celsius
        case "F":
            self = .fahrenheit
        default:
            return nil
        }
    }
}
let fahrenheitUnit = TemperatureUnit(symbol: "F")
if fahrenheitUnit != nil {
    print("This is a defined temperature unit, so initialization succeeded.")
}
// Prints "This is a defined temperature unit, so initialization succeeded."

let unknownUnit = TemperatureUnit(symbol: "X")
if unknownUnit == nil {
    print("This is not a defined temperature unit, so initialization failed.")
}
// Prints "This is not a defined temperature unit, so initialization failed."
```

#### Failable Initializers for Enumerations with Raw Values

具有初始值的枚举会自动接收一个`init?(rawValue:)`方法，这个方法有一个raw-value类型的叫`rawValue`的参数，如果参数匹配，则返回一个相匹配的枚举，如果不匹配则初始化失败。

```swift
enum TemperatureUnit: Character {
    case kelvin = "K", celsius = "C", fahrenheit = "F"
}

let fahrenheitUnit = TemperatureUnit(rawValue: "F")
if fahrenheitUnit != nil {
    print("This is a defined temperature unit, so initialization succeeded.")
}
// Prints "This is a defined temperature unit, so initialization succeeded."

let unknownUnit = TemperatureUnit(rawValue: "X")
if unknownUnit == nil {
    print("This is not a defined temperature unit, so initialization failed.")
}
// Prints "This is not a defined temperature unit, so initialization failed."
```

#### Propagation of Initialization Failure

初始化失败的传播
类、结构体、枚举的可失败构造器可以委派到相同类、结构体、枚举的另一个可失败构造器，相似的，子类的可失败构造器可以委派到父类的可失败构造器。
如果委派到的构造器初始化失败，那么整个初始化流程就会失败，不会再进行后续的初始化操作。
>NOTE
可失败构造器也可以委派给一个不会失败的构造器，通过这种方式，你需要将一个潜在的失败状态添加到现有的不会失败的初始化过程中

```swift
class Product {
    let name: String
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}

class CartItem: Product {
    let quantity: Int
    init?(name: String, quantity: Int) {
        if quantity < 1 { return nil }
        self.quantity = quantity
        super.init(name: name)
    }
}

if let twoSocks = CartItem(name: "sock", quantity: 2) {
    print("Item: \(twoSocks.name), quantity: \(twoSocks.quantity)")
}
// Prints "Item: sock, quantity: 2"

if let zeroShirts = CartItem(name: "shirt", quantity: 0) {
    print("Item: \(zeroShirts.name), quantity: \(zeroShirts.quantity)")
} else {
    print("Unable to initialize zero shirts")
}
// Prints "Unable to initialize zero shirts"

if let oneUnnamed = CartItem(name: "", quantity: 1) {
    print("Item: \(oneUnnamed.name), quantity: \(oneUnnamed.quantity)")
} else {
    print("Unable to initialize one unnamed product")
}
// Prints "Unable to initialize one unnamed product"
```

#### Overriding a Failable Initializer

你可以直接重写父类的可失败构造器，也可以用一个不可失败构造器来重写父类的可失败构造器，这就需要你定义一个拥有不会失败构造器的子类，即使父类的构造器允许失败。
如果你用了一个不可失败构造器重写了父类的可失败构造器，向上委派的唯一方法是force-unwrap父类可失败构造器的结果。
>你可以用一个不可失败构造器重写可失败构造器，但是反过来不行

```swift
class Document {
    var name: String?
    // this initializer creates a document with a nil name value
    init() {}
    // this initializer creates a document with a nonempty name value
    init?(name: String) {
        if name.isEmpty { return nil }
        self.name = name
    }
}

class AutomaticallyNamedDocument: Document {
    override init() {
        super.init()
        self.name = "[Untitled]"
    }
    override init(name: String) {
        super.init()
        if name.isEmpty {
            self.name = "[Untitled]"
        } else {
            self.name = name
        }
    }
}
```

AutomaticallyNamedDocument使用不可失败的`init(name:)`初始化方法重写其父类的可用`init?(name:)`方法。 因为AutomaticallyNamedDocument以与其父类使用不同的方式处理空字符串，所以它的初始化不会失败，因此它提供了初始化的不可失败版本。
您可以在构造器中使用强制解包来调用父类的构造器，作为子类的不可失败构造器的实现的一部分。 例如，下面的UntitledDocument子类总是命名为“[Untitled]”，它在初始化期间使用来自其父类的可失败构造器`init(name:)`（`因为它不可能初始化失败，所以可以使用强制拆包`）。

```swift
class UntitledDocument: Document {
    override init() {
        super.init(name: "[Untitled]")!
    }
}
```

#### The init! Failable Initializer

你通常会定义一个可失败的构造器，通过在`init`关键字后面放置一个`?`来创建相应类型的可选实例。 或者，可以定义一个可失败的构造器，用于创建相应类型的隐式解包的可选实例。 通过在`init`关键字之后放置一个`!`而不是`?`来做到这一点。
你可从`init?`委派到`init!`，反之亦然。你也可以用`init!`来重写`init?`，反之亦然。你也可以从`init`委托到`init!`，如果`init!`初始化失败会触发断言导致初始化失败。

# Required Initializers
在构造函数之前添加`required`的修饰符，表示该类的每个子类都必须实现该构造函数：

```swift
class SomeClass {
    required init() {
        // initializer implementation goes here
    }
}
```

同时，还必须在每个子类的构造函数之前添加`required`修饰符，来表明构造函数适用于链中的其他子类。 在覆盖`required`构造函数是时，不要`override`：

```swift
class SomeSubclass: SomeClass {
    required init() {
        // subclass implementation of the required initializer goes here
    }
}
```

# Setting a Default Property Value with a Closure or Function

如果存储属性的默认值需要某些自定义设置，可以使用闭包或全局函数为该属性提供自定义的默认值。 每当初始化属性所属类型的新实例时，都会调用闭包或函数，并将其返回值指定为属性的默认值。
这些类型的闭包或函数通常会创建与属性相同类型的临时值，定制该值以表示所需的初始状态，然后返回该临时值以用作属性的默认值。

```swift
class SomeClass {
    let someProperty: SomeType = {
        // create a default value for someProperty inside this closure
        // someValue must be of the same type as SomeType
        return someValue
    }()
}
```

>请注意，闭包的结束大括号后面是一对空括号。 这告诉Swift立即执行闭包。 如果省略这些括号，则尝试将闭包本身分配给属性，而不是闭包的返回值。如果使用闭包来初始化属性，请记住在执行闭包时尚未初始化实例的其余部分。 这意味着您无法从闭包中访问任何其他属性值，即使这些属性具有默认值。 您也不能使用隐式self属性，也不能调用任何实例的方法。

# THE END
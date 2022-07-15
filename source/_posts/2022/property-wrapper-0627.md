---
title: "Property Wrapper - 属性包装器"
catalog: true
toc_nav_num: true
date: 2022-06-27 17:01:44
subtitle: "Property wrappers in Swift"
header-img: "/img/article_header/article_header.png"
busuanzi: true
tags:
- iOS - Swift

---

# 前言
在Swift 5.1中新增了`Property Wrapper`这个新的特性，能够将一些行为和逻辑直接附加到我们的属性上，可能极大提高我们代码的重用性，所以我们本次就来介绍这个`Property Wrapper`，同时分享针对一些场景的高效用法。

# 简单使用
顾名思义，属性包装器本质上是一种对定值提供额外逻辑包装的类型，通过`@propertyWrapper`来标识结构体或类实现。`Property Wrapper`包含一个名为`wrappedValue`的存储属性，该属性告诉`Swift`正在包装哪个底层值。

举个例子，我们希望创建一个`Property Wrapper`，自动对标识属性做`String`大写，可以如下实现：

```swift
@propertyWrapper
struct Capitalized {
    var wrappedValue: String {
        didSet { wrappedValue = wrappedValue.capitalized }
    }
    
    init(wrappedValue: String) {
        self.wrappedValue = wrappedValue.capitalized
    }
}
```
声明结构体`Capitalized`使用`@propertyWrapper`标识后，我们就可以对于任意`String`属性进行`@Capitalized`就行修饰，同时可以使用正常字符串的形式一样处理对应属性，如下：
> 注意我们对于`wrappedValue`类型的声明，这将限制使用`@Capitalized`修饰属性的类型。

``` swift
struct User {
    @Capitalized var firstName: String
    @Capitalized var lastName: String
}

var user = User(firstName: "qiu", lastName: "chong")
print(user.firstName, user.lastName) // Qiu Chong

user.lastName = "qian"
print(user.lastName) // Qian
```
同时只要`Property Wrapper`定义了初始化方法，我们就可以为包装的属性分配默认值：
``` swift
struct Document {
    @Capitalized var name = "untitled document"
}

print(Document().name) // Untitled Document
```
以上就是`Property Wrapper`的简单使用，能够透明包装和修改任意存储的属性。

# 属性包装器的属性
`Property Wrapper`可以拥有自己的属性，甚至可以依赖注入。
举个例子，当我们使用`UserDefaults`API存储轻量数据时，通常我们会写比较多的重复代码，但是我们使用属性包装器里面的属性实现这种逻辑：
``` swift
@propertyWrapper
struct UserDefaultsBacked<Value> {
    let key: String
    var storage: UserDefaults = .standard
    
    var wrappedValue: Value? {
        get { storage.value(forKey: key) as? Value }
        set { storage.setValue(newValue, forKey: key) }
    }
}
```
与普通的结构体一样，`UserDefaultsBacked`结构体将自动拥有初始化方法，拥有默认值的属性可无需再次初始化，所以我们只需要指定`key`的值：
``` swift
struct SettingsViewModel {
    @UserDefaultsBacked<String>(key: "UserName")
    var userName
    
    @UserDefaultsBacked(key: "IsVip")
    var isVip: Bool?
}
```
> `UserDefaultsBacked`使用了类型推断，所以上述两种写法都可以
上述的两个属性在读取的时候其实都是可选值，即使在声明的时候声明为非可选属性，但是它的实际值仍然是可选的，因为`UserDefaultsBacked`的内部类型指定了`Value?`作为`wrappedValue`属性的类型，所以我们可以对这个属性包装器进行优化，为其提供一个默认值：
``` swift
@propertyWrapper
struct UserDefaultsBacked<Value> {
    let defaultValue: Value
    let key: String
    var storage: UserDefaults = .standard
    
    var wrappedValue: Value {
        get { storage.value(forKey: key) as? Value ?? defaultValue }
        set { storage.setValue(newValue, forKey: key) }
    }
    
    init(wrappedValue defaultValue: Value,
         key: String,
         storage: UserDefaults = .standard) {
        self.defaultValue = defaultValue
        self.key = key
        self.storage = storage
    }
}

struct SettingsViewModel {
    @UserDefaultsBacked(key: "UserName")
    var userName: String = "Q"
    
    @UserDefaultsBacked(key: "IsVip")
    var isVip: Bool = false
}
```
但实际开发过程中，我们的`UserDefaults`存储的值可能是可选的，所以我们不能将`nil`作为这些属性的默认值，所以我们可以再次优化，每当`Value`类型符合`ExpressibleByNilLiteral`协议时，我们将自动插入`nil`作为默认值：
``` swift
extension UserDefaultsBacked where Value: ExpressibleByNilLiteral {
    init(key: String, storage: UserDefaults = .standard) {
        self.defaultValue = nil
        self.key = key
        self.storage = storage
    }
}

struct SettingsViewModel {
    @UserDefaultsBacked(key: "UserName")
    var userName: String = "Q"
    
    @UserDefaultsBacked(key: "IsVip")
    var isVip: Bool = false
    
    @UserDefaultsBacked(key: "ID")
    var id: String?
}
```
因为我们可以将`nil`设置到`UserDefaultsBacked`中，为了避免`UserDefaults`存储崩溃，所以我们可以再升级我们的`UserDefaultsBacked`，判断分配的值是否为`nil`，如果不是则存储，是的话就删除存储的值：
``` swift
private protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}

@propertyWrapper
struct UserDefaultsBacked<Value> {
    var wrappedValue: Value {
        get { ... }
        set {
            if let optional = newValue as? AnyOptional, optional.isNil {
                storage.removeObject(forKey: key)
            } else {
                storage.setValue(newValue, forKey: key)
            }
        }
    }
    ...
}
```
然后，我们就可以很容易的在项目中进行轻量数据的存储，不用写过多的代码，只需要在所需要的属性前加上`@UserDefaultsBacked`修饰符就可以实现，让我们的代码变得非常整洁，同时也能充分利用到Swift强大的类型系统。

# 属性包装器的预测值

`Property Wrapper`的好处就是我们可以在不影响我们调用的属性的情况下给属性添加逻辑和方法，使用方式和没有`Property Wrapper`的属性一样读取和写入，但有时候我们想要获取`Property Wrapper`本身的一些属性或状态，而不是包装只有的属性值，这个时候我们就要用到预测值`Projected Value`。

举个例子，我们对`Capitalized`进行改造，可以获取到`String`属性的原值：
``` swift
@propertyWrapper
struct Capitalized {
    var projectedValue: String

    var wrappedValue: String {
        didSet {
            projectedValue = wrappedValue
            wrappedValue = wrappedValue.capitalized
        }
    }
    
    init(wrappedValue: String) {
        self.projectedValue = wrappedValue
        self.wrappedValue = wrappedValue.capitalized
    }
}
```
我们在`Property Wrapper`中声明了`ProjectedValue`属性，这样我们就可以通过`$`获取到`Property Wrapper`中内置的属性或状态了：
``` swift
var user = User(firstName: "qiu", lastName: "chong")
print(user.firstName, user.lastName, user.$firstName, user.$lastName) // Qiu Chong qiu chong

user.lastName = "qian"
print(user.lastName, user.$lastName) // Qian qian
```

# Codable解码

我们在`swift`中使用`Codable`解码时会经常遇到一个问题，就是设置默认值。通常情况下，我们可以将值设为可选值或在每个数据结构中重写`init:decoder:`方法，但是两者或多或少都会有点复杂，一个是可选值在使用过程中需要解包，另一个重写`init:decoder:`方法会导致代码臃肿，如下问题：

## 问题一
使用可选值解决解包时`key`不存在的问题：
``` swift
struct Video: Decodable {
    let id: Int
    let title: String
    let commentEnabled: Bool?
}
```
对于数据：
``` json
{ "id": 12345, "title": "My First Video" }
```
将解码得到：
``` swift
// Video(id: 12345, title: "My First Video", commentEnabled: nil)
```
可选值会让代码变得很丑，同时使用起来变得麻烦，并不是很优雅。

## 问题二
重写`init:decoder:`方法：
``` swift
enum State: String, Decodable {
    case streaming
    case archived
    case unknown
}

struct Video: Decodable {
    let id: Int
    let title: String
    let commentEnabled: Bool
    let state: State
    
    enum CodingKeys: String, CodingKey {
        case id, title, commentEnabled, state
    }
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(Int.self, forKey: .id)
        title = try container.decode(String.self, forKey: .title)
        commentEnabled = try container.decodeIfPresent(Bool.self, forKey: .commentEnabled) ?? false
        state = try container.decodeIfPresent(State.self, forKey: .state) ?? .unknown
    }
}
```
这个确实能解决可选值遇到的问题，但是需要对于每个模型都写一套很完整的代码，工作量不小，这也是我们项目中目前用到的，比较粗糙。

## Property Wrapper
所以我们可以利用`Property Wrapper`给出中比较优雅的方式来解决，在`key`不存在或解码失败时，为某个属性设置默认值。

我们期望对于属性的设置可以：
``` swift
@Default(value: true) var commentEnabled: Bool

@Default(value: .unknown) var state: State
```
所以对于`Default`有如下声明：
``` swift
@propertyWrapper
struct Default<T: Decodable> {
    let value: T
    
    var wrappedValue: T {
        get { fatalError("未实现") }
    }
}
```
但是如果按照上述方式修复属性，属性的类型就不是单纯的`Bool`或`State`类型，而是`Default<T>`类型，这时候所期望的数据格式是：
``` json
{ "id": 12345, "title": "My First Video", "commentEnabled": { "value": true } }
```
所以很显然，这还不是我们想要的东西。

那么我们就换种思路，使用类型约束传值，定义一个`protocol`来规定默认值：
``` swift
protocol DefaultValue {
    associatedtype Value: Decodable
    static var defaultValue: Value { get }
}
```

然后让`Bool`满足这个默认值：
``` swift
extension Bool: DefaultValue {
    static let defaultValue = false
}
```

在这里，`DefaultValue.Value`的类型会根据`defaultValue`的类型自动被推断为`Bool`。
接下来，重新定义`Default`这个`Property Wrapper`，以及用于解码的初始化方法：
``` swift
@propertyWrapper
struct Default<T: DefaultValue> {
    var wrappedValue: T.Value
}

extension Default: Decodable {
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        wrappedValue = (try? container.decode(T.Value.self)) ?? T.defaultValue
    }
}
```

这样我们就可以用这个新的`Default`修饰`commentEnabled`，并对应解码失败的情况了：
``` swift
struct Video: Decodable {
    let id: Int
    let title: String
    @Default<Bool> var commentEnabled: Bool
}

// { "id": 12345, "title": "My First Video", "commentEnabled": 123 }
// Video(id: 12345, title: "My First Video", _commentEnabled: __lldb_expr_31.Default<Swift.Bool>(wrappedValue: false))
```

现在我们已经可以解码类型不同的这类意外输入了，但是如果JSON中`commentEnabled``key`缺失时，解码依然会发生错误。因为我们所使用的解码器默认生成的代码是要求`key`存在，所以我们需要为`container`重写对于`Default`类型解码的实现：
``` swift
extension KeyedDecodingContainer {
    func decode<T>(_ type: Default<T>.Type, forKey key: KeyedDecodingContainer<K>.Key) throws -> Default<T> where T: DefaultValue {
        try decodeIfPresent(type, forKey: key) ?? Default(wrappedValue: T.defaultValue)
    }
}
```
在键值编码的`container`中遇到要解码为`Default`的情况时，如果`key`不存在，则返回`Default(wrappedValue: T.defaultValue)`这个默认值。
所以，现在对于JSON中`commentEnabled`缺失的情况，也可以正确解码了：
``` swift
{ "id": 12345, "title": "My First Video" }
// Video(id: 12345, title: "My First Video", _commentEnabled: __lldb_expr_36.Default<Swift.Bool>(wrappedValue: false))
```

最后，对于类似`Default<Bool>`这样的修饰，只能将默认值解码到`false`，但有时候需要针对不同的情况设置不同的默认值。
`DefaultValue`协议其实并没有对类型做出太多规定：只要所提供的默认值`DefaultValue`：
```
extension Bool {
    enum False: DefaultValue {
        static let defaultValue = false
    }
    enum True: DefaultValue {
        static let defaultValue = true
    }
}
```

这样，我们就可以用这样的类型来定义不同的默认解码值了：
``` swift
@Default<Bool.False> var commentEnabled: Bool
@Default<Bool.True> var publicVideo: Bool
```

或者为了可读性，更进一步，使用`typealias`给它们一些更好的名字：
``` swift
extension Default {
    typealias True = Default<Bool.True>
    typealias False = Default<Bool.False>
}

@Default.False var commentEnabled: Bool
@Default.True var publicVideo: Bool
```

# 总结

`Property Wrapper`绝对是`Swift`中比较强大的功能之一，他为代码重用和可定制性提供了更多的可能性。当然，`Property Wrapper`的透明度既是一种优势，也是一种负担，一方面，它使我们能够与未包装属性完全相同的方式访问和分配包装属性，但另一方面，我们可能在相当不明显的抽象背后隐藏太多的功能。
总之，`Property Wrapper`是非常值得我们学习的，共勉。
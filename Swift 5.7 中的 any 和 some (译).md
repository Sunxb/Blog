[原文地址](https://www.donnywals.com/whats-the-difference-between-any-and-some-in-swift-5-7/)

由于 `any` 和 `some` 都适用于协议，因此我想在这篇博文中将它们放在一起比较以便更好地解释它们解决分别解决了什么问题，以及在什么情况下使用 `any`、`some` 或其他的。

### 了解 any 和 some 解决的问题

为了解释 `any` 解决的问题，我们可以通过一个列子来了解这两个关键字。下面是一个Pizza模型的协议：

```swift
protocol Pizza {
    var size: Int { get }
    var name: String { get }
}
```

在Swift 5.6，你可能会写下面的这种方法，来接收一个Pizza

```swift
func receivePizza(_ pizza: Pizza) {
    print("Omnomnom, that's a nice \(pizza.name)")
}
```

当这个函数被调用时，`receivePizza` 函数接收一个所谓的披萨协议类型，我们可以理解为一个披萨盒子。为了知道这个披萨名称，必须打开这个盒子，也就是获取实现Pizza协议的具体对象，然后获取名称。这意味着 Pizza 几乎没有编译时优化，这使得 `receivePizza` 方法调用的开销比我们想要的更大。

另外下面的函数，看起来好像是一样的

```swift
func receivePizza<T: Pizza>(_ pizza: T) {
    print("Omnomnom, that's a nice \(pizza.name)")
}
```

不过，这里有一个很主要区别。 `Pizza` 协议在这里没有用作参数类型。它被用作泛型 T 的约束。这就使得编译器将能够在编译时解析 T 的类型，使得 `receivePizza` 接受到的是一个具体化的类型。

因为上面这两种方法差异并不是很清楚，所以 Swift 团队引入了 `any` 关键字。此关键字不添加任何新功能。它迫使我们清楚地传达“这是一种存在主义”：(有点拗口，也不是很好理解，我就把他理解成`这么类型的一个东西`)

```swift
// 上面的第一种写法，增加一个any关键字
func receivePizza(_ pizza: any Pizza) {
    print("Omnomnom, that's a nice \(pizza.name)")
}
```

使用泛型T的示例不需要 `any` 关键字，因为 `Pizza` 被用作约束而不是存在类型。

现在我们对 `any` 有了更清晰的了解，继续让我们研究一下 `some`。

在 Swift 中，许多开发人员都写如下代码：

```swift
let someCollection: Collection
```

我们会遇到编译器错误，告诉我们 `Collection` 有 Self 或关联的类型要求。在 Swift 5.1 中，我们可以告诉编译器任何访问 someCollection 的人都不应该关心这些。他们应该只需要知道这个东西符合 `Collection` 协议 ，仅此而已。

这种机制对于使 SwiftUI 的 View 协议至关重要。

但也有缺点，那就是在使用`some Colelction`的时候，无法知道其中的关联类型是什么。

然而，并非所有协议都有相关的关联类型。再次考虑下面这个`receivePizza`版本：

```swift
func receivePizza<T: Pizza>(_ pizza: T) {
    print("Omnomnom, that's a nice \(pizza.name)")
}
```

我们定义了一个通用的 T 来允许编译器针对给定的具体类型的 Pizza 进行优化。 some 关键字还允许编译器在编译时知道 some 对象的底层实际类型是什么；它只是对我们隐藏。这正是 <T: Pizza> 所做的。我们通过 T 这个类型访问也只能访问到 Pizza 协议所公开的内容。这意味着我们可以重写 receivePizza<T: Pizza>(_:) 如下：

```swift
func receivePizza(_ pizza: some Pizza) {
    print("Omnomnom, that's a nice \(pizza.name)")
}
```

我们不再需要 T ，也就是我们不需要创建一个类型来代表我们的 Pizza。我们可以说“这个函数需要`some Pizza`”而不是“这个函数需要一些我们称之为 T 的Pizza”。这两种写法是等价的。

### 选择 some 还是 any

其实当我们了解了some 和 any, 就会知道这不选其中一个的问题，他们都解决自己的问题的。

一般来说，我们要尽可能的使用 some 或者泛型，就拿我们的 Pizza 来说，如果使用any, 就好比我们在运行时也会是接收到一个Pizza类型的盒子，具体是什么Pizza, 还需要我们再打开盒子，但是some 或者泛型，就会给我们一个实际的Pizza类型了。

### 实践

让我们再举一个例子来说明这一点，这个例子在很大程度上借鉴了我对主要关联类型的解释。

```swift
class MusicPlayer {
    var playlist: any Collection<String> = []

    func play(_ playlist: some Collection<String>) {
        self.playlist = playlist
    }
}
```

在这段代码中，我使用了 `some Collection<String> `而不是编写 `func play<T: Collection<String>>(_ playlist: T) `，因为泛型只在这一个地方使用。

我的 `var playlist` 是 `any Collection<String> `而不是 `some Collection<String> `有两个原因：

1. 无法确保编译器将为play方法推导的具体Colection与为var playlist推导的具体Colection相匹配；这意味着它们可能不一样。

2. 编译器首先无法推断`var playlist:some Collection<String>`（尝试一下，你会得到一个编译器错误）

我们可以用下面的写法来避免使用 any ：

```swift
class MusicPlayer<T: Collection<String>> {
    var playlist: T = []

    func play(_ playlist: T) {
        self.playlist = playlist
    }
}
```

但是这样就会强制我们的 T 为同一类型，比如说我们在使用时 T 是Array, 那我们在play方法中就不能在传入其他的Collection类型，比如说Set。但是前面的那种写法是可以的。

### 总结

虽然 some 和 any 听起来很复杂（事实上确实如此），但它们也是 Swift 5.7 中非常强大和重要的部分。理解他们是很有必要的，因为这可以帮助我们更好地理解 Swift 如何处理泛型和协议。


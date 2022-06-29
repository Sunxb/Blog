我们使用写时复制 copy on write 的思想，对 NSMutableData 进行封装，以此来理解我们的标准库的实现方式。

标准库中提供的所有的基本集合类型都是值类型，通过写时复制的思想保证了他的高效性。集合类型是我们比较常用到的数据类型，所以了解他的性能特性很重要，我们来一起看一下写时复制是如何工作的，并且尝试自己手动实现一个。

### 引用类型

举个例子，我们比较一下Swift的Data（结构体）和Foundation库中的NSMutableData（类）。首先我们使用一些字节数据来初始化 NSMutableData 实例。

```swift
var sampleBytes: [UInt8] = [0x0b,0xad,0xf0,0x0d]
let nsData = NSMutableData(bytes: sampleBytes, length: sampleBytes.count)
```

我们使用了 let 来声明 nsData，但是像 NSMutableData 这样的引用类型不收let/var 的控制。对于引用类型来说，用 let 声明代表 nsData 这个指针不能在指向别的内存，但是他指向的这个内存中的数据是可以变化的。也就是说我们依然可以往 nsData 中 append 数据。

```swift
nsData.append(sampleBytes, length: sampleBytes.count)
```

当我们再声明一个对象，改变其中一个对象，另一个对象也会发生变化。

```swift
let nsOtherData = nsData
nsData.append(sampleBytes, length: sampleBytes.count)
// nsOtherData 也会变
```

如果我们想产生一个独立的副本，我们需要使用 mutableCopy（返回一个 Any 类型），我们需要把返回值强转成我们需要的 NSMutableData 类型。

```swift
let nsOtherData = nsData.mutableCopy() as! NSMutableData
nsData.append(sampleBytes, length: sampleBytes.count)
// nsOtherData 不变
```

### 值类型

首先我们也是通过 sampleBytes 来初始化一个 Data。

```swift
let data = Data(bytes: sampleBytes, count: sampleBytes.count)
```

如果我们使用 let 关键字，那编译器就不会允许我们调用类型 append 这样的方法。所以如果要改变 data 的值，要使用 var 。

```swift
var data = Data(bytes: sampleBytes, count: sampleBytes.count)
data.append(contentsOf: sampleBytes)
```

Data 和 NSData 最主要的不同之处是：把值赋给另一个变量时或者作为参数传到方法中，Data 总是会生成一个新的副本，但是 NSData 只会生成一个新的引用，但是两个引用指向同一个内存区域。

当我们创建 Data 的一个副本的时候，他的所有的字段都会被复制，但是又不是立刻复制，因为 Data 内存有对实际内存空间的引用，所以当结构体被复制时，也只是会生成一个新的引用，只有我们对这个新的引用修改数据是，实际的数据才会被复制。

### 实现写时复制

我们自己实现一个 Data 类型来帮我们理解写时复制是如何工作的，我们内部使用 NSMutableData 来实际的存储数据（只是为了更快的完成，实际的Data 内部肯定是用到更底层的数据结构来存储数据）。改变数据的方法我们只实现一个 append 方法。

```swift
struct MyData {
    var data = NSMutableData()
    
    func append(_ bytes: [UInt8]) {
        data.append(bytes, length: bytes.count)
    }
}
```

我们可以创建一个 MyData

```swift
let data = MyData()
```
 为了能更好的打印出 data 中存储的数据，我们可以让 MyData 实现 CustomDebugStringConvertible 协议。
 
```swift
extension MyData: CustomDebugStringConvertible {
    var debugDescription: String {
        return String(describing: data)
    }
}
```

现在我们可以调用 append 方法了。

```swift
data.append(sampleBytes)
```

但这是有问题的，首先我们的MyData是结构体，而且创建 data 使用的是let，我们不应该可以修改他的值。

而且看下面的代码，他的复制行为也是有问题的，在我们声明了一个新的引用是，并没有获得一个完全独立的副本。

```swift
var copy = data
copy.append(sampleBytes)

print(data)
print(copy)
// copy 调用 append, data 也会改变
```

所以说我们虽然创建了一个结构体，但是他并没有表现出值语义来。

目前，我们在把 data 赋给一个新的变量时，虽然他是所有字段都复制，但是我们MyData内部的 data 是一个 NSMutableData 引用类型，所以说 data 和 copy 这两个变量的值现在都包含对同一个 NSMutableData 实例的引用。

为了解决这个问题，我们要先处理写时复制的’写时‘问题。当我们在调用 append 方法添加数据时，我们要把内部进行实际存储功能的data进行深拷贝，此时 我们的 append 方法就必须加上 mutating 关键字，要不然编译器不允许修改结构体的变量。

```swift
struct MyData {
    var data = NSMutableData()
    
    mutating func append(_ bytes: [UInt8]) {
        print("make a copy")
        data = data.mutableCopy() as! NSMutableData
        data.append(bytes, length: bytes.count)
    }
}
```

现在我们要重新生成一个 var 类型的 data 来调用 append 方法，因为编译器不允许let 类型的调用带 mutating 关键字的方法。

```swift
var data = MyData()
var copy = data
copy.append(sampleBytes)
```

在我们继续之前，进行一个小的重构，并将生成 NSMutableData 实例副本的代码提取到一个单独的属性中。

```swift
struct MyData {
    var data = NSMutableData()
    var dataForWriting: NSMutableData {
        mutating get {
            print("make a copy")
            data = data.mutableCopy() as! NSMutableData
            return data
        }
    }
    
    mutating func append(_ bytes: [UInt8]) {
        dataForWriting.append(bytes, length: bytes.count)
    }
}
```

### 让写时复制更高效

目前我们的写时复制是非常简单的，就是每次当我们调用 append 的时候，都会拷贝，不管我们是不是这个实例的唯一持有者。

```swift
for _ in 0..<10 {
    data.append(sampleBytes)
}
// making a copy 会打印10次
```

其实真正需要执行复制操作的是当我们把data赋值给另一个变量后，这时调用append 方法，因为此时有两个引用，所以需要进行深拷贝。当拷贝结束后，这两个都是引用指向的都是完全独立的备份了，所以再一次调用时就不需要拷贝了。

所以说我们的MyData结构没有问题，但是多次拷贝会降低性能。我们可以使用 isKnownUniquelyReferenced 这个方法来帮助我们实现想要的效果。

```swift
var dataForWriting: NSMutableData {
    mutating get {
        if isKnownUniquelyReferenced(&data) {
            return data
        }
        print("make a copy")
        data = data.mutableCopy() as! NSMutableData
        return data
    }
}
```

虽然我们现在加上了 isKnownUniquelyReferenced 检查，但是运行一下测试代码还是会copy多次，那是因为 isKnownUniquelyReferenced 方法只是对Swift类型有效果，如果是传入的OC类型的对象，总是会返回false，所以我们应该使用一个Swift类型来包装一下这个data类型。

```swift
final class Box<A> {
    let unbox: A
    init(_ value: A) {
        self.unbox = value
    }
}
```

我们使用这个Box类来包装 NSMutableData , 最终我们的MyData 变成下面这样子

```swift
struct MyData {
    var data = Box(NSMutableData())
    var dataForWriting: NSMutableData {
        mutating get {
            if isKnownUniquelyReferenced(&data) {
                return data.unbox
            }
            print("make a copy")
            data = Box(data.unbox.mutableCopy() as! NSMutableData)
            return data.unbox
        }
    }
    
    mutating func append(_ bytes: [UInt8]) {
        dataForWriting.append(bytes, length: bytes.count)
    }
}
```

现在我们的代码只对 NSMutableData 实例copy一次。

```swift
var data = MyData()
var copy = data
for _ in 0..<10 {
    data.append(sampleBytes)
}
// Prints:
// making a copy 一次
```

标准库中数组和字典的实现方式其实也是类似的，只是他们用了更低级的数据结构来存储，我们这样手动实现一次写时复制，有助于我们更好理解他们内部的性能。

### 写时复制注意点

写时复制很高效，但是他不是适应于所有的场景，比如说我们上面的for循环是可以的，但是如果我们使用reduce来实现上面的循环，他就不起作用了。

```swift
(0..<10).reduce(data) { result, _ in
    var copy = result
    copy.append(sampleBytes)
    return copy
}
```

这个实现方式会生成 10 个副本，因为当我们调用 append 时，总是有两个变量——copy 和 result——引用指向同一个实例。

所以我们应该注意我们代码中那些产品大量不必要副本的地方，不过我们一般都不会这么写，所以说问题不大。

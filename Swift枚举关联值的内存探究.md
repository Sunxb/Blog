 
 ```swift
enum Season {
    case Spring, Summer, Autumn, Winter
}
let s = Season.Spring
 ```
 
这是枚举最基础的用法，但是在swift中，对枚举的功能进行了加强，也就是关联值。
 
关联值可以将额外信息附加到 enum case中，像下面这样子。

```swift

enum Test {
    case test1(v1: Int, v2: Int, v3: Int)
    case test2(v1: Int, v2: Int)
    case test3(v1: Int)
    case test4
}
let t = Test.test1(v1: 1, v2: 2, v3: 3)
    
switch t {
case .test1(let v1, let v2, let v3):
    print(v1, v2, v3)
default:
    break
}
// 输出： 1 2 3
```

我们可以看到，在我们创建一个枚举值t的时候，设置他的选项为test1，同时可以关联3个Int类型的值，然后在switch中，我们还可以把这3个Int值取出来进行使用。

我们今天的主要任务就是探索一下有关联值的枚举类型，再底层的内存布局是什么样子的，这些值都是怎么储存的。

在OC中我们使用sizeOf此类方法，可以输出一个变量占用内存的大小，在swift中也有此类的工作类，那就是MemoryLayout。

```swift
print(MemoryLayout<Int>.size)// 实际使用内存大小
print(MemoryLayout<Int>.stride)//分配内存大小
print(MemoryLayout<Int>.alignment)//内存对其参数

// 输出 8 8 8 
```

上面的例子是只是简单的实例MemoryLayout的用法，这个我们知道，在64位的系统中Int类型确实是占用8个字节（64位）。接下来我们就看一下枚举的内存占用情况。

点击Xcode菜单栏中的Debug -> Debug Workflow -> View Memory，然后在下面红色框中输入变量的内存地址，就可以看到变量的内存使用情况。

使用swift后，从xcode没法直接打印变量的内存地址，这里我们使用了github上的一个工具类（[github链接](https://github.com/CoderMJLee/Mems)）来帮助我们输出变量的内存地址。

![图1](https://raw.githubusercontent.com/Sunxb/Blog/develop/Img/1.png)

准备工作完成后，我们先从最基础的枚举开始。

```swift

enum Season {
    case Spring, Summer, Autumn, Winter
}
print("实际占用:",MemoryLayout<Season>.size)
print("分配:",MemoryLayout<Season>.stride)
print("对齐参数:", MemoryLayout<Season>.alignment)
    
var s = Season.Spring
print("内存地址",Mems.ptr(ofVal: &s))
    
print("内存数据",Mems.memStr(ofVal: &s, alignment: .one))
    
s = Season.Summer
print("内存数据",Mems.memStr(ofVal: &s, alignment: .one))
    
s = Season.Autumn
print("内存数据",Mems.memStr(ofVal: &s, alignment: .one))
    
s = Season.Winter
print("内存数据",Mems.memStr(ofVal: &s, alignment: .one))

```

注：Mems.memStr可以直接打印内存数据，这样我们就不用每次拿到地址再去工具中看了

```
实际占用: 1
分配: 1
对齐参数: 1
内存地址 0x00007ffee753f0f0
内存数据 0x00
内存数据 0x01
内存数据 0x02
内存数据 0x03
```

我们可以看到这种普通的枚举类型，只占用一个字节。而且通过我们对变量设置不同的枚举值，打印的这一个字节的数据也是不同的，其实也就是使用这一个字节通过设置不同的数值来表示不同的枚举值，这样的话其实可以至少储存0x00-0xFF共256个值。那如果超过256个case呢？其实我觉得没有必要考虑这种情况，枚举本来设计出就是为了区分有限中情况，如果太多，就像200多个，那完全可以使用Int来设置不同的值了，就没必要用枚举了，当然，如果您愿意探究一下的话也是可以的。

接下来我们使用一个带关联值的枚举来看一下。

```swift

enum Test {
    case test1(v1: Int, v2: Int, v3: Int)
    case test2(v1: Int, v2: Int)
    case test3(v1: Int)
    case test4
}
    
print("实际占用:",MemoryLayout<Test>.size)
print("分配:",MemoryLayout<Test>.stride)
print("对齐参数:", MemoryLayout<Test>.alignment)
    
var t = Test.test1(v1: 1, v2: 2, v3: 3)
print("内存地址",Mems.ptr(ofVal: &t))
    
print("内存数据",Mems.memStr(ofVal: &t, alignment: .one))
    
t = Test.test2(v1: 4, v2: 5)
print("内存数据",Mems.memStr(ofVal: &t, alignment: .one))
    
t = Test.test3(v1: 6)
print("内存数据",Mems.memStr(ofVal: &t, alignment: .one))
    
t = Test.test4
print("内存数据",Mems.memStr(ofVal: &t, alignment: .one))

```

下面是输出, 为了能直观一下，我给插了几个换行

```swift
实际占用: 25
分配: 32
对齐参数: 8
内存地址 0x00007ffee0afe0d8
内存数据 
0x01 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x02 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x03 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x00 
0x00 0x00 0x00 0x00 0x00 0x00 0x00
内存数据 
0x04 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x05 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x01 
0x00 0x00 0x00 0x00 0x00 0x00 0x00
内存数据 
0x06 0x00 0x00 0x00 0x00 0x00 0x00 0x00
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x02 
0x00 0x00 0x00 0x00 0x00 0x00 0x00
内存数据 
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x00 0x00 0x00 0x00 0x00 0x00 0x00 0x00 
0x03 
0x00 0x00 0x00 0x00 0x00 0x00 0x00
```

实际占用了25个字节，我们至少可以确定，枚举的关联值是存储在枚举值的内存中的。

但是通过这一个例子其实可能还看不出有什么规律，大家可以多用几个例子来验证，这是我就直接说结论了。

有关联值得枚举实际占用的内存是**最多关联值占用的内存+1**，在我们这个Test中，test1的关联值是最多的，有3个Int类型的关联值，所以要8*3=24字节来存放关联值，但是还需要一个字节来储存（辨别）是哪一个case。

带着这个结论我们看一下输出的结果：

当t=.test1时，前面24个字节分配给3个Int类型关联值，分别存储了1，2，3， 第25个字节是0。

当t=.test2时，前面24个字节还是留给关联值的，但是test2只有两个关联值，所以使用了前面16个字节分配给他的关联值，此时17到24这8字节就空置，第25个字节是1。

...

最后当t = test4 , 没有关联值，所以前面的字节都是0， 只有第25个字节是3

以此类推...

第25个字节其实完全可以看成一个辨识位，或者说第25个字节就是枚举的本质，通过不同值来区分不同case，只是因为有了关联值，所以开辟了更多的空间来存储而已。

后面多余的字节都是为了内存对齐，内存对其相关的知识大家可以自行上网查阅。

补充：
既然说到了关联值，那就顺便对枚举原始值说两句。具通过你打印带原始值的枚举的内存数据，发现是否带有原始值对枚举的内存占用并无影响，所以原始值应该不是存储在枚举变量的内部的。大家可以自己试验一下







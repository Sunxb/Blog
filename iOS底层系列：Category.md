
## 前言

Category是我们平时用到的比较多的一种技术，比如说给某个类增加方法，"添加"属性，或者用Category优化代码结构。

我们通过下面这几个问题作为切入点，结合runtime的源码，探究一下Category的底层原理。

我们在Category中，可以直接添加方法，而且我们也都知道，添加的方法会合并到本类当中，同时我们也可以声明属性，但是此时的属性没有功能，也就是不能存值，这就类似于Swift中的计算属性，如果我们想让这个属性可以储存值，就要用runtime的方式，动态的添加。

## 探究

### 1. Category为什么能添加方法不能"添加"属性

首先我们先创建一个Person类，然后创建一个Person+Run的Category，并在Person+Run中实现-run方法。

我们可以使用命令行对Person+Run.m进行编译

```
xcrun -sdk iphonesimulator clang -rewrite-objc Person+Run.m
```

得到一个Person+Run.cpp文件，在文件的底布，可以找到这样一个结构体

```c++
struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};
```

这些字段几乎都是见名知意了。

每一个Category都会编译然后存储在一个_category_t类型的变量中

```
static struct _category_t _OBJC_$_CATEGORY_Person_$_Run __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"Person",
	0, // &OBJC_CLASS_$_Person,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_Person_$_Run,
	0,
	0,
	0,
};
```
因为我们的Person+Run里面只有一个实例方法，所以从上述代码中来看，也只有对应的位置传值了。

通过这个_category_t的结构结构我们也可以看出，变量存储在_prop_list_t，并不是类中的objc_ivar_list结构体，而且我们都知道property=ivar+set+get，Category中不会生成ivar，所以根本都不能"添加"成员变量。

如果我们在分类中手动为成员变量添加了set和get方法之后，也可以调用，但实际上是没有内存来储值的，这就好像Swift中的计算属性，只起到了计算的作用，就相当于是两个方法（set和get），但是并不能拥有真用的内存来存储值。


### 2. Category的方法是何时合并到类中的

大家都知道Category分类肯定是我们的应用启动是，通过运行时特性加载的，但是这个加载过程具体的细节就要结合runtime的源码来分析了。

runtime源码太多了，我们先通过大概浏览代码来定位实现功能的相关位置。

我从`objc-runtime-new.mm`中找到了下面这个方法。

```
/***********************************************************************
* methodizeClass
* Fixes up cls's method list, protocol list, and property list.
* Attaches any outstanding categories.
* Locking: runtimeLock must be held by the caller
**********************************************************************/
static void methodizeClass(Class cls, Class previously)
```

而且他的注释一些的很清楚，修复类的方法，协议和变量列表，关联还未关联的分类。

然后我们继续找，就找到了我们需要的这个方法。

```
 void attachLists(List* const * addedLists, uint32_t addedCount)
```

我们从其中摘出一段代码来分析就可以解决我们的问题了。

```
 // many lists -> many lists
  uint32_t oldCount = array()->count;
  uint32_t newCount = oldCount + addedCount;
  setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
  array()->count = newCount;
  memmove(array()->lists + addedCount, array()->lists, 
          oldCount * sizeof(array()->lists[0]));
  memcpy(array()->lists, addedLists, 
         addedCount * sizeof(array()->lists[0]));
```

在调用此方法之前我们所有的分类会被方法一个list里面（每一个分类都是一个元素），然后再调用attachLists方法，我们可以看到，在realloc的时候传进一个newCount，这是因为要增加分类中的方法，所以需要对之前的数组扩容，在扩容结束后先调用了memmove方法，在调用memcopy，大家可以上网查一下这两个方法具体的区别，这里简单一说，其实完成的效果都是把后面的内存的内容拷贝到前面内存中去，但是memmove可以处理内存重叠的问题。

其实也就是首先将原来数组中的每个元素先往后移动（我们要添加几个元素，就移动几位），因为移动后的位置，其实也是数组自己的内存空间，所以存在重叠问题，直接移动会导致元素丢失的问题，所以用memmove（会检测是否有内存重叠）。

移动完之后，把我们储存分类中方法的list中的元素移动到数组前面位置。

过程就是这样子了，其实我们第三个问题就顺便解决完了。

### 3. Category方法和类中方法的执行顺序

上面其实说到了，类中原来的方法是要往后面移动的，分类的方法添加到前面的位置，而且调用方法的时候是在list中遍历查找，所以我们调用方法的时候，肯定会先调用到Category中的方法，但是这并不是覆盖，因为我们的原方法还在，只是这中机制保证了如果分类中有重写类的方法，会被优先查找到。

### 4. +load和+initialize的区别

对于这个问题我们从两个角度出发分析，`调用方式`和`调用时刻`。

#### +load

简单的举一个例子，我们创建一个Person类，然后重写+load方法，然后为Person新建两个Category，都分别实现+load。

```
@implementation Person
+ (void)load {
    NSLog(@"Person - load");
}
@end

@implementation Person (Test1)
+ (void)load {
    NSLog(@"Person Test1 - load");
}
@end

@implementation Person (Test2)
+ (void)load {
    NSLog(@"Person Test2 - load");
}
@end
```

当我们进行项目的时候，会得到下面的打印结果。

```
2020-09-14 09:34:41.900161+0800 Category[4533:53426] Person - load
2020-09-14 09:34:41.900629+0800 Category[4533:53426] Person Test1 - load
2020-09-14 09:34:41.900700+0800 Category[4533:53426] Person Test2 - load
```
我们并没有使用这个Person类和他的Category，所以应该是项目运行后，runtime在加载类和分类的时候，就会调用+load方法。

我们从源码中找到下面这个方法
`void load_images(const char *path __unused, const struct mach_header *mh)`

方法中的最后一行调用了`call_load_methods()`，这个`call_load_methods()`中就是实现了+load的调用方式。

下面是`call_load_methods()`函数的实现 ，大家简单浏览一遍

```
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
```

从源码中很清楚的可以看到， 先调用`call_class_loads()`， 再调用`call_category_loads()`，这就说明了在调用所有的+load方法时，实现调用了所有类的+load方法，再去调用分类中的+load方法。

然后我们在进入到`call_class_loads()`函数中

```
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, @selector(load));
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}
```

从中间的循环中可以看出，**是取到了每个类的+load函数的指针，直接通过指针调用了这个函数。** `call_category_loads()`函数中体现出来的Category的+load方法的调用，也是同理。

同时这也解答了我们的另一个疑惑，那就是为什么总是先调用类的+load，在调用Category的+load。

**思考：如果存在继承的情况，+load又会是怎样的调用顺序呢？**

从上面`call_class_loads()`函数中可以看到有一个list：loadable_classes，我们猜测这里面应该就是存放着我们所有的类，因为下面的循环是从0开始循环，所以我们要研究所有的类的+load方法的执行顺序，就要看这个list中的类的顺序是怎么样的。

我们从个源码中可以找到这样一个方法，`prepare_load_methods`，在其实现中调用了`schedule_class_load`方法，我们看一下`schedule_class_load`的源码

```
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    ASSERT(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
```

从源码中` schedule_class_load(cls->superclass);`这一句中可以看出，递归调用自己本身，并且传入自己的父类，结果递归之后，才调用`add_class_to_loadable_list`，这就说明父类总是在子类前面加入到list当中，所有在调用一个类的+load方法之前，肯定要先调用其父类的+load方法。

那如果是其他没有继承关系的类呢，这就跟编译顺序有关系了，大家可以自己尝试验证一下。

小结：

+load方法会在runtime加载类和分类时调用
每个类和分类的+load方法之后调用一次
调用顺序：
先调用类的+load
- 按照编译顺序调用
- 调用子类+load之前，先调用父类的+load
再调用分类的+load
- 按照编译顺序调用

小结中如果有我没提到的，大家可以自行验证。

#### +initialize

+initialize的调用是不同的，如果某一个类我们没有使用过，他的+initialize方法是不会调用的，到我们使用这个类（调用了类的某个方法）的时候，才会触发+initialize方法的调用。

```
@implementation Person
+ (void)initialize {
    NSLog(@"Person - initialize");
}
@end

@implementation Person (Test1)
+ (void)initialize {
    NSLog(@"Person Test1 - initialize");
}
@end

@implementation Person (Test2)
+ (void)initialize {
    NSLog(@"Person Test2 - initialize");
}
@end
```

当我们执行`[Person alloc];`的时候，才会走+initialize方法，而且执行的Category中的+initialize：

```
2020-09-14 10:40:23.579623+0800 Category[9134:94173] Person Test2 - initialize
```
这个我们之前已经说过了，Category的方法会添加list的前面，所以会先被找到并且执行，所以我们猜测+initialize的执行是走的正常的消息机制，objc_msgSend。


由于objc_msgSend实现并没有完全开源，都是汇编代码，所以我们需要换一个思路来研究源码。

objc_msgSend本质是什么？以调用实例方法为例，其实就是通过isa指针找到该类，然后寻找方法，找到之后调用。如果没有找到则通过superClass找到父类，继续查找方法。上面的例子中，我们仅仅是调用了一个alloc方法，但是也执行了+initialize方法，所以我们猜测+initialize会在查找方法的时候调用到。通过这个思路，我们定位到了`class_getInstanceMethod()`函数（class_getInstanceMethod函数就是在类中查找某个sel时候调用的），在这个函数中，又调用了`IMP lookUpImpOrForward(id inst, SEL sel, Class cls, int behavior)`

在该函数中我们可以找到下面这段代码

```
if ((behavior & LOOKUP_INITIALIZE)  &&  !cls->isInitialized()) {
    initializeNonMetaClass (_class_getNonMetaClass(cls, inst));
}
```

可以看出如果类还没有执行+initialize 就会先执行，我们再看一下if语句中的`initializeNonMetaClass`函数，他会先拿到superClass，执行superClass的+initialize

```
supercls = cls->superclass;
if (supercls  &&  !supercls->isInitialized()) {
    initializeNonMetaClass(supercls);
}
```
这就是存在继承的情况，为什么会先执行父类的+initialize。

##### 大总结

1. 调用方式
    
    load是根据函数地址直接调用
    initialize是通过消息机制objc_msgSend调用

2. 调用时刻
    
    load是在runtime加载类和分类时调用（只会调用一次）
    initialize是在类第一次收到消息时调用个，默认没有继承的情况下每个类只会initialize一次（父类的initialize可能会被执行多次）

3. 调用顺序
    
    - load
    先调用类的load：先编译的类先调用，子类调用之前，先调用父类的
    在调用Category的load：先编译的先调用
    
    - initialize
    先初始化父类
    在初始化子类（初始化子类可能调用父类的initialize）
    
    
#### 补充

上面总结的时候说到父类的initialize会被执行多次，什么情况下会被执行多次，为什么？举个例子：

```
@implementation Person
+ (void)initialize {
    NSLog(@"Person - initialize");
}
@end

@implementation Student
@end
```
**Student类继承Person类，并且只有父类Person中实现了+initialize，Student类中并没有实现**

此时我们调用`[Student alloc];`， 会得到如下的打印。

```
2020-09-14 11:31:55.377569+0800 Category[11483:125034] Person - initialize
2020-09-14 11:31:55.377659+0800 Category[11483:125034] Person - initialize
```

Person的+initialize被执行了两次，但这两个意义是不同的，第一次执行是因为在调用子类的+initialize方法之前必须先执行父类的了+initialize，所以会打印一次。当要执行子类的+initialize时，通过消息机制，student类中并没有找到实现的+initialize的实现，所以要通过superClass指针去到父类中继续查找，因为父类中实现了+initialize，所以才会有了第二次的打印。

#### 结尾

本文的篇幅略长，笔者按照自己的思路和想法写完了此文，陈述过程不一定那么调理和完善，大家在阅读过程中发现问题，可以留言交流。

感谢阅读。


    





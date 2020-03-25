---
title: MJExtension源码学习(二)
date: 2018-06-04 17:10:09
tags: [iOS,MJExtension]
---
接上篇[MJExtension源码学习（一）](https://www.jianshu.com/p/b676c80c2880)
#### 总览
这一次我们来看MJExtension最新版本的代码，当前最新为3.0.15

在看源码之前，注意MJExtensionConfig这个类。因为它重写了+load方法，然后把使用的model的一些配置，统一写到了这个文件中。

我大致的看了一下代码，随时版本的更新，改变了一些方法的名字和类名，但是其本质的思路是没有变的，跟最初版本一直。然后再看代码的过程中会发现各种判断的情况，还有一些看起来不知道做什么用的代码，其实没关系，因为随着更新这套框架的功能越来越晚上，可以做的事多了，自然为了代码的健壮性就会看到很多类型判断的代码。

我们直接看到核心代码这个方法就可以了

<!-- more -->

```objc
/**
 核心代码：
 */
- (instancetype)mj_setKeyValues:(id)keyValues context:(NSManagedObjectContext *)context

```

添加了黑名单和白名单的功能，这是获取到响应名单中的成员变量（前提是有设置）

```objc
NSArray *allowedPropertyNames = [clazz mj_totalAllowedPropertyNames];
NSArray *ignoredPropertyNames = [clazz mj_totalIgnoredPropertyNames];
```

往下的`mj_enumerateProperties`遍历中，返回了一个`MJProperty`类，他的做用是对每个成员变量做一层封装，代替了最初的`MJIvar`这个类，`MJProperty`这个类的内部也更加完善了，这个地方大家可以自己看一下。

再往后就是一些判断然后复制，跟之前的差不多，大家看一下就很容易看明白的。



#### 分析封装MJProperty

这里我想说一下`MJProperty`中下面的这两个方法

```objc
/**** 同一个成员属性 - 父类和子类的行为可能不一致（originKey、propertyKeys、objectClassInArray） ****/
/** 设置最原始的key */
- (void)setOriginKey:(id)originKey forClass:(Class)c;
/** 对应着字典中的多级key（里面存放的数组，数组里面都是MJPropertyKey对象） */
- (NSArray *)propertyKeysForClass:(Class)c;
```
但从代码来看很难明确这两个方法到底是做什么用的，我用debug跟了一下，发现其实这个`propertyKeysForClass`返回的数组中储存的其实是当前成员变量的一个层级关机，我猜测是现在MJExtension可以实现多层级的映射，这样记录下来层级的关系方便后面的映射取值。


下面我们着重看一下在把成员变量封装为`MJProperty`类的对象时，`- (void)setOriginKey:(id)originKey forClass:(Class)c;`到底做了什么。

```objc
/**
 * 简单的字典 -> 模型（key替换，比如ID和id。多级映射，比如 oldName 和 name.oldName）
 */
void keyValues2object4()
{
    // 1.定义一个字典
    NSDictionary *dict = @{
                           @"id" : @"20",
                           @"desciption" : @"好孩子",
                           @"name" : @{
                                   @"newName" : @"lufy",
                                   @"oldName" : @"kitty",
                                   @"info" : @[
                                           @"test-data",
                                           @{@"nameChangedTime" : @"2013-08-07"}
                                           ]
                                   },
                           @"other" : @{
                                   @"bag" : @{
                                           @"name" : @"小书包",
                                           @"price" : @100.7
                                           }
                                   }
                           };
    
    // 2.将字典转为MJStudent模型
    MJStudent *stu = [MJStudent mj_objectWithKeyValues:dict];
```
我们使用这个例子，MJStudent这个类中有下列的成员变量

```objc
@property (copy, nonatomic) NSString *ID;
@property (copy, nonatomic) NSString *otherName;
@property (copy, nonatomic) NSString *nowName;
@property (copy, nonatomic) NSString *oldName;
@property (copy, nonatomic) NSString *nameChangedTime;
@property (copy, nonatomic) NSString *desc;
@property (strong, nonatomic) MJBag *bag;
@property (strong, nonatomic) NSArray *books;
```
而且一开始就已经说过了，在MJExtensionConfig中已经实现下面的映射关系

```objc
#pragma mark MJStudent中的desc属性对应着字典中的desciption
#pragma mark ....
    [MJStudent mj_setupReplacedKeyFromPropertyName:^NSDictionary *{
        return @{
                 @"desc" : @"desciption",
                 @"oldName" : @"name.oldName",
                 @"nowName" : @"name.newName",
                 @"otherName" : @[@"otherName", @"name.newName", @"name.oldName"],
                 @"nameChangedTime" : @"name.info[1].nameChangedTime",
                 @"bag" : @"other.bag"
                 };
    }];
    // 相当于在MJStudent.m中实现了+(NSDictionary *)mj_replacedKeyFromPropertyName方法
    
```
我们就找一个位置比较深的字段来debug，然后看看她的
`[property setOriginKey:[self propertyKey:property.name] forClass:self];`到底为他储存了什么数据

我们把断点先打在block的回调中

```objc
[clazz mj_enumerateProperties:^(MJProperty *property, BOOL *stop) {
}
```
当遍历到`otherName`这个成员变量对应的property时，我截下了property中的所有数据，如下图
![property](https://upload-images.jianshu.io/upload_images/1491333-62dadee9e0498c5c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很容易看出来上面的set方法为_propertyKeysDict存进去一个键值对，key为model的类名，value是一个数组，这是我们只能看到这个数组中又有三个数组，第一个数组有一个元素，第二个两个元素，第三个两个元素。

单从这里看不出这这个_propertyKeysDict种到底存的是什么，如果大家debug跟一下整个set的过程就会看到，整个value大数组有三个元素，是因为我们在映射处理的代码中`@"otherName" : @[@"otherName", @"name.newName", @"name.oldName"],`otherName对应了一个三个元素的数组，然后从每个元素中看，因为`name.newName`和`name.oldName`都是做了一级的映射，所以他们下面的数组会有两个元素，`name.newName`对应的数组中的两个元素分别name对应的MJPropertyKey和newName对应的MJPropertyKey。

其实到现在，我们已经了解了MJProperty中的成员变量都是为了储存社会，但是就拿这个_propertyKeysDict来说，我们还没有很清楚的明白他为什么要储存一个这样的层级结构，所以我们得继续往下看后面的取值和赋值。

这个地方的就很简单了，其实还是为了多级映射的取值

```objc
 // 1.取出属性值
id value;
NSArray *propertyKeyses = [property propertyKeysForClass:clazz];
for (NSArray *propertyKeys in propertyKeyses) {
    value = keyValues;
    for (MJPropertyKey *propertyKey in propertyKeys) {
        value = [propertyKey valueInObject:value];
    }
    if (value) break;
}
```
其中嵌套的for循环，就可以一层层的找到最深的那个映射，就拿刚才的name.newName来说，以为封装了两个MJPropertyKey，在遍历第一次的时候，拿到了name对应的数据，然后在遍历第二次拿到newName的数据，可以看出来，每一次的**外层的遍历**过程中，如果如果取到value值，遍历就结束了， 所以咱们用的例子最终打印的结果`otherName=lufy,`，也就是name.newName的值。

个人感觉这种写成对应数组的映射，可能是处理有些映射取不到值，哪个有值用哪个 ~ 如果是单纯的多级映射，其实`@"oldName" : @"name.oldName",`这种写法就是ok的。

项目中还有很多其他的细节就不单说了。















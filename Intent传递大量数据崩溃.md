---
title: Intent传递大量数据崩溃
date: 2017-12-12 11:16:14
tags: Android
---

#### 背景:
上一级的列表页, 需要选中几条或者多条或者全部列表, 然后将数据传递到下一级页面. 

    intent.putExtra("selectList", selectDataArr);
    


![bean.jpg](http://upload-images.jianshu.io/upload_images/1491333-d1dd9976d1f00c2e.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!-- more -->

这是创建的Bean文件, 全选传值时要传递的就是dataList里面的内容, 那么问题来了 :

在声明dataList的ArrayList的泛型时, 这个小的BackListItemBean如果是直接嵌套在大的class BacklogListEntity内部, 当时测试数据有180条, 则直接崩溃, 报一个intent传递内容太大的错误. 如果把这个小的 BackListItemBean 单独拿出来自己新建一个java文件, 则180条测试数据可以传递, 可以传递900条左右, 如果数据在数, 也会超过intent的限制.
    
    

问题1 : 小的BackListItemBean是嵌套或者不嵌套, 为什么会占内存不同? 
问题2 : intent传递较大量级的数据, 用什么方式?


回答1:(个人见解,如错请指正,谢谢)
    
如果是嵌套的小bean, 要想取到小的bean就要使用BackListItemBean.BacklogListEntity.BackListItemBean 这样等于嵌套两层, 这样即使只是用小的bean, 也会多存储上两层的地址信息, 单独的文件就不需要这两层的地址信息,所以可以传递的条数会多一些.
    
回答2:

方法1,使用静态类,传递时把数据存在静态类里面,但是使用后要注意及时置为null, 不过里面会牵扯一些java内存机制的问题.

方法2, 写数据库, 但是存取耗时, 不推荐

方法3, 只记住选中的item的数据的id, 拼成字符串传到下一级页面,然后再鞋以及页面再请求一次获取数据的接口,然后根据传进来的id比对出师选中的那些数据, 当然也有缺陷,如果选的数据多,循环次数也很多
    

-----

毕竟刚接触安卓不久, 很多底层原理都还不大懂, 解决问题的方法也可能不够好, 如果有更好的方式, 希望不累赐教.



-----
##### 网友建议：
在Android建議使用Parcelable而不是Serializable，有效能上的差距
另外，如果是使用Android內置的SQLife的話方法2、3才會效率不好
但目前多數人都不使用內置SQLife，而是使用第三方Library，例如：Realm、ObjectBox
這些的存取效率都非常好，所以大量數據傳遞時，應使用方法2、3



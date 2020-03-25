---
title: SQLite学习和业务封装
date: 2018-04-30 10:08:29
tags: [iOS,SQLite]
---


看了一段时间的MySQL, 正好借此机会把常用于嵌入式和移动设备的SQLite复习了一下. 我结合了公司的项目 以实际的业务需求为导向, 对sqlite3进行了简单的封装, 实现对项目中搜索记录的管理. 

#### 需求分析

1. 需要实现的功能

    搜索记录的保存, 删除某一条记录, 删除所有记录, 数据所占内存的大小

2. 应用的场景

    在整个项目中, 可能会存在对个搜索页面, 而且搜索的内容也不同, 就拿我们项目来说, 有项目列表页的搜索, 资讯搜索, 这些不同类型的搜索是要分开来处理的.

    
#### 准备工作
在准备的过程中遇到的第一个问题就是, 项目中用来存储历史记录的数据库和对应的表, 是要动态调用sql语句创建还是直接在本地创建好, 然后把文件导入工程中. 

如果是动态创建, 那么在项目的整体逻辑中要经常判断是否已经存在了我们所需要的库和表, 个人觉得很烦, 而且一个db文件占内存也就才几K, 所以我就选择了直接创建然后导入工程的方式.

我用的sqlite图形化工具是 **DB Browser for SQLite**, 操作起来很简单, 库和表分分钟搞定. 

![新建的表.png](https://upload-images.jianshu.io/upload_images/1491333-ef7c3e5ffcd3044f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

表名search_history
**id**就不用提了
**search_type**储存的是搜索的使用场景, 一个场景对应一个type. 举个例子, 比如项目搜索相关的都是type1, 资讯搜索的是type2, 这样就把场景区分开了. 
**search_content**就是保存起来的搜索关键字. 
**search_time**是用来记录数据存储的时间(直接使用时间戳), 加这个time字段就是排序用的, 因为我们在展示搜索记录是, 默认的逻辑是最新搜索的在最前面, 而且如果一个你曾经搜索过的记录, 如果重新搜索了一次, 那也要更新这个记录对应的时间. 

<!-- more -->

#### 封装类的方法

![代码文件.png](https://upload-images.jianshu.io/upload_images/1491333-84bd67c64c95f4c5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实就是这两个类, 先忽略那个Error文件吧, 我感觉自己处理error做的不太好, 里面也就没写啥, 只有两个字段. 重点是**SearchDBHelper**这个类.

下面是.h文件中的代码, 我只抛出了需要的这几个方法


    #import <Foundation/Foundation.h>
    @class BaseSQLError;
    
    NS_ASSUME_NONNULL_BEGIN
    
    // 历史记录类型
    // 根据项目中的实际使用场景来定义,每个场景定义一个type.
    // 比如说项目搜索和商品搜索页面都要分别记录搜索历史, 那就创建两个类型, 一个对应项目列表,一个对应商品列表, 以此类推
    extern NSString * const SEARCH_TYPE_1;
    extern NSString * const SEARCH_TYPE_2;
    
    @interface SearchDBHelper : NSObject
    
    /**
     创建SearchDBHelper实例,设置变量的默认
     @param searchType 历史记录类型
     @return           实例
     */
    + (instancetype)initWithSearchType:(NSString *)searchType;
    
    /**
     设置最大存储数量和类型
     @param max        最大储存数量
     */
    - (void)setMax:(NSInteger)max;
    
    /**
     获取当前分类下所有的历史记录
     @return 所有历史记录
     */
    - (NSArray<NSString *> *)getAllSearchData;
     
    /**
     插入数据
     @param keyword  要要入到表中的搜索记录
     @param complete  完成的回调
     */
    - (void)insertData:(NSString *)keyword
              callBack:(void (^)(BaseSQLError * _Nullable error))complete;
    
    /**
     清除一条搜索记录
     @param keyword     要清除的搜索关键字
     @param complete    完成回调
     */
    - (void)clearSingleSearchData:(NSString *)keyword
                         callBack:(void (^)( BaseSQLError * _Nullable error))complete;
    
    /**
     删除当前类型的所有搜索记录 
     */
    - (void)clearAllSearchDataFromCurrentType:(void (^)(BaseSQLError * _Nullable error))complete;
    
    /**
     获取该数据库占内存大小
     @return 数据库大小:单位k
     */
    + (NSString *)getSizeFromDataBase;
     
    /**
     删除所有的搜索历史记录(清空记录缓存)
     */
    + (void)clearAllSearchData:(void (^)(BaseSQLError * _Nullable error))complete;
     
    NS_ASSUME_NONNULL_END
    
    @end
    

针对这个.h文件中的代码, 我说几个点

1.  方法参数的为空问题
 
    现在看到的.h 文件中的方法 其实是我修改之后的, 在最开始的设计中, 几乎每一个方法都要传进去searchType, 但是呢, 我又想要限制传的searchType不能为空, 所以就想到了限制符.

    苹果咋子Xcode6.3引入OC的新特性, **Nullability**和**Annotations**, 核心呢就是两个修饰符, __nonnull和__nullable, __nonnull表示修饰的对象不可以为NULL或nil, __nullable表示可以为空. 有了这种修饰符, 在使用方法时如果传nil或NULL, 编译器就会报警.  在 Xcode7 中，为了避免与第三方库潜在的冲突，苹果把 __nonnull / __nullable 改成 _Nonnull / _Nullable。再加上苹果同样支持了没有下划线的写法 nonnull/nullable，很乱, 但本质上是一样的, 效果也一样, 就是修饰符放的位置不太一样, 具体的使用方法就不啰嗦, 从工程里随便找个原生的类点进去看看就一目了然了. 

    下面是苹果的建议  :
    >1. 对于属性、方法返回值、方法参数的修饰，使用：nonnull/nullable；
    >2. 对于 C 函数的参数、Block 的参数、Block 返回值的修饰, 使用：_Nonnull/_Nullable，建议弃用 ~~__nonnull / __nullable~~

    但是像我这样一个.h文件, 如果加太多的nonnull的话, 首先是很繁琐, 在一个确实也不太好看, 为了解决这个问题, Foundation框架给出了这对宏, **NS_ASSUME_NONNULL_BEGIN**和 **NS_ASSUME_NONNULL_END**.  包含在这对宏里的对象默认添加nonnull修改符, 然后有极个别的可以为空的(比如block)单独添加添加可以为空的修饰符

        (void (^)(BaseSQLError * _Nullable error))complete
 
2. 定义的外部变量
    extern NSString * const SEARCH_TYPE_1;
    extern NSString * const SEARCH_TYPE_2;

    这个是为了统一定义searchType, 然后在外部使用这个helper类的时候, 要传searchType时可以直接使用定义好的SEARCH_TYPE_1此类的变量名, 不至于都是直接使用@"xxx"字符串形式, 容易出错.



#### 方法实现逻辑

其实大部分的功能看 .m 中的代码都是可以看明白的, 我这里稍微简单的说一下一些思路问题吧.  

就拿insert这个方法来说, 这个方法也是比较麻烦的一个了吧,下面是方法的实现代码

    - (void)insertData:(NSString *)keyword callBack:(void (^)(BaseSQLError *_Nullable error))complete {
        
        // 判断是否已经存储过此keyword  已存储:update 未存储:insert
        if ([self selectKeyWordIsExit:keyword] == ExitKeyWord) {
            [self updateSearchKeyword:keyword callBack:complete];
        }
        else if ([self selectKeyWordIsExit:keyword] == NotExitKeyWord){
            [self insertSearchKeyword:keyword callBack:complete];
        }
        else if ([self selectKeyWordIsExit:keyword] == ExitUnknown){
            BaseSQLError * sqlErr = [BaseSQLError new];
            sqlErr.errInfo = @"插入数据库失败";
            sqlite3_close(_sql3);
            if(complete){
                complete(sqlErr);
            }
            return;
            
        }
        
    }
    
    
可以看出来, 在插入一个数据之前 , 要先要先判断这个内容是否已经存在了, 也就是**selectKeyWordIsExit:**这个方法, 他返回的是一个枚举值(存在, 不存在, 未知), 未知就是代表在操作数据库是出错. 

如果已经存在了, 那就调用update的方法, 来更新这个search_content对应的search_time.  如果不存在, 在进行下一步的判断. 

那就是判断当前的这个search_type下储存的数据条数,是否已经到了最大值了, 这个最大值我们可以通过调用方法设置, 如果不设置默认我给的是10条. 就拿max为10来说, 如果还没有到达10条, 现在可以调用insert的方法了, 如果已经有10条了, 那就得把时间最早的那一条删掉, 然后再insert这条新的.

数据库操作无非就是增删改查, 增删改这个三类操作比较简单, 因为没有返回的数据, 但是查询呢 , 有一种数量的查询(查询某个searchTpye下已经储存的数据的条数), 这种只是放回了一个int的值.  在一种就是查询数据了(比如说当前searchType下所有的储存的历史距离), 那这一种就要返回一个数组NSArray <NSString \*>\*. 这两种查询的sql实现不太一样, 我在代码里都有标识了. 大家可以具体看看代码 .

这个类的封装逻辑就是这个样子, 具体的sqlite语句的用法, 大家可以看看代码. 我这里把整个.m代码也放出来, 大家可以这样看, 也可以去github上面下载代码看一下,  而且我在github的代码中还做了一个小demo来演示用法. 

[本项目的github地址](https://github.com/Sunxb/SQLiteHelper)




#### .m实现代码

```objc

//
//  SearchDBHelper.m
//  LearnSQLite
//
//  Created by Sunxb on 2018/4/26.
//  Copyright © 2018年 Sunxb. All rights reserved.
//

#import "SearchDBHelper.h"
#import <sqlite3.h>
#import "BaseSQLError.h"

// 表名
static NSString * const TABLE_NAME = @"search_history";

// 类型名
NSString * const SEARCH_TYPE_1 = @"type1";
NSString * const SEARCH_TYPE_2 = @"type2";


// 是否已经存在某条记录
typedef NS_ENUM(NSInteger, DBExitKeyWord) {
    ExitKeyWord = 0,
    NotExitKeyWord,
    ExitUnknown
};

@interface SearchDBHelper()
@property (nonatomic,strong) NSString * searchType;
@property (nonatomic,assign) NSInteger maxNum;
@property (nonatomic,assign) sqlite3 *sql3;
@end

@implementation SearchDBHelper

+ (instancetype)initWithSearchType:(NSString *)searchType {
    SearchDBHelper * helper = [[SearchDBHelper alloc] init];
    helper.searchType = searchType;
    return helper;
}


- (instancetype)init {
    if (self = [super init]){
        self.maxNum = 10;
        self.searchType = @"";
    }
    return self;
}


- (void)setMax:(NSInteger)max {
    self.maxNum = max;
}


- (NSArray<NSString *> *)getAllSearchData {
    return [self runSelectListSQLWithType:self.searchType];
}


- (void)insertData:(NSString *)keyword callBack:(void (^)(BaseSQLError *_Nullable error))complete {
    
    // 判断是否已经存储过此keyword  已存储:update 未存储:insert
    if ([self selectKeyWordIsExit:keyword] == ExitKeyWord) {
        [self updateSearchKeyword:keyword callBack:complete];
    }
    else if ([self selectKeyWordIsExit:keyword] == NotExitKeyWord){
        [self insertSearchKeyword:keyword callBack:complete];
    }
    else if ([self selectKeyWordIsExit:keyword] == ExitUnknown){
        BaseSQLError * sqlErr = [BaseSQLError new];
        sqlErr.errInfo = @"插入数据库失败";
        sqlite3_close(_sql3);
        if(complete){
            complete(sqlErr);
        }
        return;
        
    }
    
}


- (void)clearSingleSearchData:(NSString *)keyword callBack:(void (^)(BaseSQLError *_Nullable error))complete {
    NSString * deleteSingle = [NSString stringWithFormat:@"delete from search_history where search_content=\"%@\" and search_type=\"%@\";",keyword,self.searchType];
    [self runSQL:deleteSingle callBack:complete];
}


- (void)clearAllSearchDataFromCurrentType:(void (^)(BaseSQLError *_Nullable error))complete {
    NSString * deleteSingle = [NSString stringWithFormat:@"delete from search_history where search_type=\"%@\";",self.searchType];
    [self runSQL:deleteSingle callBack:complete];
}


+ (NSString *)getSizeFromDataBase {
    NSString * dbPath = [self getDBPath];
    NSLog(@"=== %@",dbPath);
    NSFileManager * fileManager = [NSFileManager defaultManager];
    if (dbPath.length > 0) {
        long long dbSize = [fileManager attributesOfItemAtPath:dbPath error:nil].fileSize;
        return [NSString stringWithFormat:@"%gk",dbSize/1024.0];
    }
    
    return @"0k";
}


+ (void)clearAllSearchData:(void (^)(BaseSQLError *))complete {
    SearchDBHelper * helper = [[SearchDBHelper alloc] init];
    NSString * deleteAll = [NSString stringWithFormat:@"delete from search_history"];
    [helper runSQL:deleteAll callBack:complete];
}





#pragma mark - private

// 获取数据库文件在工程中的路径
+ (NSString *)getDBPath {
    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"mydb" ofType:@"db"];
    NSFileManager * manager = [NSFileManager defaultManager];
    if ([manager fileExistsAtPath:filePath]) {
        return filePath;
    }
    return @"";
}

// 是否打开数据库
- (BOOL)openDataBase {
    NSString * dbPath = [SearchDBHelper getDBPath];
    if(dbPath.length > 0) {
        if (sqlite3_open([dbPath UTF8String], &_sql3) != SQLITE_OK) {
            sqlite3_close(_sql3);
            _sql3 = nil;
            return NO;
        }
        else {
            return YES;
        }
    }
    return NO;
}

// 运行普通无返回sql语句 : insert || update
- (void)runSQL:(NSString *)sql callBack:(void (^)(BaseSQLError *))complete {
    BaseSQLError * sqlErr = [BaseSQLError new];
    
    if (![self openDataBase]) {
        sqlErr.errInfo = @"打开数据库失败";
        if(complete){
            complete(sqlErr);
        }
        return;
    }
    
    char * err;
    int code = sqlite3_exec(_sql3, [sql UTF8String], NULL, NULL, &err);
    if (code != SQLITE_OK) {
        BaseSQLError * sqlErr = [BaseSQLError new];
        sqlErr.errInfo = [NSString stringWithCString:err encoding:NSUTF8StringEncoding];
        sqlErr.errCode = code;
        complete(sqlErr);
    }
    else {
        complete(nil);
    }
    
    sqlite3_close(_sql3);
    _sql3 = nil;
}


// TODO: 运行查询性质的sql语句 - 返回数组 (比如查询所有的数据)
- (NSArray *)runSelectListSQLWithType:(NSString *)searchType {
    if (![self openDataBase]) {
        BaseSQLError * sqlErr = [BaseSQLError new];
        sqlErr.errInfo = @"打开数据库失败";
        return nil;
    }
    NSString * sql = [NSString stringWithFormat:@"select search_content from search_history where search_type=\"%@\" order by search_time desc;",searchType];
    sqlite3_stmt * statement;
    if (sqlite3_prepare_v2(_sql3, [sql UTF8String], -1, &statement, NULL)==SQLITE_OK){
        NSMutableArray * historyArr = [NSMutableArray new];
        while (sqlite3_step(statement) == SQLITE_ROW) {
            NSString * str = [NSString stringWithUTF8String:(const char *)sqlite3_column_text(statement, 0)];
            [historyArr addObject:str];
        }
        sqlite3_finalize(statement);
        statement = nil;
        return historyArr;
    }
    
    sqlite3_close(_sql3);
    return nil;
}

/**
 运行查询性质的sql语句 -  返回数量 (比如查询符合某条件的结果数)

 @param dict     需要查询的条件(key value形式)
 @param complete 完成回调
 @return         查询的数值 (查询过程出错返回 -1)
 */
- (NSInteger)runCountSelectSQL:(NSDictionary *)dict callBack:(void (^)(BaseSQLError *))complete {

    if (![self openDataBase]) {
        BaseSQLError * sqlErr = [BaseSQLError new];
        sqlErr.errInfo = @"打开数据库失败";
        if(complete){
            complete(sqlErr);
        }
        return -1;
    }
    
    NSMutableString * searchRequire = [NSMutableString new];
    NSArray * keysArr = dict.allKeys;
    for (int i = 0; i < keysArr.count; i ++){
        NSString * key = keysArr[i];
        if (i == 0){
            [searchRequire appendFormat:@"%@=\"%@\"",key,[dict objectForKey:key]];
        }
        else {
            [searchRequire appendFormat:@"and %@=\"%@\"",key,[dict objectForKey:key]];
        }
    }
    
    
    NSString * selSql = [NSString stringWithFormat:@"select count(*) from search_history where %@;",searchRequire];
    sqlite3_stmt * statement;
    //预处理
    if(sqlite3_prepare_v2(_sql3, [selSql UTF8String], -1, &statement, NULL)==SQLITE_OK){
        sqlite3_step(statement);
        NSInteger result =  sqlite3_column_int(statement, 0);
        sqlite3_finalize(statement);
        statement = nil;
        return result;
    }
    
    return -1;
}

// 查询某记录是否已经存在
- (DBExitKeyWord)selectKeyWordIsExit:(NSString *)keyword {
    
    NSInteger count = [self runCountSelectSQL:@{@"search_content":keyword,@"search_type":self.searchType} callBack:nil];
    if(count > 0) {
        return ExitKeyWord;
    }
    else if (count == 0){
        return NotExitKeyWord;
    }
    else {
        return ExitUnknown;
    }
}

//插入数据
- (void)insertSearchKeyword:(NSString *)keyword callBack:(void (^)(BaseSQLError *))complete {
    // TODO: 是否已经超过了最大存储数量
    NSInteger alreadCount = [self runCountSelectSQL:@{@"search_type":self.searchType} callBack:complete];
    
    if(alreadCount >= 0) {
        if (alreadCount >= self.maxNum) {
            // 先删除时间最早的记录,再插入
            [self deleteEarliestSearchKeywordWithType:self.searchType callBack:^(BaseSQLError *err) {
                if (err==nil) {
                    [self sqlInsert:keyword callBack:complete];
                }
                else {
                    if(complete){
                        complete(err);
                    }
                }
            }];
        }
        else {
            // 直接插入
            [self sqlInsert:keyword callBack:complete];
        }
    }
    else {
        BaseSQLError * sqlErr = [BaseSQLError new];
        sqlErr.errInfo = @"插入数据失败";
        if(complete){
            complete(sqlErr);
        }
    }
}

- (void)sqlInsert:(NSString *)keyword callBack:(void (^)(BaseSQLError *))complete{
    NSString * keyStr = @"search_type,search_content,search_time";
    NSString * valuesStr = [NSString stringWithFormat:@"\"%@\",\"%@\",\"%@\"",self.searchType,keyword,[self getCurrentTimestamp]];
    NSString * insertSql = [NSString stringWithFormat:@"insert into search_history (%@) values (%@);",keyStr,valuesStr];
    [self runSQL:insertSql callBack:complete];
}


//更新数据
- (void)updateSearchKeyword:(NSString *)keyword callBack:(void (^)(BaseSQLError *))complete {
    NSString * updateSql = [NSString stringWithFormat:@"update search_history set search_time=\"%@\" where search_content=\"%@\";",[self getCurrentTimestamp],keyword];
    [self runSQL:updateSql callBack:complete];
}

// 删除当前分类下, 时间最早的记录
- (void)deleteEarliestSearchKeywordWithType:(NSString *)searchType callBack:(void (^)(BaseSQLError *))complete{
    NSString * deleteStr = @"delete from search_history where id in \
    (select id from search_history where search_type=\"%@\" order by search_time limit 0,1)";
    NSString * deleteSql = [NSString stringWithFormat:deleteStr,searchType];
    [self runSQL:deleteSql callBack:complete];
}


// 查询当前类型下已经存储的历史记录的条数
- (NSInteger)getHistoryCountInDB {
    return [self getHistoryCountInDBWithSearchType:self.searchType];
}
- (NSInteger)getHistoryCountInDBWithSearchType:(NSString *)type {
    NSInteger count = [self runCountSelectSQL:@{@"search_type":type} callBack:nil];
    count = (count>0?count:0);
    return count;
}


// 获取当前时间戳
- (NSString*)getCurrentTimestamp {
    NSDate * data = [NSDate dateWithTimeIntervalSinceNow:0];
    NSTimeInterval a = [data timeIntervalSince1970];
    NSString * timeString = [NSString stringWithFormat:@"%0.f", a];
    return timeString;
}


@end


```


   
特此感谢: https://my.oschina.net/u/2340880/blog/601802, 给我巨大的灵感.

  




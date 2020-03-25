---
title: YYCache 源码学习（二）：YYDiskCache
date: 2019-02-14 13:54:30
tags: [iOS,YYCache]
---



##### 整体思路

从作者的《YYCache 设计思路》一文中可以看出，作者在设计YYDiskCache之前做了充分的测试：**iPhone 6 64G 下，SQLite 写入性能比直接写文件要高，但读取性能取决于数据大小：当单条数据小于 20K 时，数据越小 SQLite 读取性能越高；单条数据大于 20K 时，直接写为文件速度会更快一些。**

YYDiskCache的磁盘缓存结合使用了文件储存和数据库储存。

个人理解：在进行磁盘缓存的时候，会判断要储存数据的大小，如果数据小于20K，则直接存入数据库（数据储存到inline_data字段，此时filename为空）。如果数据大于20K，先把数据以文件形式进行存储，然后再在数据库中储存对应的文件名（此时inline_data为NULL，filename为文件地址），具体的可以结合下文中提到的磁盘缓存的文件结构来看。

磁盘缓存的核心类是`YYKVStorage`，他主要封装了文件储存操作和SQLite数据库的操作。`YYDiskCache`是对`YYKVStorage`的封装，抛出的API和内存缓存相似，都有数据读写和修剪内存。

##### 磁盘缓存的文件结构

```objc
/*
 File:
 /path/
      /manifest.sqlite
      /manifest.sqlite-shm
      /manifest.sqlite-wal
      /data/
           /e10adc3949ba59abbe56e057f20f883e
           /e10adc3949ba59abbe56e057f20f883e
      /trash/
            /unused_file_or_folder
 
 SQL:
 create table if not exists manifest (
    key                 text,
    filename            text,
    size                integer,
    inline_data         blob,
    modification_time   integer,
    last_access_time    integer,
    extended_data       blob,
    primary key(key)
 ); 
 create index if not exists last_access_time_idx on manifest(last_access_time);
 */
```

这个结构我们不需要多说什么，只提一个小点，作者在path路径下面设计了一个/data/和一个/trash/。删除文件是一个比较耗时的操作，在删除文件的时候，先进行文件的移动，然后在一个子线程中处理要删掉的文件，提高了整体的效率。

##### 实现 LRU
磁盘缓存对缓存淘汰算法的实现就比较简单了，因为每次存储都有对应的数据库记录，而且表中设计了last_access_time这个字段，我们可以直接使用数据库的排序语句就可以找到最不常用的文件了。

##### 代码分析

1.

```objc
- (sqlite3_stmt *)_dbPrepareStmt:(NSString *)sql {
    if (![self _dbCheck] || sql.length == 0 || !_dbStmtCache) return NULL;
    sqlite3_stmt *stmt = (sqlite3_stmt *)CFDictionaryGetValue(_dbStmtCache, (__bridge const void *)(sql));
    if (!stmt) {
        int result = sqlite3_prepare_v2(_db, sql.UTF8String, -1, &stmt, NULL);
        if (result != SQLITE_OK) {
            if (_errorLogsEnabled) NSLog(@"%s line:%d sqlite stmt prepare error (%d): %s", __FUNCTION__, __LINE__, result, sqlite3_errmsg(_db));
            return NULL;
        }
        CFDictionarySetValue(_dbStmtCache, (__bridge const void *)(sql), stmt);
    } else {
        sqlite3_reset(stmt);
    }
    return stmt;
}
```
这个方法是提前生成了sql语句的句柄，可以理解成提前把sql语句编译成字节码留给后面的执行函数（当前不执行）。同时，作者使用
`_dbStmtCache`对语句进行缓存，下次使用时可以更快度的加载出来。


2.

```objc
- (BOOL)_dbClose {
    if (!_db) return YES;
    
    int  result = 0;
    BOOL retry = NO;
    BOOL stmtFinalized = NO;
    
    if (_dbStmtCache) CFRelease(_dbStmtCache);
    _dbStmtCache = NULL;
    
    do {
        retry = NO;
        result = sqlite3_close(_db);
        // 状态为busy或者lock
        if (result == SQLITE_BUSY || result == SQLITE_LOCKED) {
            if (!stmtFinalized) {
                stmtFinalized = YES;
                sqlite3_stmt *stmt;
                //sqlite3_stmt *sqlite3_next_stmt(sqlite3 *pDb, sqlite3_stmt *pStmt);
                //表示从数据库pDb中对应的pStmt语句开始一个个往下找出相应prepared语句，如果pStmt为nil，那么就从pDb的第一个prepared语句开始。
                while ((stmt = sqlite3_next_stmt(_db, nil)) != 0) {
                    //释放数据库中的prepared语句资源
                    sqlite3_finalize(stmt);
                    retry = YES;
                }
            }
        } else if (result != SQLITE_OK) {
            if (_errorLogsEnabled) {
                NSLog(@"%s line:%d sqlite close failed (%d).", __FUNCTION__, __LINE__, result);
            }
        }
    } while (retry);
    _db = NULL;
    return YES;
}
```
这个是关闭数据库的方法，`_dbStmtCache`中缓存了我们使用的句柄，所以首先要释放掉了`_dbStmtCache`。

在真正关闭数据库的代码中使用了do-while循环，因为一次访问数据库并不一定成功，数据库可能是busy或者lock的状态，所以要使用一个循环来多次访问。

如果为能关闭数据库，作者使用了`sqlite3_next_stmt`一个个的找出prepared语句，并使用`sqlite3_finalize`释放了prepared资源(防止内存泄露)。


其他的就没什么好说的了，主要就是一些sql语句的用法，这些大家看一下，碰到陌生的api谷歌一下就有了 ~ 具体的文件的操作，比较常用，看起来就容易很多。



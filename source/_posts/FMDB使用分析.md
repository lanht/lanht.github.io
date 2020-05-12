---
title: FMDB使用分析
date: 2020-05-12 13:57:41
tags:
categories:
	- iOS
---

### 一、基本介绍
+ 什么是FMDB  
-- FMDB是iOS平台的SQLite数据库框架
-- FMDB以OC的方式封装了SQLite的C语言API

+ FMDB常用类：  
-- **FMDatabase**：一个FMDatabase对象就代表一个单独的SQLite数据库用来执行SQL语句  
-- **FMResultSet**：使用FMDatabase执行查询后的结果集  
-- **FMDatabaseQueue**：用于在多线程中执行多个查询或更新，它是线程安全的  

+ FMDB的优点  
-- 使用起来更加面向对象，省去了很多麻烦、冗余的C语言代码
-- 对比苹果自带的Core Data框架，更加轻量级和灵活
-- 提供了多线程安全的数据库操作方法，有效地防止数据混乱

+ FMDB的缺点  
-- 它本身也存在一些问题，比如跨平台，因为它是用oc的语言封装的，所以
只能在ios开发的时候使用，如果想实现跨平台的操作，来降低开发的成本
和维护的成本,就需要使用比较原始的SQLite。

### 二、使用方法

+ 打开数据库  
通过指定SQLite数据库文件路径来创建FMDatabase对象
```objc
FMDatabase *db = [FMDatabase databaseWithPath:path];
if (![db open]) {
  NSLog(@"数据库打开失败！");
}
```
关于文件路径的三种情况：
> 1.具体的文件路径,或是数据库的文件不存在时,fmdb会自己创建一个。  
> 2.若文件路径为空字符串@""，会在临时目录创建一个空的数据库。  
> 并且，当(FMDatabase)数据库断开连接时，数据库文件被删除。  
> 3.若你传入的参数为NULL，则它会建立一个在内存中的数据库，数据库断开连接时，数据库文件被删除。  

+ 执行更新：  
在FMDB中,除查询以外的所有操作,都称为“更新”如：create、drop、insert、update、delete等。
FMDB数据库操作使用 executeUpdate: 方法执行更新,返回BOOL型。 

executeUpdate:用法  
> -(BOOL)executeUpdate:(NSString)sql, ...  
> -(BOOL)executeUpdateWithFormat:(NSString)format, ...  
> -(BOOL)executeUpdate:(NSString*)sql withArgumentsInArray:(NSArray *)arguments  
> 示例
> [db executeUpdate:@"UPDATE t_student SET age = ? WHERE name = ?;", @20, @"Jack"]

+ 执行查询  
查询方法
> -(FMResultSet )executeQuery: (NSString)sql, ...  
> -(FMResultSet )executeQueryWithFormat:(NSString)format, ...  
> -(FMResultSet *)executeQuery:(NSString *)sql withArgumentsInArray:(NSArray *)arguments  

示例
```objc
// 查询数据
FMResultSet *rs = [db executeQuery:@"SELECT * FROM t_student"];

// 遍历结果集
while ([rs next]) { 
  NSString *name = [rs stringForColumn:@"name"];
  int age = [rs intForColumn:@"age"];
  double score = [rs doubleForColumn:@"score"];
}
```

+ FMDatabaseQueue

注意：  
> FMDatabase这个类是线程不安全的，如果在多个线程中同时使用一个  
> FMDatabase实例，会造成数据混乱等问题  
> 为了保证线程安全，FMDB提供方便快捷的FMDatabaseQueue  
> FMDatabaseQueue的创建  
> FMDatabaseQueue *queue = [FMDatabaseQueue databaseQueueWithPath:path];

简单使用
```objc
[queue inDatabase:^(FMDatabase *db) { 
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Jack"];
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Rose"];
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Jim"];
  FMResultSet *rs = [db executeQuery:@"select * from t_student"];
  while ([rs next]) {
    // …
  }
}];
```
使用事务
```objc
[queue inTransaction:^(FMDatabase *db, BOOL *rollback) {
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Jack"];
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Rose"];
  [db executeUpdate:@"INSERT INTO t_student(name) VALUES (?)", @"Jim"];
  FMResultSet *rs = [db executeQuery:@"select * from t_student"];
  while ([rs next]) {
    // …
  }
}];
```
### 三、数据库迁移
使用FMDB与FMDBMigrationManager结合 进行数据库版本的升级  

具体可以查看下面博客
iOS的数据库升级FMDBMigrationManager:[https://www.jianshu.com/p/66bc680c4360](https://www.jianshu.com/p/66bc680c4360)  
iOS 使用FMDB与FMDBMigrationManager结合 进行数据库版本的升级:[https://blog.csdn.net/u014305730/article/details/82385126](https://blog.csdn.net/u014305730/article/details/82385126)  

### 四、数据库加密
数据库加密一般有两种方式

1、对所有数据进行加密  

2、对数据库文件加密

对比以上两种方式，第一种方式的常见做法是是将要存储的内容先加密然后存到数据库中，使用的时候将数据库解密，但是这样会消耗很多时间，大部分性能消耗在数据的加解密上，同时，第二种方式，SQLite本身支持加密功能(免费版的不支持) ，SQLCipher是一个开源的SQLite加密扩展，支持对db文件进行256位的AES加密，通常我们会用FMDB这个工具库，FMDB对原生的SQLite进行了封装，提供了面向对象的方式对数据库操作，同时FMDB 也提供了对 SQLCipher 的支持。（在github上集成fmdb上可以看到pod ’fmdb/sqlCipher‘）  

FMDB数据库加密:[https://www.jianshu.com/p/0c38bf93dcc0](https://www.jianshu.com/p/0c38bf93dcc0)

### 五、示例代码
```objc
- (BOOL)openDB {
    NSString *path = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject stringByAppendingPathComponent:@"database.sqlite"];
    FMDatabase *db = [FMDatabase databaseWithPath:path];
    _db = db;
    if ([db open]) {
        BOOL result = [db executeUpdate:@"CREATE TABLE IF NOT EXISTS t_student (id integer PRIMARY KEY AUTOINCREMENT, name text NOT NULL, age integer NOT NULL);"];
        if (result) {
            NSLog(@"表创建成功");
            [db close];
            return YES;
        } else {
            NSLog(@"表创建失败");
             [db close];
            return NO;
        }
    } else {
        NSLog(@"打开失败");
         [db close];
        return NO;
    }
}

- (void)add {
    if ([self.db open]) {
        [self.db executeUpdate:@"INSERT INTO t_student (name, age) VALUES (?,?);", @"tom", @(arc4random_uniform(40))];
        //    [self.db executeUpdate:@"INSERT INTO t_student (name, age) VALUES (?,?);" withArgumentsInArray:@[@"tom",@(arc4random_uniform(40))]];
        //    [self.db executeUpdateWithFormat:@"INSERT INTO t_student (name, age) VALUES (%@,%d);", @"tom", @(arc4random_uniform(40))];
        [self.db close];
    }
}

- (void)deleteStudent:(NSInteger)ID {
    if ([self.db open]) {
        [self.db executeUpdate:@"delete from t_student where id = ?;", @(ID)];
        [self.db close];
    }
}

- (void)update:(Student *)s {
    if ([self.db open]) {
        s.age = s.age - 1;
        [self.db executeUpdate:@"update t_student set age = ? where id = ?;",s.age, s.ID];
        [self.db close];
    }
}

- (void)query {
    if ([self.db open]) {
        FMResultSet *resultSet = [self.db executeQuery:@"SELECT * FROM t_student;"];
        NSMutableArray *tmp = [NSMutableArray array];
        while ([resultSet next]) {
            int ID = [resultSet intForColumn:@"id"];
            NSString *name = [resultSet stringForColumn:@"name"];
            int age = [resultSet intForColumn:@"age"];
            
            Student *s = [[Student alloc]init];
            s.ID = ID;
            s.name = name;
            s.age = age;
            
            [tmp addObject:s];
        }
        self.students = tmp;
        
        [self.db close];
    }
}

```

### 参考链接
iOS开发数据库篇—FMDB简单介绍:[https://www.cnblogs.com/wendingding/p/3871848.html](https://www.cnblogs.com/wendingding/p/3871848.html)
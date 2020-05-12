---
title: CoreData使用分析
date: 2020-05-12 13:55:59
tags:
categories:
	- iOS
---

### 一、基本介绍
+ 什么是CoreData  
CoreData是Apple官方为iOS提供的一个数据持久化方案，其本质是一个通过封装底层数据操作，让程序员以面向对象的方式存储和管理数据的ORM框架（Object-Relational Mapping：对象-关系映射，简称ORM）。虽然底层支持SQLite、二进制数据、xml等多种文件存储，但是主要还是用来操作SQLite数据库。    
程序员不需要学习或者使用SQL语句，只需要使用CoreData框架提供的对象和接口以及图形化工具，即可完成SQLite数据库的创建、表关系、增删改查等一系列操作，在一定程度上降低了程序员的学习成本并增加了代码的统一性和可阅读性。

+ CoreData主要包含以下几个类  
 -- **NSManagedObjectModel**：托管对象模型，映射实体类和数据库数据的关系，本质是一个XML文件，后面简称MOM  
 -- **NSManagedObject**：托管对象，对应数据库数据的实体，后面简称MO  
 -- **NSManagedObjectContext**：托管对象上下文，管理托管对象，后面简称MOC  
 -- **NSPersistentStoreCoordinator**：持久化存储调度器，用来处理磁盘持久化数据和实体类对象的相互转化，后面简称PSC  
 -- **NSPersistentStore**：持久化存储器，负责磁盘持久化数据存取，后面简称PS  
 -- **NSEntityDescription**：用来描述实体
 -- **NSFetchRequest**：操作请求

+ CoreData的总体框架如下图  
![](images/db/db1.jpg)  
在上层通过MOC操作对应的托管对象，然后MOC会将操作传递给PSC，PSC通过托管对象模型中的映射关系，再将托管对象的操作转化为对底层数据的操作，进行数据存取操作。

### 二、简单创建流程
1. 模型文件操作     
   1.1 创建模型文件，后缀为`.xcdatamodeld`。创建模型文件之后，可以在内部进行添加实体等操作（用于表示数据库文件的数据结构）。  
   1.2 添加实体（表示数据库文件的表结构），添加实体后需要通过实体来创建托管对象类文件。  
   1.3 天剑属性并设置类型，可以在属性的右侧面板中设置默认值等选项。（每种数据类型设置的选项是不同的）。  
   1.4 创建请求模板，设置配置模板等。 
   1.5 根据指定实体，创建托管对象类文件（基于`NSManagedObject`）.  

2. 实例化上下文对象  
   2.1 创建托管上下文（`NSManagedObjectContext`）。  
   2.2 创建托管对象模型（`NSManagedObjectModel`）。  
   2.3 根据托管对象模型，创建持久化存储协调器（`NSPersistentStoreCoordinator`）。  
   2.4 关联并创建本地数据库文件，并返回持久化存储持对象（`NSPersistentStore`）。
   2.5 将持久化存储协调器赋值给托管对象上下文，完成基本创建。

3. 示例  
下面是根据Company模型文件，创建了一个主队列并发类型的MOC  
```objc
// 创建上下文对象，并发队列设置为主队列
NSManagedObjectContext *context = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];

// 创建托管对象模型，并使用Company.momd路径当做初始化参数
NSURL *modelPath = [[NSBundle mainBundle] URLForResource:@"Company" withExtension:@"momd"];
NSManagedObjectModel *model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelPath];

// 创建持久化存储调度器
NSPersistentStoreCoordinator *coordinator = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:model];

// 创建并关联SQLite数据库文件，如果已经存在则不会重复创建
NSString *dataPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
dataPath = [dataPath stringByAppendingFormat:@"/%@.sqlite", @"Company"];
[coordinator addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:[NSURL fileURLWithPath:dataPath] options:nil error:nil];

// 上下文对象设置属性为持久化存储器
context.persistentStoreCoordinator = coordinator;
```

### 三、基本操作
1. 插入操作
```objc
// 创建托管对象，并指明创建的托管对象所属实体名
Employee *emp = [NSEntityDescription insertNewObjectForEntityForName:@"Employee" inManagedObjectContext:context];
emp.name = @"lxz";
emp.height = @1.7;
emp.brithday = [NSDate date];

// 通过上下文保存对象，并在保存前判断是否有更改
NSError *error = nil;
if (context.hasChanges) {
    [context save:&error];
}

// 错误处理
if (error) {
    NSLog(@"CoreData Insert Data Error : %@", error);
}   
```

2. 删除操作
```objc
// 建立获取数据的请求对象，指明对Employee实体进行删除操作
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Employee"];

// 创建谓词对象，过滤出符合要求的对象，也就是要删除的对象
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name = %@", @"lxz"];
request.predicate = predicate;

// 执行获取操作，找到要删除的对象
NSError *error = nil;
NSArray<Employee *> *employees = [context executeFetchRequest:request error:&error];

// 遍历符合删除要求的对象数组，执行删除操作
[employees enumerateObjectsUsingBlock:^(Employee * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    [context deleteObject:obj];
}];

// 保存上下文
if (context.hasChanges) {
    [context save:nil];
}

// 错误处理
if (error) {
    NSLog(@"CoreData Delete Data Error : %@", error);
}
```

3. 修改操作
```objc
// 建立获取数据的请求对象，并指明操作的实体为Employee
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Employee"];

// 创建谓词对象，设置过滤条件
NSPredicate *predicate = [NSPredicate predicateWithFormat:@"name = %@", @"lxz"];
request.predicate = predicate;

// 执行获取请求，获取到符合要求的托管对象
NSError *error = nil;
NSArray<Employee *> *employees = [context executeFetchRequest:request error:&error];
[employees enumerateObjectsUsingBlock:^(Employee * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    obj.height = @3.f;
}];

// 将上面的修改进行存储
if (context.hasChanges) {
    [context save:nil];
}

// 错误处理
if (error) {
    NSLog(@"CoreData Update Data Error : %@", error);
}
```

4. 查找操作
```objc
// 建立获取数据的请求对象，指明操作的实体为Employee
NSFetchRequest *request = [NSFetchRequest fetchRequestWithEntityName:@"Employee"];

// 执行获取操作，获取所有Employee托管对象
NSError *error = nil;
NSArray<Employee *> *employees = [context executeFetchRequest:request error:&error];
[employees enumerateObjectsUsingBlock:^(Employee * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
    NSLog(@"Employee Name : %@, Height : %@, Brithday : %@", obj.name, obj.height, obj.brithday);
}];

// 错误处理
if (error) {
    NSLog(@"CoreData Ergodic Data Error : %@", error);
}
```
查找操作最简单粗暴，因为是演示代码，所以直接将所有Employee表中的托管对象加载出来。在实际开发中肯定不会这样做，只需要加载需要的数据。后面还会讲到一些更高级的操作，会涉及到获取方面的东西。

### 四、版本迁移
几种版本迁移方案，在迁移之前都需要对原有的模型文件创建新的版本

**选中需要做迁移的模型文件 -> 点击菜单栏Editor -> Add Model Versioon -> 选择基于哪个版本的的模型文件（一般都是选择目前最新的版本），新建模型文件完成。**

对于新版本的模型文件的命名，一般会拿当前工程版本号当后缀，这样在模型文件比较多时，可以很容易将模型文件版本和工程版本对应起来。

1. 轻量级迁移 （修改了model文件，不过不做迁移，在使用数据操作就会闪退，除非删掉沙盒里的sqlite）

2. Mapping Model迁移方案

3. 渐进式迁移

4. 更复杂的迁移需求


### 五、多线程
在业务比较复杂的情况下，需要进行大量数据处理，并且还需要涉及到UI的操作。对于这种复杂需求，如果都放在主队列中，对性能和界面流畅度都会有很大的影响，导致用户体验非常差，降低屏幕FPS。对于这种情况，可以采取多个MOC配合的方式。

**Core Data多线程操作的基本原则**
不允许跨线程访问MOC： 在某一个MOC上的CRUD操作只能在它的操作线程上进行
不允许跨线程访问MO：对MO的操作只能在它所属的MOC的操作线程上进行。需要注意的是，访问一个FRC的fetchedObjects数组也只能在FRC所属的MOC的操作线程上进行

**iOS5之前使用多个MOC**

在iOS5之前实现MOC的多线程，可以创建多个MOC，多个MOC使用同一个PSC，并让多个MOC实现数据同步。通过这种方式不用担心PSC在调用过程中的线程问题，MOC在使用PSC进行save操作时，会对PSC进行加锁，等当前加锁的MOC执行完操作之后，其他MOC才能继续执行操作。

每一个PSC都对应着一个持久化存储区，PSC知道存储区中数据存储的数据结构，而MOC需要使用这个PSC进行save操作的实现。
这样做有一个问题，当一个MOC发生改变并持久化到本地时，系统并不会将其他MOC缓存在内存中的NSManagedObject对象改变。所以这就需要我们在MOC发生改变时，将其他MOC数据更新。

根据上面的解释，在下面例子中创建了一个主队列的mainMOC，主要用于UI操作。一个私有队列的backgroundMOC，用于除UI之外的耗时操作，两个MOC使用的同一个PSC。

```objc
// 获取PSC实例对象
- (NSPersistentStoreCoordinator *)persistentStoreCoordinator {

    // 创建托管对象模型，并指明加载Company模型文件
    NSURL *modelPath = [[NSBundle mainBundle] URLForResource:@"Company" withExtension:@"momd"];
    NSManagedObjectModel *model = [[NSManagedObjectModel alloc] initWithContentsOfURL:modelPath];

    // 创建PSC对象，并将托管对象模型当做参数传入，其他MOC都是用这一个PSC。
    NSPersistentStoreCoordinator *PSC = [[NSPersistentStoreCoordinator alloc] initWithManagedObjectModel:model];

    // 根据指定的路径，创建并关联本地数据库
    NSString *dataPath = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES).lastObject;
    dataPath = [dataPath stringByAppendingFormat:@"/%@.sqlite", @"Company"];
    [PSC addPersistentStoreWithType:NSSQLiteStoreType configuration:nil URL:[NSURL fileURLWithPath:dataPath] options:nil error:nil];

    return PSC;
}

// 初始化用于本地存储的所有MOC
- (void)createManagedObjectContext {

    // 创建PSC实例对象，其他MOC都用这一个PSC。
    NSPersistentStoreCoordinator *PSC = self.persistentStoreCoordinator;

    // 创建主队列MOC，用于执行UI操作
    NSManagedObjectContext *mainMOC = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    mainMOC.persistentStoreCoordinator = PSC;

    // 创建私有队列MOC，用于执行其他耗时操作
    NSManagedObjectContext *backgroundMOC = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    backgroundMOC.persistentStoreCoordinator = PSC;

    // 通过监听NSManagedObjectContextDidSaveNotification通知，来获取所有MOC的改变消息
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(contextChanged:) name:NSManagedObjectContextDidSaveNotification object:nil];
}

// MOC改变后的通知回调
- (void)contextChanged:(NSNotification *)noti {
    NSManagedObjectContext *MOC = noti.object;
    // 这里需要做判断操作，判断当前改变的MOC是否我们将要做同步的MOC，如果就是当前MOC自己做的改变，那就不需要再同步自己了。
    // 由于项目中可能存在多个PSC，所以下面还需要判断PSC是否当前操作的PSC，如果不是当前PSC则不需要同步，不要去同步其他本地存储的数据。
    [MOC performBlock:^{
        // 直接调用系统提供的同步API，系统内部会完成同步的实现细节。
        [MOC mergeChangesFromContextDidSaveNotification:noti];
    }];
}
```
在上面的Demo中，创建了一个PSC，并将其他MOC都关联到这个PSC上，这样所有的MOC执行本地持久化相关的操作时，都是通过同一个PSC进行操作的。并在下面添加了一个通知，这个通知是监听所有MOC执行save操作后的通知，并在通知的回调方法中进行数据的合并。

**iOS5之后使用多个MOC**

在iOS5之后，MOC可以设置parentContext，一个parentContext可以拥有多个ChildContext。在ChildContext执行save操作后，会将操作push到parentContext，由parentContext去完成真正的save操作，而ChildContext所有的改变都会被parentContext所知晓，这解决了之前MOC手动同步数据的问题。

需要注意的是，在ChildContext调用save方法之后，此时并没有将数据写入存储区，还需要调用parentContext的save方法。因为ChildContext并不拥有PSC，ChildContext也不需要设置PSC，所以需要parentContext调用PSC来执行真正的save操作。也就是只有拥有PSC的MOC执行save操作后，才是真正的执行了写入存储区的操作。
```objc
- (void)createManagedObjectContext {
    // 创建PSC实例对象，还是用上面Demo的实例化代码
    NSPersistentStoreCoordinator *PSC = self.persistentStoreCoordinator;

    // 创建主队列MOC，用于执行UI操作
    NSManagedObjectContext *mainMOC = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSMainQueueConcurrencyType];
    mainMOC.persistentStoreCoordinator = PSC;

    // 创建私有队列MOC，用于执行其他耗时操作，backgroundMOC并不需要设置PSC
    NSManagedObjectContext *backgroundMOC = [[NSManagedObjectContext alloc] initWithConcurrencyType:NSPrivateQueueConcurrencyType];
    backgroundMOC.parentContext = mainMOC;

    // 私有队列的MOC和主队列的MOC，在执行save操作时，都应该调用performBlock:方法，在自己的队列中执行save操作。
    // 私有队列的MOC执行完自己的save操作后，还调用了主队列MOC的save方法，来完成真正的持久化操作，否则不能持久化到本地
    [backgroundMOC performBlock:^{
        [backgroundMOC save:nil];
    
        [mainMOC performBlock:^{
            [mainMOC save:nil];
        }];
    }];
}
```
上面例子中创建一个主队列的mainMOC，来完成UI相关的操作。创建私有队列的backgroundMOC，处理复杂逻辑以及数据处理操作，在实际开发中可以根据需求创建多个backgroundMOC。需要注意的是，在backgroundMOC执行完save方法后，又在mainMOC中执行了一次save方法，这步是很重要的。

### 多个MOC面临的多线程问题
**1.线程安全**
无论是MOC还是托管对象，都不应该在其他MOC的线程中执行操作，这两个API都不是线程安全的。但MOC可以在其他MOC线程中调用performBlock:方法，切换到自己的线程执行操作。

如果其他MOC想要拿到托管对象，并在自己的队列中使用托管对象，这是不允许的，托管对象是不能直接传递到其他MOC的线程的。但是可以通过获取NSManagedObject的NSManagedObjectID对象，在其他MOC中通过NSManagedObjectID对象，从持久化存储区中获取NSManagedObject对象，这样就是允许的。NSManagedObjectID是线程安全，并且可以跨线程使用的。

可以通过MOC获取NSManagedObjectID对应的NSManagedObject对象，例如下面几个MOC的API。

```objc
NSManagedObject *object = [context objectRegisteredForID:objectID];
NSManagedObject *object = [context objectWithID:objectID];
```

通过NSManagedObject对象的objectID属性，获取NSManagedObjectID类型的objectID对象。
```objc
NSManagedObjectID *objectID = object.objectID;
```

**2.MOC同步时机**
设置MOC的parentContext属性之后，parent对于child的改变是知道的，但是child对于parent的改变是不知道的。苹果这样设计，应该是为了更好的数据同步。
```objc
Employee *emp = [NSEntityDescription insertNewObjectForEntityForName:@"Employee" inManagedObjectContext:backgroundMOC];
emp.name = @"lxz";
emp.brithday = [NSDate date];
emp.height = @1.7f;

[backgroundMOC performBlock:^{
    [backgroundMOC save:nil];
    [mainMOC performBlock:^{
        [mainMOC save:nil];
    }];
}];
```
在上面这段代码中，mainMOC是backgroundMOC的parentContext。在backgroundMOC执行save方法前，backgroundMOC和mainMOC都不能获取到Employee的数据，在backgroundMOC执行完save方法后，自身上下文发生改变的同时，也将改变push到mainMOC中，mainMOC也具有了Employee对象。

所以在backgroundMOC的save方法执行时，是对内存中的上下文做了改变，当拥有PSC的mainMOC执行save方法后，是对本地存储区做了改变。

### 参看链接
Core Data 概述:(相关CoreData系列文章)[https://objccn.io/issue-4-1/](https://objccn.io/issue-4-1/)  
认识CoreData - 初识CoreData(写了CoreData系列文章，都可以看看)：[https://www.jianshu.com/p/c0e12a897971](https://www.jianshu.com/p/c0e12a897971)

Core Data三层设计方案：[https://github.com/ChenYongJunM/CoreDataMultithread](https://github.com/ChenYongJunM/CoreDataMultithread)
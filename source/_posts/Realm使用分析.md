---
title: Realm使用分析
date: 2020-05-12 13:59:36
tags:
categories:
 	- iOS
---

+ [Realm介绍](#Realm)
+ [Realm数据库](#Realm数据库)
  - [打开数据库](#打开数据库)
  - [配置 Realm 数据库](#1配置-realm-数据库)
  - [默认Realm数据库](#2默认Realm数据库)
  - [内存中realm数据库](#3内存中-realm-数据库)
  - [错误处理](#4错误处理)
  - [预植 Realm 数据库](#5预植-Realm-数据库)
+ [常用类](#常用类)
+ [数据模型](#数据模型)
  - [支持的数据类型](#1支持的数据类型)
  - [必要属性](#2必要属性)
  - [主键](#3主键)
  - [索引属性](#4索引属性)
  - [被忽略属性](#5被忽略属性)
  - [默认属性值](#6默认属性值)
  - [属性特性](#7属性特性)
  - [属性备忘单](#8属性备忘单)
+ [模型继承](#模型继承)
+ [关系](#关系)
  - [多对一关系](#1多对一关系)
  - [多对多关系](#2多对多关系)
  - [双向关系](#3双向关系)
+ [数据库操作](#数据库操作-增删改查)
  - [增](#一增)
  - [改](#二改)
  - [删](#三删)
  - [查](#四查)
+ [数据迁移](#数据迁移)
  - [本地迁移](#一本地迁移)
  - [同步迁移](#二迁移同步比较复杂不常用)
---

# Realm 
[官方文档](https://realm.io/cn/docs/objc/latest/#using-the-realm-framework)  
Realm 数据库是 Realm 移动端数据库容器的一个实例。Realm 数据库可以是本地化的，也可以是可同步的。

RLM_ARRAY_TYPE 宏创建了一个协议，从而允许您使用 RLMArray<Dog> 这种语法。如果这条宏没有放置在模型接口定义的底部，那么这个模型类就必须前置声明。

注意，目前暂时不支持对包含原始类型的 RLMArray 进行查询。

对象的所有更改（添加、修改和删除）都必须在写入事务内完成。

RLMArray 只能够包含 RLMObject 类型，诸如 NSString 之类的基础类型是无法包含在内的。

与传统数据库相比，Realm 查询引擎的一个独特特性就是：它能够用很小的事务开销来实现链式查询，而不是每条查询都要接二连三地分别去单独访问数据库服务器。  

您可以订阅 Realm 通知，以了解 Realm 数据何时发生了更新，比如说可以决定应用 UI 何时进行刷新，而无需重新检索 RLMResults。

---

## Realm数据库
### 打开数据库
要打开一个 Realm 数据库，首先需要初始化一个新的 `RLMRealm` 对象：
```objc
RLMRealm *realm = [RLMRealm defaultRealm];

[realm transactionWithBlock:^{
    [realm addObject:mydog];
}];
```

#### 1.配置 Realm 数据库
在打开 `Realm` 数据库之前，可以对其进行配置。通过创建一个 `RLMRealmConfiguration` 的对象实例，然后配置相应的属性。通过创建并自定义相关的配置值，使得您可以实现个性化的设置，包括如下方面：
+ 对于本地 `Realm` 数据库而言，可以配置 `Realm` 文件在磁盘上的路径；
+ 对于可同步 `Realm` 数据库而言，可以配置管理该 `Realm` 数据库的用户，以及 `Realm` 数据库在 `Realm` 对象服务器上的远程路径；
+ 对于架构版本之间发生变化的 `Realm` 数据库而言，可以通过迁移功能来控制旧架构的 `Realm` 数据该如何更新到最新的架构。
+ 对于存储的数据量过大、或者数据频繁发生变化的 Realm 数据库而言，可以通过压缩功能来控制 Realm 文件该如何实现压缩，从而确保能高效地利用磁盘空间。  
要应用配置，可以在每次需要获取 Realm 实例的时候，通过向` +[RLMRealm realmWithConfiguration:config error:&err]` 方法传递该配置对象，或者通过 `[RLMRealmConfiguration setDefaultConfiguration:config]` 方法，将默认 Realm 数据库的默认配置设置为我们所需的配置。

例如，假设有一个应用要求用户必须要登录到 Web 后端服务器中，并且需要支持账户快速切换功能的话。 那么您可以通过以下代码，来为每个账户提供一个独立的 Realm 数据库，并且当前账户所使用的数据库将作为默认 Realm 数据库来使用：
```objc
@implementation SomeClass
+ (void)setDefaultRealmForUser:(NSString *)username {
    RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];

    // 使用默认的目录，但是请将文件名替换为用户名
    config.fileURL = [[[config.fileURL URLByDeletingLastPathComponent]
                            URLByAppendingPathComponent:username]
                            URLByAppendingPathExtension:@"realm"];

    // 将该配置设置为默认 Realm 配置
    [RLMRealmConfiguration setDefaultConfiguration:config];
}
@end
```
您可以创建多个配置对象，这样便可以单独控制各个 Realm 数据库的版本、架构以及位置。
```objc
RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];

// 获取预植数据库文件的 URL
config.fileURL = [[NSBundle mainBundle] URLForResource:@"MyBundledData" withExtension:@"realm"];
// 以只读模式打开该文件，这是因为应用的预植数据库是不可写的
config.readOnly = YES;

// 使用该配置来打开 Realm 数据库
RLMRealm *realm = [RLMRealm realmWithConfiguration:config error:nil];

// 从预植 Realm 数据库中读取某些数据
RLMResults<Dog *> *dogs = [Dog objectsInRealm:realm where:@"age > 5"];
```

#### 2.默认Realm数据库
您或许已经注意到，我们是通过调用 `[RLMRealm defaultRealm]` 来初始化 realm 变量并访问的。这个方法会返回一个 `RLMRealm` 对象，该对象映射到应用 `Documents` 文件夹（iOS）或者 `Application Support` 文件夹（macOS）中的 default.realm 文件。

Realm API 中的许多方法都存在一个接受 `RLMRealm` 实例为参数的版本，以及另一个使用默认 `Realm` 数据库的便利版本。例如，`[RLMObject allObjects]` 等同于 `[RLMObject allObjectsInRealm:[RLMRealm defaultRealm]]`。

请注意，默认的 Realm 构造方法和默认的 Realm 便利方法均不允许进行错误处理；只有在初始化 Realm 数据库不可能失败的情况下，才去使用它们。欲了解更多详情，请参见文档的错误处理部分。

#### 3.内存中 Realm 数据库
通过配置 RLMRealmConfiguration 中的 inMemoryIdentifier 属性，而不是 fileURL 属性，这样就能够创建一个完全在内存中运行的 Realm 数据库 (in-memory Realm)，它将不会存储在磁盘当中。设置 inMemoryIdentifier 会将 fileURL 置为 nil（反之亦然）。
```objc
RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
config.inMemoryIdentifier = @"MyInMemoryRealm";
RLMRealm *realm = [RLMRealm realmWithConfiguration:config error:nil];
````
内存中 Realm 数据库无法在应用启动期间存储数据，但是 Realm 数据库的其他功能都能正常使用，包括查询、关系以及线程安全。 如果您需要提供一种灵活的数据访问方式，而不占用磁盘空间的话，那么这是一个很有用的选择。

#### 4.错误处理
与任何磁盘 I/O 操作类似，如果资源受到限制，那么创建 RLMRealm 实例有可能会失败。实际上，只有在指定线程中第一次创建 Realm 实例时才可能会发生这种情况。在同一个线程中继续访问 Realm 数据库将会重用缓存的实例，这个操作是不可能失败的。

为了处理在指定线程中第一次创建 `Realm` 数据库时所发生的错误，我们提供了一个 `NSError` 指针类型的 `error` 参数：
```objc
NSError *error = nil;

RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
RLMRealm *realm = [RLMRealm realmWithConfiguration:config error:&error];
if (!realm) {
    // 错误处理
}
```

#### 5.预植 Realm 数据库
为应用提供一些初始数据的做法非常常见，这样就让用户在首次启动时进行访问。具体做法是：

1.首先，向 Realm 数据库中植入数据。所使用的数据模型应当与最终发布应用时所使用的 Realm 数据模型相同，然后向数据库中写入所需要的初始数据。由于 Realm 文件是跨平台的，因此您可以使用 macOS 应用（参见我们的 JSONImport 示例）,或者运行在模拟器中的 iOS 应用来完成数据的植入；  
2.在生成此 Realm 文件的代码中，最后您应当制作一个此数据库的压缩版本（参见 -[RLMRealm writeCopyToPath:error:])）。这可以减少 Realm 文件的大小，使您的应用体积更小，便于用户下载。  
3.将您 Realm 文件的压缩版本拖曳到应用的 Xcode 项目导航栏中；    
4.前往 Xcode 中应用目标的 “Build Phases” 选项卡，将 Realm 文件添加到 “Copy Bundle Resources” 构建阶段中。  
5.此时，应用已经可以访问该预植 Realm 文件 (bundled Realm) 了。您可以使用 `[[NSBundle mainBundle] pathForResource:ofType:]` 来获取路径；  
6.如果预植 Realm 数据库中的数据是固定不变、不需要修改的，那么您可以在 `RLMRealmConfiguration` 对象中，通过设置 readOnly = true 来直接从该路径中打开此文件。否则，如果要对初始数据进行更改的话，您需要使用` [[NSFileManager defaultManager] copyItemAtPath:toPath:error:]` 将预植文件复制到应用的 Documents 目录下。  

---

## 常用类
* `Realm` 移动端数据库容器的一个实例
* `RLMRealmConfiguration` 在打开 Realm 数据库之前，可以对其进行配置。通过创建一个 RLMRealmConfiguration 的对象实例，然后配置相应的属性
* `RLMObject `
* `RLMResults` 表示**检索**所返回的对象集合。
* `RLMArray ` 表示模型之间的**对多关系**。
* RLMLinkingObjects 类，表示模型之间的双向关系](#inverse-relationships)。
* RLMCollection 协议，定义了所有 Realm 集合的常用接口。

---
## 数据模型
### 1.支持的数据类型
Realm 支持下述属性类型：**`BOOL`**、**`bool`**、**`int`**、**`NSInteger`**、**`long`**、**`long long`**、**`float`**、**`double`**、**`NSString`**、**`NSDate`**、**`NSData`** 以及 被特殊类型标记的 **`NSNumber`** 。

CGFloat 属性被取消了，因为它不具备平台独立性。

您可以使用 `RLMArray<Object *><Object>` 和 RLMObject 的相关子类来构建关系模型，诸如一对多、一对一等。

RLMArray 支持 Objective-C 泛型。下面是不同类型的属性定义含义，以及相关用途：
* `RLMArray` 属性类型。
* `<Object *>` 泛型特化。这可以在编译时防止错误对象类型数组的使用。
* `<Object>` RLMArray 所遵守的协议。可以让 Realm 知晓如何在运行时确定该模型的架构。

### 2.必要属性
通常情况下，NSString *、NSData * 以及 NSDate * 属性可以设置为 nil。如果你必须需要这些值来进行展示的话，那么可以重写 RLMObject 子类的 +requiredProperties 方法。

例如，对于下述模型定义而言，尝试将 name 属性设置为 nil 将会抛出异常，但是将 birthday 属性设置为 nil 则是允许的：
```objc
@interface Person : RLMObject
@property NSString *name;
@property NSDate *birthday;
@end

@implementation Person
+ (NSArray *)requiredProperties {
    return @[@"name"];
}
@end
```
使用 NSNumber * 属性来存储可空数字。由于对于不同类型的数字而言，Realm 均使用了不同的存储格式，因此该属性必须由 RLMInt、RLMFloat、RLMDouble 或者 RLMBool 所标记。所有赋给该属性的值都会被转换为指定的类型。

请注意，NSDecimalNumber 值只能够分配给 RLMDouble 属性，并且 Realm 会存储双进度浮点数的近似值，而不是基础的十进制数值。

如果我们打算存储某人的年龄，而不是存储生日的话，那么还要允许不知道用户年龄的时候，可以将其设置为 nil：
```objc
@interface Person : RLMObject
@property NSString *name;
@property NSNumber<RLMInt> *age;
@end

@implementation Person
+ (NSArray *)requiredProperties {
    return @[@"name"];
}
@end
```
**RLMProperty 子类属性始终可以为 nil，因此无法包含在 requiredProperties 当中，此外 RLMArray 不支持存储 nil。**

### 3.主键
重写 +primaryKey 可以设置模型的主键。声明主键允许对象的查询和更新更加高效，并且会强制要求每个值保持唯一性。一旦将带有主键的对象添加到 Realm 数据库，那么该对象的主键将无法更改。
```objc
@interface Person : RLMObject
@property NSInteger id;
@property NSString *name;
@end

@implementation Person
+ (NSString *)primaryKey {
    return @"id";
}
@end
```

### 4.索引属性
要为某个属性建立索引，那么重写 +indexedProperties 即可。与主键类似，索引会稍微减慢写入速度，但是使用比较运算符进行查询的速度将会更快（它同样会造成 Realm 文件体积的增大，因为需要存储索引。）当您需要为某些特定情况优化读取性能的时候，那么最好添加索引。
```objc
@interface Book : RLMObject
@property float price;
@property NSString *title;
@end

@implementation Book
+ (NSArray *)indexedProperties {
    return @[@"title"];
}
@end 
```
### 5.被忽略属性
如果您不想将模型中的某些字段保存在 Realm 数据库中，那么可以重写 +ignoredProperties。Realm 不会干涉这些属性的正常操作；它们被成员变量所持有，并且可以随意重写它们的 Setter 和 Getter。
```objc
@interface Person : RLMObject
@property NSInteger tmpID;
@property (readonly) NSString *name; // 只读属性会被自动忽略
@property NSString *firstName;
@property NSString *lastName;
@end

@implementation Person
+ (NSArray *)ignoredProperties {
    return @[@"tmpID"];
}
- (NSString *)name {
    return [NSString stringWithFormat:@"%@ %@", self.firstName, self.lastName];
}
@end
```
被忽略属性的行为与正常属性完全相同。不过它们不支持 Realm 属性所特有的功能（例如：无法在查询中使用，也无法触发通知）。这些属性仍能够使用 KVO 进行观察。

### 6.默认属性值
重写 `+defaultPropertyValues`， 可以在每次创建对象时为属性提供默认值。
```objc
@interface Book : RLMObject
@property float price;
@property NSString *title;
@end

@implementation Book
+ (NSDictionary *)defaultPropertyValues {
    return @{@"price" : @0, @"title": @""};
}
@end
```

### 7.属性特性
Realm 将会忽略诸如 `nonatomic`、`atomic`、`strong`、`copy`、`weak` 之类的 Objective-C 属性特性。这些特性对于 Realm 存储机制而言并没有意义。Realm 有自己优化过的存储语义。所以为了避免有人对代码产生误解，我们建议您在编写模型时不要附加任何属性特性。不过，如果您切实设置了属性特性，那么在有 `RLMObject` 被写入到 Realm 数据库之前，这些属性特性都会一直生效。

无论该 `RLMObject` 对象是否被 Realm 数据库所管理，Getter 和 Setter 的自定名称仍然可以正常使用。

由于未被管理的 Realm 对象（即不被 Realm 数据库所管理的 Realm 模型类实例）只是单纯的 NSObject 子类，因此其中的属性特性可以像其他 NSObject 对象一样被观察到。

如果在 Swift 中使用 Realm Objective-C，那么模型属性需要添加 @objc dynamic var 特性，才能使这些属性能够访问到底层数据库的数据。（您同样可以用 objcMembers 来声明类，然后使用 dynamic var 来声明模型属性。）

### 8.属性备忘单
这个表格提供了声明模型属性的简易参考：

| 类型           | 非可空值形式                                               | 可空值形式                            |
| -------------- | ---------------------------------------------------------- | ------------------------------------- |
| Bool           | @property BOOL value;                                      | @property NSNumber<RLMBool> *value;   |
| Int            | @property int value;                                       | @property NSNumber<RLMInt> *value;    |
| Float          | @property float value;                                     | @property NSNumber<RLMFloat> *value;  |
| Double         | @property double value;                                    | @property NSNumber<RLMDouble> *value; |
| String         | @property NSString *value; 1                               | @property NSString *value;            |
| Data           | @property NSData *value; 1                                 | @property NSData *value;              |
| Date           | @property NSDate *value; 1                                 | @property NSDate *value;              |
| Object         | 不存在：必须是可空值                                       | @property Object *value;              |
| List           | @property RLMArray<Class *><Class> *value;                 | 不存在：必须是非可空值                |
| LinkingObjects | @property (readonly) RLMLinkingObjects<Object *> *value; 2 | 不存在：必须是非可空值                |

---
## 模型继承
Realm 允许对模型进行多级继承，从而允许跨模型实现代码复用，但是某些 Cocoa 特性是没有办法使用的，比如说那些支撑运行时类的多态性的特性。下面是可以实现的操作：  
+ 父类当中的类方法、实例方法和属性可以被子类继承；
+ 子类可以使用以父类为参数的方法和函数。
  

下列操作目前是无法实现的：  
+ 多态类之间的强制转换（例如：子类转换为另一个子类，子类转换为父类，父类转换成子类，等等）；
+ 同时对多个类进行检索；
+包含多个类的容器（RLMArray 以及 RLMResults）。

此外，如果您的代码实现允许的话，我们建议您使用下述模式，即使用类组合模式来构建子类，从而将其他类当中的逻辑给包含进去：
```objc
// Base Model
@interface Animal : RLMObject
@property NSInteger age;
@end
@implementation Animal
@end

// 与 Animal 一并组合的模型
@interface Duck : RLMObject
@property Animal *animal;
@property NSString *name;
@end
@implementation Duck
@end

@interface Frog : RLMObject
@property Animal *animal;
@property NSDate *dateProp;
@end
@implementation Frog
@end

// Usage
Duck *duck =  [[Duck alloc] initWithValue:@{@"animal" : @{@"age" : @(3)}, @"name" : @"Gustav" }];
```
---
## 关系
#### 1.多对一关系
要配置多对一或者一对一关系，在数据模型当中声明一个 RLMObject 子类类型的属性即可：
```objc
// Dog.h
@interface Dog : RLMObject
// ... 其余属性声明
@property Person *owner;
@end
```
操作关系属性的方法与其他属性类似：
```objc
Person *jim = [[Person alloc] init];
Dog    *rex = [[Dog alloc] init];
rex.owner = jim;
```
在使用 `RLMObject` 属性时，您可以使用正常的属性访问语法来访问嵌套属性。例如，`rex.owner.address.country` 将会遍历对象图，然后自动从 Realm 中检索出每个所需的对象。

#### 2.多对多关系
通过 `RLMArray` 属性，您可以为任意数量的对象或者所支持的原始类型之间构建关系。`RLMArray` 可以包含其它 `RLMObject` 类型，也可以包含简单类型的原始值，其接口与 `NSMutableArray` 非常类似。

`RLMArray` 所包含的 `Realm` 对象可能会存储多个相同 Realm 对象的引用，即便对象带有主键也是如此。例如，您或许会创建一个空的 `RLMArray`，然后连续三次向其中插入同一个对象；每次分别使用 0、1、2 的元素索引来访问的时候，`RLMArray` 都会返回同一个对象。

`RLMArray` 可以存储原始类型，从而代替一般的 `Realm` 对象。为了实现此功能， 请使用下列协议来约束 `RLMArray`：`RLMBool`、`RLMInt`、`RLMFloat`、`RLMDouble`、 `RLMString`、`RLMData` 或者 `RLMDate`。

默认情况下，包含原始类型的 `RLMArray` 可能也会包含空值（由 `NSNull` 表示）。 将数组标记为非可空（通过在数组所在的模型对象类型中重写 `+requiredProperties:`）， 同样也会导致数组当中的值变为非可空。

让我们给 `Person` 模型添加一个 `dogs` 属性，从而让其能够与多个 `Dog` 对象建立关系。首先，我们需要定义 `RLMArray<Dog>` 类型，也就是在 `Dog` 模型接口定义的底部使用这条宏：
```objc
// Dog.h
@interface Dog : RLMObject
// ... property declarations
@end

RLM_ARRAY_TYPE(Dog) // 定义 RLMArray<Dog> 类型
```
`RLM_ARRAY_TYPE` 宏创建了一个协议，从而允许您使用 `RLMArray<Dog>` 这种语法。如果这条宏没有放置在模型接口定义的底部，那么这个模型类就必须前置声明。

接下来，您就可以声明 `RLMArray<Dog>` 类型的属性了：
```objc
// Person.h
@interface Person : RLMObject
// ...其他属性声明
@property RLMArray<Dog *><Dog> *dogs;
@end
```
您可以照常对 RLMArray 属性进行访问和赋值：
```objc
// Jim 是 Rex 和所有名为 "Fido" 狗狗的主人
RLMResults<Dog *> *someDogs = [Dog objectsWhere:@"name contains 'Fido'"];
[jim.dogs addObjects:someDogs];
[jim.dogs addObject:rex];
```
注意，虽然可以给 `RLMArray` 属性赋值为 nil，但是这仅仅只会“清空”该数组，而不会将该数组给移除。这意味着您永远都可以向 `RLMArray` 属性中添加对象，即便其之前曾被置为 nil。

`RLMArray` 属性会确保其内部的插入次序不会被打乱。

**注意，目前暂时不支持对包含原始类型的 RLMArray 进行查询。**

#### 3.双向关系
关系是单向的。以 `Person` 和 `Dog` 这两个类为例。如果 `Person.dogs` 连接了一个 Dog 实例，那么您可以随着该连接从 `Person` 访问到对应的 `Dog`，但是是没有办法从 `Dog` 访问到对应的 `Person` 对象的。您可以设置一个一对一属性 `Dog.owner` 从而连接到 `Person`，但是这些连接实际上仍然是互相独立的。**给 `Person.dogs` 添加一个 `Dog` 对象并不会将该对象的 `Dog.owner` 属性设置为对应的 `Person`。为了解决这个问题，Realm 提供了连接对象属性，从而表示这种双向关系**
```objc
@interface Dog : RLMObject
@property NSString *name;
@property NSInteger age;
@property (readonly) RLMLinkingObjects *owners;
@end

@implementation Dog
+ (NSDictionary *)linkingObjectsProperties {
    return @{
        @"owners": [RLMPropertyDescriptor descriptorWithClass:Person.class propertyName:@"dogs"],
    };
}
@end
```
借助连接对象属性，可以从特定属性获取连接到指定对象的所有对象。Dog 对象可以拥有一个名为 owners 属性，它包含所有 dogs 属性有该 Dog 对象的 Person 对象。将这个 owners 属性设置为 RLMLinkingObjects 类型，然后重写 +[RLMObject linkingObjectsProperties] 来表示 owners 与 Person 模型对象之间的关系。

---
## 数据库操作-增删改查
### (一)增
#### 创建对象

当定义完数据模型之后，就可以实例化 RLMObject 子类了， 然后还可以向 Realm 数据库中添加新的实例
```objc
// Dog 数据模型
@interface Dog : RLMObject
@property NSString *name;
@property NSInteger age;
@end

// Implementation
@implementation Dog
@end
```
创建新对象的方法有很多种：
```objc
// (1) 创建 Dog 对象，然后设置其属性
Dog *myDog = [[Dog alloc] init];
myDog.name = @"Rex";
myDog.age = 10;

// (2) 从字典中创建 Dog 对象
Dog *myOtherDog = [[Dog alloc] initWithValue:@{@"name" : @"Pluto", @"age" : @3}];

// (3) 从数组中创建 Dog 对象
Dog *myThirdDog = [[Dog alloc] initWithValue:@[@"Pluto", @3]];
```
对象创建之后，您就可以将其添加到 Realm 数据库了：
```objc
// 获取默认的 Realm 数据库
RLMRealm *realm = [RLMRealm defaultRealm];
// （每个线程）只需执行一次

// 在事务中向 Realm 数据库中添加数据
[realm beginWriteTransaction];
[realm addObject:myDog];
[realm commitWriteTransaction];
```

### (二)改
将对象添加到 Realm 数据库之后，您仍然可以继续使用它，并且对其进行的所有更改都会被存储（必须要在写入事务当中进行）。当写入事务提交之后，其他使用同一个 Realm 数据库的线程所做的更改都可以继续进行。

#### 1.直接更新
您可以在写入事务中，通过设置对象的属性从而完成更新。
```objc
// 在事务中更新对象
[realm beginWriteTransaction];
author.name = @"Thomas Pynchon";
[realm commitWriteTransaction];
```

#### 2.键值编码
RLMObject、RLMResult 和 RLMArray 均允许使用 键值编码(KVC)。 当您需要在运行时决定何种属性需要进行更新的时候， 这个方法就非常有用了。

**批量更新对象时，为集合实现 KVC 是一个很好的做法， 这样就不用承受遍历集合时为每个项目创建访问器 所带来的性能损耗。**
```objc
RLMResults<Person *> *persons = [Person allObjects];
[[RLMRealm defaultRealm] transactionWithBlock:^{
    [[persons firstObject] setValue:@YES forKeyPath:@"isFirst"];
    // 将每个 person 对象的 planet 属性设置为 "Earth"
    [persons setValue:@"Earth" forKeyPath:@"planet"];
}];
```

#### 3.通过主键更新
如果数据模型类中包含了主键，那么 可以使用 -[RLMRealm addOrUpdateObject:]，从而让 Realm 基于主键来自动更新或者添加对象。
```objc
// 创建一个 book 对象，其主键与之前存储的 book 对象相同
Book *cheeseBook = [[Book alloc] init];
cheeseBook.title = @"Cheese recipes";
cheeseBook.price = @9000;
cheeseBook.id = @1;

// 更新这个 id = 1 的 book
[realm beginWriteTransaction];
[realm addOrUpdateObject:cheeseBook];
[realm commitWriteTransaction];
```
如果这个主键值为 “1” 的 Book 对象已经存在于数据库当中 ，那么该对象只会进行更新。如果不存在的话， 那么一个全新的 Book 对象就会被创建出来，并被添加到数据库当中。

您可以通过传递一个子集，其中只包含打算更新的值， 从而对带有主键的对象进行部分更新：
```objc
// 假设主键为 `1` 的 "Book" 对象已经存在
[realm beginWriteTransaction];
[Book createOrUpdateInRealm:realm withValue:@{@"id": @1, @"price": @9000.0f}];
// book 对象的 `title` 属性仍旧保持不变
[realm commitWriteTransaction];
```
**如果没有定义主键，那么最好不要对这类对象调用本节中所示的方法（也就是这些以 OrUpdate 结尾的方法）。**

### (三)删
在写入事务中，将要删除的对象传递给 -[RLMRealm deleteObject:] 方法。
```objc
// cheeseBook 存储在 Realm 数据库中

// 在事务中删除对象
[realm beginWriteTransaction];
[realm deleteObject:cheeseBook];
[realm commitWriteTransaction];
```

您同样也可以删除存储在 Realm 数据库当中的所有数据。请注意，Realm 文件会保留在磁盘上所占用的空间，从而为以后的对象预留足够的空间，从而实现快速存储。
```objc
// 从 Realm 数据库中删除所有对象
[realm beginWriteTransaction];
[realm deleteAllObjects];
[realm commitWriteTransaction];
```

### (四)查
查询将会返回一个 RLMResults 实例，其中包含了一组 RLMObject 对象。RLMResults 的接口与 NSArray 基本相同，并且可以使用索引下标来访问包含在 RLMResults 当中的对象。与 NSArray 所不同的是，RLMResults 中元素的类型是固定的，并且只能持有一个 RLMObject 子类类型。

从 Realm 数据库中检索对象的最基本方法是 +[RLMObject allObjects]，这个方法将会返回 RLMObject 子类类型在默认 Realm 数据库当中的查询到的所有数据，并以 RLMResults 实例的形式返回。
```objc
RLMResults<Dog *> *dogs = [Dog allObjects]; // 从默认的 Realm 数据库中遍历所有 Dog 对象
```

#### 1.条件查询
如果您对 NSPredicate 有所了解的话，那么您就已经掌握了在 Realm 中进行查询的方法了。RLMObjects、RLMRealm、RLMArray 和 RLMResults 均提供了相关的方法，从而只需传递 NSPredicate 实例、断言字符串、或者断言格式化字符串来查询特定的 RLMObject 实例，这与对 NSArray 进行查询所类似。

例如，下面这个例子通过调用 [RLMObject objectsWhere:] 方法，从默认 Realm 数据库中遍历出所有棕黄色、名字以 “B” 开头的狗狗：
```objc
// 使用断言字符串来查询
RLMResults<Dog *> *tanDogs = [Dog objectsWhere:@"color = 'tan' AND name BEGINSWITH 'B'"];

// 使用 NSPredicate 来查询
NSPredicate *pred = [NSPredicate predicateWithFormat:@"color = %@ AND name BEGINSWITH %@",
                                                     @"tan", @"B"];
tanDogs = [Dog objectsWithPredicate:pred];
```

Realm 支持大多数常见的断言：

+ 比较操作数可以是属性名，也可以是常量。但至少要有一个操作数是属性名；
+ 比较操作符 ==、<=、<、>=、>、!= 和 BETWEEN 支持 int、long、long long、float、double 以及 NSDate 这几种属性类型，例如 age == 45；
+ 比较是否相同：== 和 !=，例如，[Employee objectsWhere:@"company == %@", company]；
+ 比较操作符 == 和 != 支持布尔属性；
+ 对于 NSString 和 NSData 属性而言，支持使用 ==、!=、BEGINSWITH、CONTAINS 和 ENDSWITH 操作符，例如 name CONTAINS 'Ja'；
+ 对于 NSString 属性而言，LIKE 操作符可以用来比较左端属性和右端表达式：? 和 * 可用作通配符，其中 ? 可以匹配任意一个字符，* 匹配 0 个及其以上的字符。例如：value LIKE '?bc*' 可以匹配到诸如 “abcde” 和 “cbc” 之类的字符串；
+ 字符串的比较忽略大小写，例如 name CONTAINS[c] 'Ja'。请注意，只有 “A-Z” 和 “a-z” 之间的字符大小写会被忽略。[c] 修饰符可以与 [d] 修饰符结合使用；
+ 字符串的比较忽略变音符号，例如 name BEGINSWITH[d] 'e' 能够匹配到 étoile。这个修饰符可以与 [c] 修饰符结合使用。（这个修饰符只能够用于 Realm 所支持的字符串子集：参见当前的限制一节来了解详细信息。）
+ Realm 支持以下组合操作符：“AND”、“OR” 和 “NOT”，例如 name BEGINSWITH 'J' AND age >= 32；
+ 包含操作符：IN，例如 name IN {'Lisa', 'Spike', 'Hachi'}；
+ 空值比较：==、!=，例如 [Company objectsWhere:@"ceo == nil"]。请注意，Realm 将 nil 视为一种特殊值，而不是某种缺失值；这与 SQL 不同，nil 等同于自身；
+ ANY 比较，例如 ANY student.age < 21；
+ RLMArray 和 RLMResults 属性支持聚集表达式：@count、@min、@max、@sum 和 @avg，例如 [Company objectsWhere:@"employees.@count > 5"] 可用以检索所有拥有 5 名以上雇员的公司。
+ 支持子查询，不过存在以下限制：
  - @count 是唯一一个能在 SUBQUERY 表达式当中使用的操作符；
  - SUBQUERY(…).@count 表达式只能与常量相比较；
  - 目前仍不支持关联子查询。

#### 2.排序
RLMResults 允许您指定一个排序标准，然后基于**`关键路径`**、**`属性`**或者多个**`排序描述符`**来进行排序。例如，下列代码让上述示例中返回的 Dog 对象按名字进行升序排序：
```objc
// 对颜色为棕黄色、名字以 "B" 开头的狗狗进行排序
RLMResults<Dog *> *sortedDogs = [[Dog objectsWhere:@"color = 'tan' AND name BEGINSWITH 'B'"]
                                    sortedResultsUsingKeyPath:@"name" ascending:YES];
```
关键路径同样也可以是某个多对一关系属性。
```objc
RLMResults<Person *> *dogOwners = [Person allObjects];
RLMResults<Person *> *ownersByDogAge = [dogOwners sortedResultsUsingKeyPath:@"dog.age" ascending:YES];
```

请注意，`sortedResultsUsingKeyPath:` 和 `sortedResultsUsingProperty:` 不支持 将多个属性用作排序基准，此外也无法链式排序（只有最后一个 sortedResults... 调用会被使用）。 如果要对多个属性进行排序，请使用 `sortedResultsUsingDescriptors:`，然后向其中输入多个 RLMSortDescriptor` 对象。

#### 3.链式查询
与传统数据库相比，Realm 查询引擎的一个独特特性就是：它能够用很小的事务开销来实现链式查询，而不是每条查询都要接二连三地分别去单独访问数据库服务器。

如果您需要获取一个棕黄色狗狗的结果集，然后在此基础上再获取名字以 ‘B’ 开头的棕黄色狗狗，那么您可以像这样将这两个查询连接起来：
```objc
RLMResults<Dog *> *tanDogs = [Dog objectsWhere:@"color = 'tan'"];
RLMResults<Dog *> *tanDogsWithBNames = [tanDogs objectsWhere:@"name BEGINSWITH 'B'"];
```

#### 4.结果的自更新
RLMObject 实例是底层数据的动态体现，其会自动进行更新，这意味着您无需去重新检索结果。它们会直接映射出 Realm 数据库在当前线程中的状态，包括当前线程上的写入事务。唯一的例外是，在使用 for...in 枚举时，它会将刚开始遍历时满足匹配条件的所有对象给遍历完，即使在遍历过程中有对象被过滤器修改或者删除。
```objc
RLMResults<Dog *> *puppies = [Dog objectsInRealm:realm where:@"age < 2"];
puppies.count; // => 0

[realm transactionWithBlock:^{
    [Dog createInRealm:realm withValue:@{@"name": @"Fido", @"age": @1}];
}];

puppies.count; // => 1
```

#### 5.限制查询结果
大多数其他数据库技术都提供了从检索中对结果进行“分页”的能力（例如 SQLite 中的 “LIMIT” 关键字）。这通常是很有必要的，可以避免一次性从硬盘中读取太多的数据，或者将太多查询结果加载到内存当中。

**`由于 Realm 中的检索是惰性的，因此这行这种分页行为是没有必要的`**。因为 Realm 只会在检索到的结果被明确访问时，才会从其中加载对象。

如果由于 UI 相关或者其他代码实现相关的原因导致您需要从检索中获取一个特定的对象子集，这和获取 RLMResults 对象一样简单，只需要读出您所需要的对象即可。
```objc
// 循环读取出前 5 个 Dog 对象
// 从而限制从磁盘中读取的对象数量
RLMResults<Dog *> *dogs = [Dog allObjects];
for (NSInteger i = 0; i < 5; i++) {
    Dog *dog = dogs[i];
    // ...
}
```
---

## 数据迁移

### (一)本地迁移
通过设置 RLMRealmConfiguration.schemaVersion 以及 RLMRealmConfiguration.migrationBlock 可以定义本地迁移。迁移模块将提供所有的逻辑操作，以便将数据模型从之前的架构转换为新的架构。每当用配置对象创建完 RLMRealm 之后，如果需要进行迁移的话，那么迁移模块就会将 RLMRealm 更新至指定的架构版本。
假设我们需要将上面所声明的 Person 模型进行迁移。下述代码是最精简的数据模块：
```objc
// 此段代码位于 [AppDelegate didFinishLaunchingWithOptions:]

RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
// 设置新的架构版本。必须大于之前所使用的版本
// （如果之前从未设置过架构版本，那么当前的架构版本为 0）
config.schemaVersion = 1;

// 设置模块，如果 Realm 的架构版本低于上面所定义的版本，
// 那么这段代码就会自动调用
config.migrationBlock = ^(RLMMigration *migration, uint64_t oldSchemaVersion) {
    // 我们目前还未执行过迁移，因此 oldSchemaVersion == 0
    if (oldSchemaVersion < 1) {
        // 没有什么要做的！
        // Realm 会自行检测新增和被移除的属性
        // 然后会自动更新磁盘上的架构
    }
};

// 通知 Realm 为默认的 Realm 数据库使用这个新的配置对象
[RLMRealmConfiguration setDefaultConfiguration:config];

// 现在我们已经通知了 Realm 如何处理架构变化，
// 打开文件将会自动执行迁移
[RLMRealm defaultRealm];
```

#### 1.值的更新
虽然这个迁移操作是最精简的了，但是我们需要让这个闭包能够自行计算新的属性（这里指的是 fullName），这样才有意义。 在迁移模块中，我们能够调用 [RLMMigration enumerateObjects:block:] 来枚举特定类型的每个 RLMObject 对象，然后执行必要的迁移逻辑。注意，对枚举中每个已存在的 RLMObject 实例来说，应该是通过访问 oldObject 对象进行访问，而更新之后的实例应该通过 newObject 进行访问：
```objc
// 此段代码位于 [AppDelegate didFinishLaunchingWithOptions:]

RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
config.schemaVersion = 1;
config.migrationBlock = ^(RLMMigration *migration, uint64_t oldSchemaVersion) {
    // 我们目前还未执行过迁移，因此 oldSchemaVersion == 0
    if (oldSchemaVersion < 1) {
        // enumerateObjects:block: 方法将会遍历
        // 所有存储在 Realm 文件当中的 `Person` 对象
        [migration enumerateObjects:Person.className
                              block:^(RLMObject *oldObject, RLMObject *newObject) {

        // 将两个 name 合并到 fullName 当中
        newObject[@"fullName"] = [NSString stringWithFormat:@"%@ %@",
                                      oldObject[@"firstName"],
                                      oldObject[@"lastName"]];
        }];
    }
};
[RLMRealmConfiguration setDefaultConfiguration:config];
```
一旦迁移成功结束，Realm 文件和其中的所有对象都可被您的应用正常访问。

#### 2.属性重命名
在迁移过程中对类中某个属性进行重命名操作， 比起拷贝值和保留关系来说要更为高效。

要在迁移过程中对某个属性就进行重命名的话，请确保您的新模型当中的这个属性是一个全新的名字， 它的名字不能和原有模型当中的名字重合。

如果新的属性拥有不同的可空性或者索引设置的话， 这些配置会在重命名操作期间生效。

下面是一个例子，展示了您该如何将 Person 的 yearsSinceBirth 属性重命名为 age 属性：
```objc
// 此段代码位于 [AppDelegate didFinishLaunchingWithOptions:]

RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
config.schemaVersion = 1;
config.migrationBlock = ^(RLMMigration *migration, uint64_t oldSchemaVersion) {
    // 我们目前还未执行过迁移，因此 oldSchemaVersion == 0
    if (oldSchemaVersion < 1) {
        // 重命名操作必须要在 `enumerateObjects:` 调用之外进行
        [migration renamePropertyForClass:Person.className oldName:@"yearsSinceBirth" newName:@"age"];
    }
};
[RLMRealmConfiguration setDefaultConfiguration:config];
```

#### 3.线性迁移
假如说，我们的应用有两个用户： JP 和 Tim。JP 经常更新应用，但 Tim 却经常跳过某些版本。所以 JP 可能下载过这个应用的每一个版本，并且一步一步地跟着更新构架：第一次下载更新后，数据库架构从 v0 更新到 v1；第二次架构从 v1 更新到 v2…以此类推，井然有序。相反，Tim 很有可能直接从 v0 版本直接跳到了 v2 版本。 因此，您应该使用非嵌套的 if (oldSchemaVersion < X) 结构来构造您的数据库迁移模块，以确保无论用户在使用哪个版本的架构，都能完成必需的更新。

当您的用户不按套路出牌，跳过有些更新版本的时候，另一种情况也会发生。假如您在 v2 里删掉了一个 “email” 属性，然后在 v3 里又把它重新引进了。假如有个用户从 v1 直接跳到 v3，那 Realm 不会自动检测到 v2 的这个删除操作，因为存储的数据架构和代码中的架构吻合。这会导致 Tim 的 Person 对象有一个 v3 的 email 属性，但里面的内容却是 v1 的。这个看起来没什么大问题，但是假如两者的内部存储类型不同（比如说： 从 ISO email 标准格式变成了自定义格式），那麻烦就大了。为了避免这种不必要的麻烦，我们推荐您在 if (oldSchemaVersion < 3) 语句中，清空所有的 email 属性。

### （二）迁移同步(比较复杂不常用)
当 Realm 数据库与 Realm 对象服务器同步时，迁移过程会有所不同——在很多情况下，其实会更加简单。下面是您所需要知晓的全部内容：

+ 无需设置架构版本（尽管您可以这样做）；
+ 新增内容的更改会自动进行，例如添加类或者向类中添加字段；
+ 从架构中将某个字段移除并不会从数据库中删除该字段，而是指示 Realm 忽略该字段。新的对象创建的时候仍然会使用这些属性，但是它们都将会被设置为 null。不可空的字段将被恰当地设置为零/空值：数字字段将被置为 0，字符串属性将被置为空字符串，等等。
+ 您不能添加迁移模块。
假设您的应用中有一个 Dog 类：
```objc
@interface Dog : RLMObject
@property NSString *name;
@end
```
现在您需要添加 Person 类，并建立一个到 Dog 的 owner 关系。除了添加类和相关属性之外，在同步之前您无需执行任何操作：
```objc
@interface Dog : RLMObject
@property NSString *name;
@property Person   *owner;
@end
RLM_ARRAY_TYPE(Dog)

@interface Person : RLMObject
@property NSString *name;
@property NSDate   *birthdate;
@end
RLM_ARRAY_TYPE(Person)

// Objecitve-C 引用类型的非可空属性
// 必须以这种形式进行声明：
@implementation Person
+ (NSArray *)requiredProperties {
    return @[@"name"];
}
@end

NSURL *syncServerURL = [NSURL URLWithString:@"http://localhost:9080/Dogs"];
RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
config.syncConfiguration = [[RLMSyncConfiguration alloc] initWithUser:user realmURL:syncServerURL];

RLMRealm *realm = [RLMRealm realmWithConfiguration:config error:nil];
```
由于可同步 Realm 数据库不支持迁移模块，因此迁移当中的破坏性更改——例如主键更改、既有字段的字段类型更改（同时保留相同的名称），以及将属性从可空更改为非可空，诸如此类的操作，都需要用另外的方式来进行处理。创建一个新的具备新架构的可同步 Realm 数据库，然后将数据从旧的 Realm 数据库复制到新的 Realm 数据库：
```objc
@interface Dog : RLMObject
@property NSString *name;
@property Person *owner;
@end
RLM_ARRAY_TYPE(Dog)

@interface Person : RLMObject
@property NSString *name;
@end
RLM_ARRAY_TYPE(Person)

@implementation Person
+ (NSArray *)requiredProperties {
    return @[@"name"];
}

@interface PersonV2 : RLMObject
@property NSString *name;
@end
RLM_ARRAY_TYPE(PersonV2)

@implementation PersonV2
+ (NSArray *)requiredProperties {
    return @[];
}

NSURL *syncServerURL = [NSURL URLWithString: @"realm://localhost:9080/Dogs"];
RLMRealmConfiguration *config = [RLMRealmConfiguration defaultConfiguration];
config.syncConfiguration = [[RLMSyncConfiguration alloc] initWithUser:user realmURL:syncServerURL];
// 限制初始对象类型
config.objectClasses = @[Dog.class, Person.class];

RLMRealm *initialRealm = [RLMRealm realmWithConfiguration:config error:nil];

syncServerURL = [NSURL URLWithString: @"realm://localhost:9080/DogsV2"];
config = [RLMRealmConfiguration defaultConfiguration];
config.syncConfiguration = [[RLMSyncConfiguration alloc] initWithUser:user realmURL:syncServerURL];
// 限制新对象类型
config.objectClasses = @[Dog.class, PersonV2.class];

RLMRealm *newRealm = [RLMRealm realmWithConfiguration:config error:nil];
```
此外，对于可同步 Realm 数据库而言，还可以在客户端上编写一个通知处理器，或者使用 Node.js SDK 在服务器上编写一段 JavaScript 函数（如果您所使用的对象服务器版本支持的话），来执行自定义迁移。但是，如果迁移过程中出现了破坏性更改，那么 Realm 将停止与 Realm 对象服务器进行同步，并产生 Bad changeset received 错误。
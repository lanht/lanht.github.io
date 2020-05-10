---
title: iOS面试题收录（一）
date: 2020-05-08 20:03:48
tags:
categories:
    - iOS面试汇总
---
主题：是什么？

#### 关键字const什么含义？

```objc
const int a;
int const a;
const int *a;
int const *a;
int * const a;
int const * const a;
```

1>. 前两个的作用是一样：a 是一个常整型数
2>. 第三、四个意味着 a 是一个指向常整型数的指针(整型数是不可修改的，但指针可以)
3>. 第五个的意思：a 是一个指向整型数的常指针(指针指向的整型数是可以修改的，但指针是不可修改的)
4>. 最后一个意味着：a 是一个指向常整型数的常指针(指针指向的整型数是不可修改的，同时指针也是不可修改的)

#### #import跟 #include 有什么区别，@class呢，#import<> 跟 #import“”有什么区别？

1. \#import是Objective-C导入头文件的关键字，#include是C/C++导入头文件的关键字，使用#import头文件会自动只导入一次，不会重复导入。

2. @class告诉编译器某个类的声明，当执行时，才去查看类的实现文件，可以解决头文件的相互包含。

3. \#import<>用来包含系统的头文件，#import””用来包含用户头文件。



#### @property 的本质是什么？ivar、getter、setter 是如何生成并添加到这个类中的？

@property 的本质是:@property = ivar + getter + setter

“属性” (property)有两大概念：ivar（实例变量）、getter+setter（存取方法）

“属性” (property)作为 Objective-C 的一项特性，主要的作用就在于封装对象中的数据。 Objective-C 对象通常会把其所需要的数据保存为各种实例变量。实例变量一般通过“存取方法”(access method)来访问。其中，“获取方法” (getter)用于读取变量值，而“设置方法” (setter)用于写入变量值。



#### @property中有哪些属性关键字以及作用？

**线程安全**

**nonatomic ：**非原子操作。决定编译器生成的setter和getter方法是否是原子操作，一般使用nonatomic，效率高。
**atomic：**多线程安全，但是性能低

**内存管理**

**strong：**持有特性。setter方法将传入参数先保留，再赋值，传入参数的retaincount会+1。
**copy ：**拷贝特性。setter方法将传入对象复制一份，需要完全一份新的变量时。
**assign：**用于基本数据类型
**retain**：相当于ARC中的strong

**读写操作**

**readwrite：**可读可写特性。需要生成getter方法和setter方法
**readonly：**只读特性。只会生成getter方法，不会生成setter方法，不希望属性在类外改变。



#### 什么情况使用 weak 关键字，相比 assign 有什么不同？

1>.在 ARC 中,在有可能出现循环引用的时候,往往要通过让其中一端使用 weak 来解决,比如: delegate 代理属性。
2>.自身已经对它进行一次强引用,没有必要再强引用一次,此时也会使用 weak,自定义 IBOutlet 控件属性一般也使用 weak（因为父控件的subViews数组已经对它有一个强引用）。

不同点：
assign 可以用非 OC 对象，而 weak 必须用于 OC 对象。
weak 表明该属性定义了一种“非拥有关系”。在属性所指的对象销毁时，属性值会自动清空(nil)。



#### 用@property声明的 NSString / NSArray / NSDictionary 经常使用 copy 关键字，为什么？如果改用strong关键字，可能造成什么问题？

用 @property 声明 NSString、NSArray、NSDictionary 经常使用 copy 关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary，他们之间可能进行赋值操作（就是把可变的赋值给不可变的），为确保对象中的字符串值不会无意间变动，应该在设置新属性值时拷贝一份。

1>. 因为父类指针可以指向子类对象,使用 copy 的目的是为了让本对象的属性不受外界影响,使用 copy 无论给我传入是一个可变对象还是不可对象,我本身持有的就是一个不可变的副本。
2>. 如果我们使用是 strong ,那么这个属性就有可能指向一个可变对象,如果这个可变对象在外部被修改了,那么会影响该属性。
总结：使用copy的目的是，防止把可变类型的对象赋值给不可变类型的对象时，可变类型对象的值发送变化会无意间篡改不可变类型对象原来的值。



#### 浅拷贝和深拷贝的区别？

**浅拷贝：**对一个对象地址的拷贝。源对象和副本对象是同一对象
**深拷贝：**对一个对象的拷贝。源对象和副本对象是不同的两个对象



#### self.跟self->什么区别？

1>. self.是调用get方法或者set放
2>. self是当前本身，是一个指向当前对象的指针
3>. self->是直接访问成员变量



#### 一个objc对象的isa的指针指向什么？有什么作用？

指向他的类对象,从而可以找到对象上的方法



#### frame 和 bounds 有什么不同？

**frame：**该view在父view坐标系统中的位置和大小。(参照点是父view的坐标系统)

**bounds：**该view在本身坐标系统中的位置和大小。(参照点是本身坐标系统)



#### Objective-C的类可以多重继承么？没有的话用什么代替？可以实现多个接口么？Category是什么？重写一个类的方式用继承好还是分类好？为什么？

OC不可以多继承，OC是单继承。有时可以用分类和协议来代替多继承
可以实现多个接口（协议）
Category是类别；一般情况用分类好，用Category去重写类的方法，仅对本Category有效，不会影响到其他类与原有类的关系。



#### Object-C有私有方法吗？私有变量呢？

1>.OC没有类似@private的修饰词来修饰方法，只要写在.h文件中，就是公共方法
2>. 如果你不在.h文件中声明，只在.m文件中实现，或在.m文件的Class Extension里声明，那么基本上和私有方法差不多，可以使用类扩展（Extension）来增加私有方法和私有变量
3>. 使用private修饰的全局变量是私有变量



#### 用伪代码写一个线程安全的单例模式

```objc

static XXManager * instance = nil;
+ (instancetype)shareInstance {
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
});
return instance;
}

+ (id)allocWithZone:(struct _NSZone *)zone {

      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
        instance = [super allocWithZone:zone];
});
return instance;
}
- (id)copyWithZone:(NSZone *)zone {
return instance;
}
```



#### category(类别) 和 extension(扩展) 的区别

1>. 类别有名字，类扩展没有分类名字，是一种特殊的分类。
2>. 类别只能扩展方法（属性仅仅是声明，并没真正实现），类扩展可以扩展属性、成员变量和方法。
3>. 继承可以增加，修改或者删除方法，并且可以增加属性。



#### delegate 和 notification 的区别

二者都用于传递消息，不同之处主要在于一个是一对一的，另一个是一对多的
**notification：**不需要两者之间有联系,实现一对多消息的转发
**delegate：**需要两者之间必须建立联系，不然没法调用代理的方法



#### Objective-C 如何对内存管理的，说说你的看法和解决方法？

Objective-C的内存管理主要有三种方式ARC(自动内存计数)、手动内存计数、内存池。
1>. 自动内存计数ARC：由Xcode自动在App编译阶段，在代码中添加内存管理代码。
2>. 手动内存计数MRC：遵循内存谁申请、谁释放；谁添加，谁释放的原则。
3>. 内存释放池Release Pool：把需要释放的内存统一放在一个池子中，当池子被抽干后(drain)，池子中所有的内存空间也被自动释放掉。内存池的释放操作分为自动和手动。自动释放受runloop机制影响。



#### GCD 与 NSOperation 的区别

**NSOperation:**相对于GCD来说，更加强大。可以给operation之间添加依赖关系、取消一个正在执行的operation、暂停和恢复operationQueue等

**GCD:** 是一种更轻量级的，以FIFO(先进先出，后进后出)的顺序执行并发任务。使用GCD我们并不用关心任务的调度情况，而是系统会自动帮我们处理。但是GCD的短板也是非常明显的，比如我们想要给任务之间添加依赖关系、取消或者暂停一个正在执行的任务时就会变得束手无策。



#### OC中创建线程的方法是什么？如果在主线程中执行代码，方法是什么？

```objc
// 创建线程的方法

- [NSThread detachNewThreadSelector:nil toTarget:nil withObject:nil]

- [self performSelectorInBackground:nil withObject:nil];

- [[NSThread alloc] initWithTarget:nil selector:nil object:nil];

- dispatch_async(dispatch_get_global_queue(0, 0), ^{});

- [[NSOperationQueue new] addOperation:nil];

// 主线程中执行代码的方法

- [self performSelectorOnMainThread:nil withObject:nil waitUntilDone:YES];

- dispatch_async(dispatch_get_main_queue(), ^{});

- [[NSOperationQueue mainQueue] addOperation:nil];
```



#### runloop 和线程有什么关系?

runloop与线程是一一对应的，一个runloop对应一个核心的线程，为什么说是核心的，是因为runloop是可以嵌套的，但是核心的只能有一个，他们的关系保存在一个全局的字典里。 runloop是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。 runloop在第一次获取时被创建，在线程结束时被销毁。 对于主线程来说，runloop在程序一启动就默认创建好了。 对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

#### 介绍下layoutSubview和drawRect

layoutSubviews调用情况

- init初始化UIView不会触发调用 

- addSubview会触发调用 
- 改变view的width和height的时候回触发
- 调用 一个UIScrollView滚动会触发调用 
- 旋转screen会触发调用 
- 改变一个UIView大小的时候会触发
- superView的layoutSubviews事件 直接调用setLayoutSubviews会触发调用 
- -(void)viewWillAppear:(BOOL)animated会触发一次调用
-  -(void)viewDidAppear:(BOOL)animated 看情况

 可能有调用 drawRect调用情况 

- 如果UIView没有设置frame大小，直接导致drawRect不能被自动调用。 
- drawRect在loadView和viewDidLoad这两个方法之后调用 调用sizeToFit后自动调用drawRect 
- 通过设置contentMode值为UIViewContentModeRedraw。那么每次设置或者更改frame自动调用drawRect。 
- 直接调用setNeedsDisplay或者setNeedsDisplayInRect会触发调用

#### UIview 和CAlayer 是什么关系? 你 CLayer做过什么?

区别： 首先UIView可以响应事件，Layer不可以.

关系：

1. UIView是CALayer的delegate 
2. UIView主要处理事件，CALayer负责绘制就更好 
3. 每个 UIView 内部都有一个 CALayer 在背后提供内容的绘制和显示，并且 UIView 的尺寸样式都由内部的 Layer 所提供。两者都有树状层级结构，layer 内部有 SubLayers，View 内部有 SubViews.但是 Layer 比 View 多了个AnchorPoint 

用layer做过什么：创建隐式动画 绘制边框圆角

#### iOS UIViewController的完整生命周期?

按照执行顺序排列：

1>. `initWithCoder：`通过nib文件初始化时触发。
2>. `awakeFromNib：`nib文件被加载的时候，会发生一个`awakeFromNib`的消息到nib文件中的每个对象。
3>. `loadView：`开始加载视图控制器自带的view。
4>. `viewDidLoad：`视图控制器的view被加载完成。
5>. `viewWillAppear：`视图控制器的view将要显示在window上。
6>. `updateViewConstraints：`视图控制器的view开始更新AutoLayout约束。
7>. `viewWillLayoutSubviews：`视图控制器的view将要更新内容视图的位置。
8>. `viewDidLayoutSubviews：`视图控制器的view已经更新视图的位置。
9>. `viewDidAppear：`视图控制器的view已经展示到window上。
10>. `viewWillDisappear：`视图控制器的view将要从window上消失。
11>.`viewDidDisappear：`视图控制器的view已经从window上消失。



#### tableView的重用机制？

UITableView 通过重用单元格来达到节省内存的目的: 通过为每个单元格指定一个重用标识符，即指定了单元格的种类,当屏幕上的单元格滑出屏幕时，系统会把这个单元格添加到重用队列中，等待被重用，当有新单元格从屏幕外滑入屏幕内时，从重用队列中找看有没有可以重用的单元格，如果有，就拿过来用，如果没有就创建一个来使用



####  NSIRLConnection 和NSLRLSession 的区别是 么? NSURLProtocol是做什么的?

1. 下载 NSURLConnection下载文件时，先是将整个文件下载到内存，然后再写入到沙盒，如果文件比较大，就会出现内存暴涨的情况。 而使用NSURLSessionUploadTask下载文件，会默认下载到沙盒中的tem文件中，不会出现内存暴涨的情况，但是在下载完成后会把tem中的临时文件删除，需要在初始化任务方法时，在completionHandler回调中增加保存文件的代码 

2. 请求方法的控制 NSURLConnection实例化对象，实例化开始，默认请求就发送(同步发送),不需要调用start方法。而cancel可以停止请求的发送，停止后不能继续访问，需要创建新的请求。 NSURLSession有三个控制方法，取消(cancel)、暂停(suspend)、继续(resume)，暂停以后可以通过继续恢复当前的请求任务。 使用NSURLSession进行断点下载更加便捷. NSURLSession的构造方法（sessionWithConfiguration:delegate:delegateQueue）中有一个NSURLSessionConfiguration类的参数可以设置配置信息，其决定了cookie，安全和高速缓存策略，最大主机连接数，资源管理，网络超时等配置。NSURLConnection不能进行这个配置，相比较与NSURLConnection依赖与一个全局的配置对象，缺乏灵活性而言，NSURLSession有很大的改进
   
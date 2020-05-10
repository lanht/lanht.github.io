---
title: iOS面试题收录（二）
date: 2020-05-08 21:02:13
tags:
categories:
	- iOS面试汇总
---

主题：怎么做？

#### 设计模式是什么？ 你知道哪些设计模式，并简要叙述？

**设计模式**是一套被 反复使用、多数人知晓、经过分类编目的、代码设计经验的总结。

**单例模式：**单例模式确保某一个类只有一个实例，并提供一个访问它的全剧访问点。[具体的详情可点击进入查看](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F4a1dYXPf1oSfZYS7J1IQRA)
**工厂模式：**工厂父类负责定义创建产品对象的公共接口，而工厂子类则负责生产具体的产品对象，即通过不停的工厂子类来创建不同的产品对象。[具体的详情可点击进入查看](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5bcb0362e51d450e7042eb6d)
**代理模式 :**为某个对象提供一个代理，并由这个代理对象控制对原对象的访问。[具体的详情可点击进入查看](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F22eVsnQP2cTccWwIaLfz5g)
**适配器模式：** 将一个接口转换成客户希望的另一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。适配器模式的别名是包装器模式（Wrapper），是一种结构型设计模式。[具体的详情可点击进入查看](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2F3abKTDIVy8BJJjThr2jgMQ)
**装饰者模式：** 不改变原有对象的前提下，动态地给一个对象增加一些额外的功能。[具体的详情可点击进入查看](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FkoaeYH1U-nfrsSQ8FH3hhQ)



#### MVC 和 MVVM 的区别？

**MVC**
MVC（Model-View-Controller）模式结构图，可分为三部分：模型（Model）、视图（View）、控制器（Controller）。其在MVC模式中所扮演的角色分别为：
Model：模型管理应用程序的数据，响应有关其状态信息（通常来自View）的请求，并响应指令以更改状态（通常来自Controller）。
View：视图管理数据的展示。
Controller：控制器解释用户的输入，并通知模型、视图进行状态更新
所有通信都是单向的。
优点：对Controller进行瘦身，将View内部的细节封装起来了，外界不知道View内部的具体实现
缺点：View和Controller依赖于Model

**MVVM**
MVVM（Model View View-Model）就是为了解决过于臃肿的问题。MVVM的思想是将Controller中UI控制逻辑与业务逻辑进行分离，并抽离出一个View-Model来完成UI控制的逻辑。而Controller只需要负责业务逻辑即可

唯一的区别是，View-Model可以调用Model定义的方法，从Model中获取数据以用于View，并对数据进行预处理，使View可以直接使用。View又可以向View-Model发出用户的操作命令，从而更改Model。MVVM实现了一种双向绑定机制。

优点：降低了View和Model之间的耦合；分离了业务逻辑和视图逻辑。
缺点：View和Model双向绑定导致bug难以定位，两者中的任何一方出现问题，另一方也会出现问题；增加了胶水代码



#### iOS 内存的使用和优化的注意事项?

**重用问题：**如`UITableViewCells`、`UICollectionViewCells`、`UITableViewHeaderFooterViews`。设置正确的`reuseIdentifier`，充分重用
**不要使用太复杂的XIB/Storyboard：**载入时就会将`XIB`/`storyboard`需要的所有资源，包括图片全部载入内存。
**尽量把views设置为不透明：**当opque为NO的时候，图层的半透明取决于图片和其本身合成的图层为结果，可提高性能
**选择正确的数据结构：**学会选择对业务场景最合适的数组结构是写出高效代码的基础。
**gzip/zip压缩：**当从服务端下载相关附件时，可以通过gzip/zip压缩后再下载，使得内存更小，下载速度也更快。
**延迟加载：**对于不应该使用的数据，使用延迟加载方式。对于不需要马上显示的视图，使用延迟加载方式。比如，网络请求失败时显示的提示界面，可能一直都不会使用到，因此应该使用延迟加载。
**数据缓存：**对于cell的行高要缓存起来，使得reload数据时，效率也极高。
而对于那些网络数据，不需要每次都请求的，应该缓存起来。可以写入数据库，也可以通过plist文件存储
**处理内存警告：**一般在基类统一处理内存警告，将相关不用资源立即释放掉



#### iOS 你在项目中是怎么优化内存的？

> 这个问题有时候笔试中也有，有时候有些面试官会在面试中问你这个问题

1>.避免庞大的Xib(Xib比frame消耗更多的CPU资源)
2>.不要阻塞主线程，尽量把耗时的操作放到子线程
3>.重用和延迟加载
4>.尽量减少视图数量和层次
5>.优化TableView,为了使TableVIew有更好的滚动性能可采取以下措施：

- 正确使用ruseIdentifier来重用cells
- 采用懒加载即延迟加载的方式加载cell上的控件
- 当TableView滑动的时候不加载
- 缓存cell的高度。在呈现cell前，把cell的高度计算好缓存起来，避免每次加载cell的时候都要计算
- 尽量使用不透明的UI控件



#### 写一个完整的代理，包括声明、实现

```objc

// 创建
@protocol PersonDelagate
@required
-(void)eat:(NSString *)foodName;
@optional
-(void)run;
@end

// 声明 .h
@interface Person: NSObject<PersonDelagate>
@end

// 实现 .m
@implementation Person

- (void)eat:(NSString *)foodName {
NSLog(@"吃:%@", foodName);
}

- (void)run {
   NSLog(@"run");
}
@end
```



#### 分析json、xml 的区别? json、xml 解析 式的底层是如何让处理的

(一) JSON与XML的区别： 

（1）可读性方面：基本相同，XML的可读性比较好； 

（2）可扩展性方面：都具有良好的扩展性； 

（3）编码难度方面：相对而言，JSON的编码比较容易； 

（4）解码难度：JSON的解码难度基本为零，XML需要考虑子节点和父节点；

（5）数据体积方面：JSON相对于XML来讲，数据体积小，传递的速度比较快； 

（6）数据交互方面：JSON与javascript的交互更加方便，更容易解析处理，更好的数据交互； 

（7）数据描述方面：XML对数据描述性比较好 

（8）传输速度方面：JSON的速度远远快于XML。 

(二）JSON与XML底层实现原理： 　

（1）JSON底层原理：遍历字符串中的字符，最终根据格式规定的特殊字符，比如{}、[]、：等进行区分，{}号表示字典，[]号表示数组，：号是字典的键和值的分水岭，最终仍是将JSON转化为字典，只不过字典中的值可能是“字典、数组或者字符串而已”。 　　

（2）XML底层原理：XML解析常用的解析方法有两种：DOM解析和SAX解析；DOM采用的是树形结构的方式访问XML文档，而SAX采用的是事件模型；DOM解析把XML文档转化为一个包含其内容的树，并可以对树进行遍历，使用DOM解析器的时候需要处理整个XML文档，所以对内存和性能的要求比较高；SAX在解析XML文档的时候可以触发一系列的事件，当发现给定的tag的时候，他可以激活一个回调方法，告诉该方法指定的标签已经找到，SAX对内存的要求通常会比较低，因为他让开发人员自己来决定所要处理的tag，特别是当开发人员只需要处理文档中所包含部分数据时，SAX这种扩展能力得到了更好的体现。



#### 如何处理UITableVier 中Cell 动态计算高度的问题，都有哪些方案?

1. 你的Cell要使用AutoLayout来布局约束这是必须的； 设置tableview的estimatedRowHeight为一个非零值，这个属性是设置一个预估的高度值，不用太精确。 设置tableview的rowHeight属性为UITableViewAutomaticDimension

2. 第三方 UITableView+FDTemplateLayoutCell



#### 怎么高效的实现控件的圆角效果

```objc
//绘制圆角 
-(UIImageView *)roundedRectImageViewWithCornerRadius:(CGFloat)cornerRadius { 
  UIBezierPath *bezierPath = [UIBezierPath bezierPathWithRoundedRect:self.bounds 		cornerRadius:cornerRadius]; 
  CAShapeLayer *layer = [CAShapeLayer layer]; 
  layer.path = bezierPath.CGPath; 
  self.layer.mask = layer; 
  return self; 
}
```



####  假如Controller太臃肿，如何优化?

1. 将网络请求抽象到单独的类中 方便在基类中处理公共逻辑； 方便在基类中处理缓存逻辑，以及其它一些公共逻辑； 方便做对象的持久化。 

2. 将界面的封装抽象到专门的类中 构造专门的 UIView 的子类，来负责这些控件的拼装。这是最彻底和优雅的方式，不过稍微麻烦一些的是，你需要把这些控件的事件回调先接管，再都一一暴露回 Controller。 
3. 构造 ViewModel 借鉴MVVM。具体做法就是将 ViewController 给 View 传递数据这个过程，抽象成构造 ViewModel 的过程。 
4. 专门构造存储类 专门来处理本地数据的存取。 
5. 整合常量
   

#### 项目中网络层如何做安全处理?

1. 判断API的调用请求是否来自于经过授权的APP。如若不是则拒绝请求访问 
2. 在数据请求的过程中进行URL加密处理：防止反编译，接口信息被静态分析。 
3. 数据传输加密：对客户端传输数据提供有效的加密方案，以防止网络接口的拦截。 如果可以尽量使用HTTPS，可以有效的避免接口数据在传输中被攻击。
   
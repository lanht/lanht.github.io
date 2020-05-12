---
title: AFNetworking使用分析
date: 2020-05-12 14:45:03
tags:
categories:	
	- iOS
---

基于3.x进行使用分析

#### 前景回顾

NSURLConnection（在iOS8.0后弃用）

NSURLSession

AF2.x是对NSURLConnection的封装

AF3.x是对NSURLSession的封装

**AF2.x为什么需要常驻线程？**

先来看看 NSURLConnection 发送请求时的线程情况，NSURLConnection 是被设计成异步发送的，调用了start方法后，NSURLConnection 会新建一些线程用底层的 CFSocket 去发送和接收请求，在发送和接收的一些事件发生后通知原来线程的Runloop去回调事件。

**AF3.x为什么不需要常驻线程？**



#### AF分为如下5个功能模块：

+ 网络通信模块NSURLSession(AFURLSessionManager、AFHTTPSessionManger)

+ 网络状态监听模块(Reachability)

+ 网络通信安全策略模块(Security)

+ 网络通信信息序列化/反序列化模块(Serialization)

+ 对于iOS UIKit库的扩展(UIKit)

其核心当然是网络通信模块AFURLSessionManager。大家都知道，AF3.x是基于NSURLSession来封装的。所以这个类围绕着NSURLSession做了一系列的封装。而其余的四个模块，均是为了配合网络通信或对已有UIKit的一个扩展工具包。

其中AFHTTPSessionManager是继承于AFURLSessionManager的，我们一般做网络请求都是用这个类，**但是它本身是没有做实事的，只是做了一些简单的封装，把请求逻辑分发给父类AFURLSessionManager或者其它类去做。**




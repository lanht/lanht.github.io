---
title: load和initialize方法的区别
date: 2020-05-12 09:59:04
tags:
categories:
	-iOS
---

#### 调用方式

1、load是根据函数地址直接调用

2、initialize是通过objc_msgSeng调用

#### 调用时刻

1、load是runtime加载类、分类的时候调用(只会调用一次)

2、initialize是类第一次接收到消息的时候调用, 每一个类只会initialize一次(如果子类没有实现initialize方法, 会调用父类的initialize方法, 所以父类的initialize方法可能会调用多次)

#### load和initializee的调用顺序

**load**

1、先调用类的load

- 先编译的类, 优先调用load（先编译，先调用）

- 调用子类的load之前, 会先调用父类的load

2、再调用分类的load

- 先编译的分类, 优先调用load（先编译，先调用）

**initialize**

先调用父类的+initialize，再调用子类的+initialize。（先初始化父类，再初始化子类，每个类只会初始化1次）
如果分类实现了+initialize，会覆盖原有类的+initialize

#### 总结

1. +initialize和+load的最大区别是

   - initialize是通过objc_msgSeng调用，load是根据函数地址直接调用；

   - 如果子类没有实现initialize方法, 会调用父类的initialize方法, 所以父类的initialize方法可能会调用多次
   - 如果分类实现了+initialize，会覆盖原有类的+initialize
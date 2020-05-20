---
title: NSURLSession
date: 2020-05-14 11:27:49
tags:
categoires:
	- iOS
---

### 一、概述

NSURLSession在2013年随着iOS7的发布一起面世，苹果对它的定位是作为NSURLConnection的替代者，然后逐步将NSURLConnection退出历史舞台。现在使用最广泛的第三方网络框架：AFNetworking、SDWebImage等等都使用了NSURLSession。

在WWDC 2013中，Apple的团队对NSURLConnection进行了重构，并推出了NSURLSession作为替代。NSURLSession将NSURLConnection替换为*NSURLSession*和*NSURLSessionConfiguration*，以及3个NSURLSessionTask的子类：*NSURLSessionDataTask*, *NSURLSessionUploadTask*, 和*NSURLSessionDownloadTask*。

![](1.jpg)

NSURLSessionTask及三个子类继承关系：

![](2.jpg)

**NSURLSessionTask及其子类**

**`NSURLSessionTask`本身是一个抽象类，在使用的时候，通常是根据具体的需求使用它的几个子类** 

**`NSURLSessionDataTask`可以用来发送常见的Get，Post请求，既可以用来上传也可以用来下载**

 **`NSURLSessionDownloadTask`可以用来发送下载请求，专门用来下载数据**

 **`NSURLSessionUploadTask`可以用来发送上传请求，专门用来上传数据**

### 二、NSURLSession的使用

NSURLSession 本身是不会进行请求的，而是通过创建 task 的形式进行网络请求（resume() 方法的调用），同一个 NSURLSession 可以创建多个 task，并且这些 task 之间的 cache 和 cookie 是共享的。NSURLSession的使用有如下几步：

- 第一步：创建NSURLSession对象
- 第二步：使用NSURLSession对象创建Task
- 第三步：启动任务



#### 创建NSURLSession对象

NSURLSession对象的创建有如下三种方法：

（1）直接创建

```objective-c
NSURLSession *session = [NSURLSession sharedSession];
```

（2）配置后创建

```objective-c
[NSURLSession sessionWithConfiguration:defaultSessionConfiguration];
```

（3）设置加代理获得

```objective-c
// 使用代理方法需要设置代理,但是session的delegate属性是只读的,要想设置代理只能通过这种方式创建session
NSURLSession *session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]
    delegate:self
    delegateQueue:[[NSOperationQueue alloc] init]];
```

关于NSURLSession的配置有三种类型：

```objective-c
//默认的配置会将缓存存储在磁盘上
+ (NSURLSessionConfiguration *)defaultSessionConfiguration;

//瞬时会话模式不会创建持久性存储的缓存
+ (NSURLSessionConfiguration *)ephemeralSessionConfiguration;

//后台会话模式允许程序在后台进行上传下载工作
+ (NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier
```



#### 使用NSURLSession对象创建Task

NSURLSessionTask的创建要根据具体需要创建相应类型的Task。

（1）NSURLSessionDataTask

通过request对象或url创建：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request;

- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url;
```

 

通过request对象或url创建，同时指定任务完成后通过completionHandler指定回调的代码块：

```objective-c
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionDataTask *)dataTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
```

（2）NSURLSessionUploadTask

通过request创建，在上传时指定文件源或数据源：

```objective-c
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;  
   
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;  
  
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;  
```

 

通过completionHandler指定任务完成后的回调代码块：

```objective-c
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
```

（3）NSURLSessionDownloadTask

下载任务支持断点续传，第三种方式是通过之前已经下载的数据来创建下载任务：

```objective-c
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;    
    
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;    
  
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData;
```

同样地可以通过completionHandler指定任务完成后的回调代码块：

```objective-c
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
```

我们在使用三种 task 的任意一种的时候都可以指定相应的代理。NSURLSession 的代理对象结构如下：

![](3.jpg)

*NSURLSessionDelegate* – 作为所有代理的基类，定义了网络请求最基础的代理方法。

*NSURLSessionTaskDelegate* – 定义了网络请求任务相关的代理方法。

*NSURLSessionDownloadDelegate* – 用于下载任务相关的代理方法，比如下载进度等等。

*NSURLSessionDataDelegate* – 用于普通数据任务和上传任务。

#### 启动任务

```objective-c
// 启动任务
[task resume];
```



### 三、GET请求与POST请求

1、GET 请求

```objective-c
//1、创建NSURLSession对象
NSURLSession *session = [NSURLSession sharedSession];

//2、利用NSURLSession创建任务(task)
NSURL *url = [NSURL URLWithString:@"http://www.xxx.com/login?username=myName&pwd=myPsd"];

NSURLSessionDataTask *task = [session dataTaskWithURL:url completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
    
    NSLog(@"%@",[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding]);
    //打印解析后的json数据
    //NSLog(@"%@", [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]);

}];

//3、执行任务
[task resume];
```

2、POST请求

```objective-c
//1、创建NSURLSession对象
NSURLSession *session = [NSURLSession sharedSession];

//2、利用NSURLSession创建任务(task)
NSURL *url = [NSURL URLWithString:@"http://www.xxx.com/login"];

//创建请求对象里面包含请求体
NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
request.HTTPMethod = @"POST";
request.HTTPBody = [@"username=myName&pwd=myPsd" dataUsingEncoding:NSUTF8StringEncoding];

NSURLSessionDataTask *task = [session dataTaskWithRequest:request completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {
      
    NSLog(@"%@",[[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding]);
    //打印解析后的json数据
    //NSLog(@"%@", [NSJSONSerialization JSONObjectWithData:data options:kNilOptions error:nil]);

}];

//3、执行任务
 [task resume];
```



### 四、文件的上传

我们可以使用NSURLSessionUploadTask进行文件的上传

**创建方法**（基于 `NSURLSession` 对象）

```objective-c
// 使用 NSURLRequest 对象创建，上传时指定文件源
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL;  

// 使用 NSURLRequest 对象创建，上传时指定数据源   
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData;  
  
- (NSURLSessionUploadTask *)uploadTaskWithStreamedRequest:(NSURLRequest *)request;
```



**CompletionHandler**

```objective-c
- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromFile:(NSURL *)fileURL completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request fromData:(NSData *)bodyData completionHandler:(void (^)(NSData *data, NSURLResponse *response, NSError *error))completionHandler;
```

**关于分片上传和断点续传上传**

可以参考阿里云AliyunOSSiOS源码及文档

文档地址：https://help.aliyun.com/document_detail/31850.html?spm=a2c4g.11186623.6.610.62d72201Fvv00V

### 五、文件的下载

我们可以使用NSURLSessionDownloadTask实现文件的下载。

`NSURLSessionDownloadTask` 主要用于 **文件下载**，它针对大文件的网络请求做了更多的处理，如：下载进度、断点续传等。

**创建方法**（基于 `NSURLSession` 对象）

```objective-c
// 使用 NSURLRequest 对象创建
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request;    
    
// 使用 NSURL 对象创建
- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url;    
  
// 使用之前已经下载的数据来创建
- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData;
```



**CompletionHandler**

```objective-c
- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionDownloadTask *)downloadTaskWithURL:(NSURL *)url completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;    

- (NSURLSessionDownloadTask *)downloadTaskWithResumeData:(NSData *)resumeData completionHandler:(void (^)(NSURL *location, NSURLResponse *response, NSError *error))completionHandler;
```



**关于断点续传下载**

设计思路是用请求的Range

iOS断点续传：https://cloud.tencent.com/developer/article/1423899

AFNetworking 实现断点续传：http://www.yangjie.hu/2018/05/15/breakpoint-resume-in-iOS/

### 六、补充

iOS 利用AFNetworking实现大文件分片上传

https://www.jianshu.com/p/7919c620967e



### 参考链接

NSURLSession：https://www.jianshu.com/p/e798c6fe26ea

iOS 网络(1)——NSURLSession: http://chuquan.me/2019/07/21/ios-network-nsurlsession/

iOS NSURLSession 详解：https://juejin.im/entry/58aacabcac502e006973ce03
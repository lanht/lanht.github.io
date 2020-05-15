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

NSURLSession发起的请求，不再需要在当前线程进行代理方法的回调！可以指定回调的delegateQueue，这样我们就不用为了等待代理回调方法而苦苦保活线程了。

**为什么AF3.0中需要设置**

```objc
self.operationQueue.maxConcurrentOperationCount = 1;
```

而AF2.0却不需要？

解答：功能不一样：AF3.0的operationQueue是用来接收NSURLSessionDelegate回调的，鉴于一些多线程数据访问的安全性考虑，设置了maxConcurrentOperationCount = 1来达到串行回调的效果。
而AF2.0的operationQueue是用来添加operation并进行并发请求的，所以不要设置为1。

#### AF分为如下5个功能模块：

+ 网络通信模块(AFURLSessionManager、AFHTTPSessionManger)

+ 网络状态监听模块(Reachability)

+ 网络通信安全策略模块(Security)

+ 网络通信信息序列化/反序列化模块(Serialization)

+ 对于iOS UIKit库的扩展(UIKit)

##### 网络通信模块

其核心当然是网络通信模块AFURLSessionManager。大家都知道，AF3.x是基于NSURLSession来封装的。所以这个类围绕着NSURLSession做了一系列的封装。而其余的四个模块，均是为了配合网络通信或对已有UIKit的一个扩展工具包。

其中AFHTTPSessionManager是继承于AFURLSessionManager的，我们一般做网络请求都是用这个类，**但是它本身是没有做实事的，只是做了一些简单的封装，把请求逻辑分发给父类AFURLSessionManager或者其它类去做。**

AFURLSessionManager初始化

```objc
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }

    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }

    self.sessionConfiguration = configuration;

    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;

    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];

    self.responseSerializer = [AFJSONResponseSerializer serializer];

    self.securityPolicy = [AFSecurityPolicy defaultPolicy];

#if !TARGET_OS_WATCH
    self.reachabilityManager = [AFNetworkReachabilityManager sharedManager];
#endif

    self.mutableTaskDelegatesKeyedByTaskIdentifier = [[NSMutableDictionary alloc] init];

    self.lock = [[NSLock alloc] init];
    self.lock.name = AFURLSessionManagerLockName;

    [self.session getTasksWithCompletionHandler:^(NSArray *dataTasks, NSArray *uploadTasks, NSArray *downloadTasks) {
        for (NSURLSessionDataTask *task in dataTasks) {
            [self addDelegateForDataTask:task uploadProgress:nil downloadProgress:nil completionHandler:nil];
        }

        for (NSURLSessionUploadTask *uploadTask in uploadTasks) {
            [self addDelegateForUploadTask:uploadTask progress:nil completionHandler:nil];
        }

        for (NSURLSessionDownloadTask *downloadTask in downloadTasks) {
            [self addDelegateForDownloadTask:downloadTask progress:nil destination:nil completionHandler:nil];
        }
    }];

    return self;
}
```

在初始化方法中，需要完成初始化一些自己持有的实例：

1. 初始化**会话配置**（NSURLSessionConfiguration），默认为 `defaultSessionConfiguration`
2. 初始化会话（session），并设置会话的代理以及代理队列
3. 初始化管理**响应序列化**（AFJSONResponseSerializer），**安全认证**（AFSecurityPolicy）以及**监控网络状态**（AFNetworkReachabilityManager）的实例
4. 初始化保存 data task 的字典（mutableTaskDelegatesKeyedByTaskIdentifier）



在获得了 `AFURLSessionManager` 的实例之后，我们可以通过以下方法创建 `NSURLSessionDataTask` 的实例：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler;

- (NSURLSessionUploadTask *)uploadTaskWithRequest:(NSURLRequest *)request
                                         fromFile:(NSURL *)fileURL
                                         progress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                                completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject, NSError  * _Nullable error))completionHandler;

...

- (NSURLSessionDownloadTask *)downloadTaskWithRequest:(NSURLRequest *)request
                                             progress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                                          destination:(nullable NSURL * (^)(NSURL *targetPath, NSURLResponse *response))destination
                                    completionHandler:(nullable void (^)(NSURLResponse *response, NSURL * _Nullable filePath, NSError * _Nullable error))completionHandler;

```



我们将以 `- [AFURLSessionManager dataTaskWithRequest:uploadProgress:downloadProgress:completionHandler:]` 方法的实现为例，分析它是如何实例化并返回一个 `NSURLSessionTask` 的实例的：

```objc
- (NSURLSessionDataTask *)dataTaskWithRequest:(NSURLRequest *)request
                               uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
                             downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
                            completionHandler:(nullable void (^)(NSURLResponse *response, id _Nullable responseObject,  NSError * _Nullable error))completionHandler {

    __block NSURLSessionDataTask *dataTask = nil;
    url_session_manager_create_task_safely(^{
        dataTask = [self.session dataTaskWithRequest:request];
    });

    [self addDelegateForDataTask:dataTask uploadProgress:uploadProgressBlock downloadProgress:downloadProgressBlock completionHandler:completionHandler];

    return dataTask;
}
```

> `url_session_manager_create_task_safely` 的调用是因为苹果框架中的一个 bug [#2093](https://github.com/AFNetworking/AFNetworking/issues/2093)，如果有兴趣可以看一下，在这里就不说明了

1. 调用 `- [NSURLSession dataTaskWithRequest:]` 方法传入 `NSURLRequest`
2. 调用 `- [AFURLSessionManager addDelegateForDataTask:uploadProgress:downloadProgress:completionHandler:]` 方法返回一个 `AFURLSessionManagerTaskDelegate` 对象
3. 将 `completionHandler` `uploadProgressBlock` 和 `downloadProgressBlock` 传入该对象并在相应事件发生时进行回调

```objc
 (void)addDelegateForDataTask:(NSURLSessionDataTask *)dataTask
                uploadProgress:(nullable void (^)(NSProgress *uploadProgress)) uploadProgressBlock
              downloadProgress:(nullable void (^)(NSProgress *downloadProgress)) downloadProgressBlock
             completionHandler:(void (^)(NSURLResponse *response, id responseObject, NSError *error))completionHandler
{
    AFURLSessionManagerTaskDelegate *delegate = [[AFURLSessionManagerTaskDelegate alloc] init];
    delegate.manager = self;
    delegate.completionHandler = completionHandler;

    dataTask.taskDescription = self.taskDescriptionForSessionTasks;
    [self setDelegate:delegate forTask:dataTask];

    delegate.uploadProgressBlock = uploadProgressBlock;
    delegate.downloadProgressBlock = downloadProgressBlock;
}
```

在这个方法中同时调用了另一个方法 `- [AFURLSessionManager setDelegate:forTask:]` 来设置代理：

```objc
- (void)setDelegate:(AFURLSessionManagerTaskDelegate *)delegate
            forTask:(NSURLSessionTask *)task
{

	#1: 检查参数, 略

    [self.lock lock];
    self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)] = delegate;
    [self addNotificationObserverForTask:task];
    [self.lock unlock];
}
```



##### 网络状态监听模块

`AFNetworkReachabilityManager` 是对 `SystemConfiguration` 模块的封装，苹果的文档中也有一个类似的项目 [Reachability](https://developer.apple.com/library/ios/samplecode/reachability/) 这里对网络状态的监控跟苹果官方的实现几乎是完全相同的。

**AFNetworkReachabilityManager 的使用和实现**

`AFNetworkReachabilityManager` 的使用还是非常简单的，只需要三个步骤，就基本可以完成对网络状态的监控。

1. [初始化 `AFNetworkReachabilityManager`]()
2. [调用 `startMonitoring` 方法开始对网络状态进行监控]()
3. [设置 `networkReachabilityStatusBlock` 在每次网络状态改变时, 调用这个 block]()



##### 网络通信安全策略模块

用于HTTPS中的SSL校验规则设置，验证证书是否正确。

自 iOS9 发布之后，由于新特性 [App Transport Security](https://developer.apple.com/library/ios/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html) 的引入，在默认行为下是不能发送 HTTP 请求的。很多网站都在转用 HTTPS，而 `AFNetworking` 中的 `AFSecurityPolicy` 就是为了阻止中间人攻击，以及其它漏洞的工具。

`AFSecurityPolicy` 主要作用就是验证 HTTPS 请求的证书是否有效，如果 app 中有一些敏感信息或者涉及交易信息，一定要使用 HTTPS 来保证交易或者用户信息的安全。

+ AFSSLPinningMode

  使用 `AFSecurityPolicy` 时，总共有三种验证服务器是否被信任的方式：

  ```objc
  typedef NS_ENUM(NSUInteger, AFSSLPinningMode) {
      AFSSLPinningModeNone,
      AFSSLPinningModePublicKey,
      AFSSLPinningModeCertificate,
  };
  ```

  - `AFSSLPinningModeNone` 是默认的认证方式，只会在系统的信任的证书列表中对服务端返回的证书进行验证
  - `AFSSLPinningModeCertificate` 需要客户端预先保存服务端的证书
  - `AFSSLPinningModePublicKey` 也需要预先保存服务端发送的证书，但是这里只会验证证书中的公钥是否正确

+ 初始化以及设置

  在使用 `AFSecurityPolicy` 验证服务端是否受到信任之前，要对其进行初始化，使用初始化方法时，主要目的是设置**验证服务器是否受信任的方式**。

  ```objc
  + (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode {
      return [self policyWithPinningMode:pinningMode withPinnedCertificates:[self defaultPinnedCertificates]];
  }
  
  + (instancetype)policyWithPinningMode:(AFSSLPinningMode)pinningMode withPinnedCertificates:(NSSet *)pinnedCertificates {
      AFSecurityPolicy *securityPolicy = [[self alloc] init];
      securityPolicy.SSLPinningMode = pinningMode;
  
      [securityPolicy setPinnedCertificates:pinnedCertificates];
  
      return securityPolicy;
  }
  ```

  这里没有什么地方值得解释的。不过在调用 `pinnedCertificate` 的 setter 方法时，会从全部的证书中**取出公钥**保存到 `pinnedPublicKeys` 属性中。

  ```objc
  - (void)setPinnedCertificates:(NSSet *)pinnedCertificates {
      _pinnedCertificates = pinnedCertificates;
  
      if (self.pinnedCertificates) {
          NSMutableSet *mutablePinnedPublicKeys = [NSMutableSet setWithCapacity:[self.pinnedCertificates count]];
          for (NSData *certificate in self.pinnedCertificates) {
              id publicKey = AFPublicKeyForCertificate(certificate);
              if (!publicKey) {
                  continue;
              }
              [mutablePinnedPublicKeys addObject:publicKey];
          }
          self.pinnedPublicKeys = [NSSet setWithSet:mutablePinnedPublicKeys];
      } else {
          self.pinnedPublicKeys = nil;
      }
  }
  ```

  在这里调用了 `AFPublicKeyForCertificate` 对证书进行操作，返回一个公钥。

+ 操作 SecTrustRef

  对 `serverTrust` 的操作的函数基本上都是 C 的 API，都定义在 `Security` 模块中，先来分析一下在上一节中 `AFPublicKeyForCertificate` 的实现

  ```objc
  static id AFPublicKeyForCertificate(NSData *certificate) {
    	//1.初始化一坨临时变量
      id allowedPublicKey = nil;
      SecCertificateRef allowedCertificate;
      SecCertificateRef allowedCertificates[1];
      CFArrayRef tempCertificates = nil;
      SecPolicyRef policy = nil;
      SecTrustRef allowedTrust = nil;
      SecTrustResultType result;
  		
    	//2.使用 SecCertificateCreateWithData 通过 DER 表示的数据生成一个 SecCertificateRef，然后判断返回值是否为 NULL
    	//这里使用了一个非常神奇的宏 __Require_Quiet，它会判断 allowedCertificate != NULL 是否成立，如果 allowedCertificate 为空就会跳到 _out 标签处继续执行
      allowedCertificate = SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificate);
      __Require_Quiet(allowedCertificate != NULL, _out);
  		
    	//3.通过上面的 allowedCertificate 创建一个 CFArray
      allowedCertificates[0] = allowedCertificate;
      tempCertificates = CFArrayCreate(NULL, (const void **)allowedCertificates, 1, NULL);
  		
    	//4.创建一个默认的符合 X509 标准的 SecPolicyRef，通过默认的 SecPolicyRef 和证书创建一个 SecTrustRef 用于信任评估，对该对象进行信任评估，确认生成的 SecTrustRef 是值得信任的。
      policy = SecPolicyCreateBasicX509();
      __Require_noErr_Quiet(SecTrustCreateWithCertificates(tempCertificates, policy, &allowedTrust), _out);
    
    	//这里使用的 __Require_noErr_Quiet 和上面的宏差不多，只是会根据返回值判断是否存在错误
      __Require_noErr_Quiet(SecTrustEvaluate(allowedTrust, &result), _out);
  	
    	//5.获取公钥
      allowedPublicKey = (__bridge_transfer id)SecTrustCopyPublicKey(allowedTrust);
  
  _out:
      if (allowedTrust) {
          CFRelease(allowedTrust);
      }
  
      if (policy) {
          CFRelease(policy);
      }
  
      if (tempCertificates) {
          CFRelease(tempCertificates);
      }
  
      if (allowedCertificate) {
          CFRelease(allowedCertificate);
      }
  
      return allowedPublicKey;
  }
  ```

  对它的操作还有 `AFCertificateTrustChainForServerTrust` 和 `AFPublicKeyTrustChainForServerTrust` 但是它们几乎调用了相同的 API。

  ```objc
  static NSArray * AFCertificateTrustChainForServerTrust(SecTrustRef serverTrust) {
      CFIndex certificateCount = SecTrustGetCertificateCount(serverTrust);
      NSMutableArray *trustChain = [NSMutableArray arrayWithCapacity:(NSUInteger)certificateCount];
  
      for (CFIndex i = 0; i < certificateCount; i++) {
          SecCertificateRef certificate = SecTrustGetCertificateAtIndex(serverTrust, i);
          [trustChain addObject:(__bridge_transfer NSData *)SecCertificateCopyData(certificate)];
      }
  
      return [NSArray arrayWithArray:trustChain];
  }
  ```



+ 验证服务端是否受信

  验证服务端是否守信是通过 `- [AFSecurityPolicy evaluateServerTrust:forDomain:]` 方法进行的。

  ```objectivec
  - (BOOL)evaluateServerTrust:(SecTrustRef)serverTrust
                    forDomain:(NSString *)domain
  {
  
  	#1: 不能隐式地信任自己签发的证书
  
  	#2: 设置 policy
  
  	#3: 验证证书是否有效
  
  	#4: 根据 SSLPinningMode 对服务端进行验证
  
      return 
  }
  ```

  1. 不能隐式地信任自己签发的证书

     ```objective-c
      if (domain && self.allowInvalidCertificates && self.validatesDomainName && (self.SSLPinningMode == AFSSLPinningModeNone || [self.pinnedCertificates count] == 0)) {
          NSLog(@"In order to validate a domain name for self signed certificates, you MUST use pinning.");
          return NO;
      }
     ```

     所以如果没有提供证书或者不验证证书，并且还设置 `allowInvalidCertificates` 为**真**，满足上面的所有条件，说明这次的验证是不安全的，会直接返回 `NO`

  2. 设置 policy

     ```objective-c
     NSMutableArray *policies = [NSMutableArray array];
      if (self.validatesDomainName) {
          [policies addObject:(__bridge_transfer id)SecPolicyCreateSSL(true, (__bridge CFStringRef)domain)];
      } else {
          [policies addObject:(__bridge_transfer id)SecPolicyCreateBasicX509()];
      }
     ```

     如果要验证域名的话，就以域名为参数创建一个 `SecPolicyRef`，否则会创建一个符合 X509 标准的默认 `SecPolicyRef` 对象

  3. 验证证书的有效性

     ```objective-c
     if (self.SSLPinningMode == AFSSLPinningModeNone) {
          return self.allowInvalidCertificates || AFServerTrustIsValid(serverTrust);
      } else if (!AFServerTrustIsValid(serverTrust) && !self.allowInvalidCertificates) {
          return NO;
      }
     ```

     - 如果**只根据信任列表中的证书**进行验证，即 `self.SSLPinningMode == AFSSLPinningModeNone`。如果允许无效的证书的就会直接返回 `YES`。不允许就会对服务端信任进行验证。
     - 如果服务器信任无效，并且不允许无效证书，就会返回 `NO`

  4. 根据 `SSLPinningMode` 对服务器信任进行验证

     ```objective-c
     switch (self.SSLPinningMode) {
          case AFSSLPinningModeNone:
          default:
              return NO;
          case AFSSLPinningModeCertificate: {
              ...
          }
          case AFSSLPinningModePublicKey: {
              ...
          }
      }
     ```

     - `AFSSLPinningModeNone` 直接返回 `NO`
     - `AFSSLPinningModeCertificate`

     ```objective-c
     NSMutableArray *pinnedCertificates = [NSMutableArray array];
       for (NSData *certificateData in self.pinnedCertificates) {
           [pinnedCertificates addObject:(__bridge_transfer id)SecCertificateCreateWithData(NULL, (__bridge CFDataRef)certificateData)];
       }
       SecTrustSetAnchorCertificates(serverTrust, (__bridge CFArrayRef)pinnedCertificates);
     
       if (!AFServerTrustIsValid(serverTrust)) {
           return NO;
       }
     
       // obtain the chain after being validated, which *should* contain the pinned certificate in the last position (if it's the Root CA)
       NSArray *serverCertificates = AFCertificateTrustChainForServerTrust(serverTrust);
     
       for (NSData *trustChainCertificate in [serverCertificates reverseObjectEnumerator]) {
           if ([self.pinnedCertificates containsObject:trustChainCertificate]) {
               return YES;
           }
       }
     
       return NO;
     ```

     1. 从 `self.pinnedCertificates` 中获取 DER 表示的数据
     2. 使用 `SecTrustSetAnchorCertificates` 为服务器信任设置证书
     3. 判断服务器信任的有效性
     4. 使用 `AFCertificateTrustChainForServerTrust` 获取服务器信任中的全部 DER 表示的证书
     5. 如果 `pinnedCertificates` 中有相同的证书，就会返回 `YES`

     + AFSSLPinningModePublicKey

     ```objective-c
      NSUInteger trustedPublicKeyCount = 0;
       NSArray *publicKeys = AFPublicKeyTrustChainForServerTrust(serverTrust);
     
       for (id trustChainPublicKey in publicKeys) {
           for (id pinnedPublicKey in self.pinnedPublicKeys) {
               if (AFSecKeyIsEqualToKey((__bridge SecKeyRef)trustChainPublicKey, (__bridge SecKeyRef)pinnedPublicKey)) {
                   trustedPublicKeyCount += 1;
               }
           }
       }
       return trustedPublicKeyCount > 0;
     ```

     这部分的实现和上面的差不多，区别有两点

     1. 会从服务器信任中获取公钥
     2. `pinnedPublicKeys` 中的公钥与服务器信任中的公钥相同的数量大于 0，就会返回真

+ 与 AFURLSessionManager 协作

  在代理协议 `- URLSession:didReceiveChallenge:completionHandler:` 或者 `- URLSession:task:didReceiveChallenge:completionHandler:` 代理方法被调用时会运行这段代码

  ```objectivec
  if ([challenge.protectionSpace.authenticationMethod isEqualToString:NSURLAuthenticationMethodServerTrust]) {
      if ([self.securityPolicy evaluateServerTrust:challenge.protectionSpace.serverTrust forDomain:challenge.protectionSpace.host]) {
          disposition = NSURLSessionAuthChallengeUseCredential;
          credential = [NSURLCredential credentialForTrust:challenge.protectionSpace.serverTrust];
      } else {
          disposition = NSURLSessionAuthChallengeRejectProtectionSpace;
      }
  } else {
      disposition = NSURLSessionAuthChallengePerformDefaultHandling;
  }
  ```

  `NSURLAuthenticationChallenge` 表示一个认证的挑战，提供了关于这次认证的全部信息。它有一个非常重要的属性 `protectionSpace`，这里保存了需要认证的保护空间, 每一个 `NSURLProtectionSpace` 对象都保存了主机地址，端口和认证方法等重要信息。

  在上面的方法中，如果保护空间中的认证方法为 `NSURLAuthenticationMethodServerTrust`，那么就会使用在上一小节中提到的方法 `- [AFSecurityPolicy evaluateServerTrust:forDomain:]` 对保护空间中的 `serverTrust` 以及域名 `host` 进行认证

  根据认证的结果，会在 `completionHandler` 中传入不同的 `disposition` 和 `credential` 参数。

  

##### 网络通信信息序列化/反序列化模块

设置**网络请求**的序列化对象：满足**AFURLRequestSerialization**协议的AFHTTPRequestSerializer对象（当然还有别的子类），配置请求的**cookies**，**timeout**，字符串编码方式（**stringEncoding**），http的**headertype**，**user-agent**等。

设置**返回数据**的序列化对象：满足**AFURLResponseSerialization**协议的**AFHTTPResponseSerializer**对象（JSON、XML等其他对象默认配置了statuscode和contenttypes），配置返回数据的**acceptableStatusCodes**和**acceptableContentTypes**，检查返回数据**是否合法**并进行转化。

注：AFJSONResponseSerializer序列化对象的底层是使用Foundation中**内置的NSJSONSerialization对象**实现的。



##### 对于iOS UIKit库的扩展



#### 参考链接

AFNetworking 概述系列文章：https://draveness.me/afnetworking1/


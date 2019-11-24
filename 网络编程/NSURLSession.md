#一 概述
NSURLSession是IOS SDK提供的一组相对容易使用的网络API。它包括几个部分NSURLRequest，NSURLCache，NSURLSession，NSURLSessionConfiguration，NSURLSessionTask。IOS的网络编程除了NSURLSession，也可以使用NSURLConnection，只不过后者的易用性较差。网络开发的整体包括五个部分

>支持的协议（例如http） 
授权和证书（例如服务器要求提供用户名密码） 
cookie 存储（例如不存储cookie） 
cache 管理（例如只在内存cache，不cache到硬盘） 
配置管理（例如http headers等配置信息）

如图 

![img](https://img-blog.csdn.net/20150321075126656)

#二 简单介绍下NSURLSession的几个核心类
知道有这些类，大概是做什么的就行。

## 1.1NSURLSessionConfiguration
指定NSURLSession的配置信息。这些配置信息决定了NSURLSession的种类，HTTP的额外headers，请求的timeout时间，Cookie的接受策略等配置信息。更多的参见官方文档。

这里详细讲解下三种NSURLSessionConfiguration，这决定了NSURLSession种类。

+ `(NSURLSessionConfiguration *)defaultSessionConfiguration` 
defaultSession，使用基于硬盘的持久话Cache，保存用户的证书到钥匙串,使用共享cookie存储

+ `(NSURLSessionConfiguration *)ephemeralSessionConfiguration` 
配置信息和default大致相同。除了，不会把cache，证书，或者任何和Session相关的数据存储到硬盘，而是存储在内存中，生命周期和Session一致。比如浏览器无痕浏览等功能就可以基于这个来做。

+ `(NSURLSessionConfiguration *)backgroundSessionConfigurationWithIdentifier:(NSString *)identifier` 
创建一个可以在后台甚至APP已经关闭的时候仍然在传输数据的会话。注意，后台Session一定要在创建的时候赋予一个唯一的identifier，这样在APP下次运行的时候，能够根据identifier来进行相关的区分。如果用户关闭了APP,IOS 系统会关闭所有的background Session。而且，被用户强制关闭了以后，IOS系统不会主动唤醒APP，只有用户下次启动了APP，数据传输才会继续。

## 1.2 NSURLSessionTask
实际的Session任务，分为三种，继承关系如图 

![](https://img-blog.csdn.net/20150321075521358)

其中， 
**DataTask**－用来请求资源，然后服务器返回数据，再内存中存储为NSData格式。default,ephemeral,shared Session支持data task。background session不支持。 
**Upload Task**－和DataTask类似，只不过在请求的时候提供了request body。并且background Session支持 upload task。 
**Download Task**－下载内容到硬盘上，所有类型的Session都支持。

- DownloadTask和DataTask的区别
  简而言之，DownloadTask是把文件直接download到磁盘。 
  详细来说，有以下几点区别

- DownloadTask支持BackgroundSession，而dataTask不支持
  DownloadTask支持断点续传（下载到一半的时候暂停，重启后继续下载，前提下载的服务器支

注意，创建的task都是挂起状态，需要resume才能执行。

## 1.3 NSURLSession
会话是基于NSURLSession网络开发的核心组件。由上文的Configuration来配置，然后作为工厂，创建NSURLSessionTask来进行实际的数据传输任务。 
一个初始化的例子，
```objective-c
self.session = [NSURLSession sessionWithConfiguration:[NSURLSessionConfiguration defaultSessionConfiguration]];
```
创建一个task
```objective-c
 NSURLSessionDataTask * dataTask = [self.session dataTaskWithURL:[NSURL URLWithString:imageURL] completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {

    }];
```
开始一个task
```objective-c
    [dataTask resume];
```
## 1.4 NSURLRequest
指定请求的URL和cache策略。 
例如，如下这个初始化函数



```objective-c
/*这个是类方法的初始化方法，参数就是缓存策略和超时时间
 这里引入了这个NSURLRequestCachePolicy缓存策略的枚举类型，下面梳理这个枚举。
 typedef NS_ENUM(NSUInteger, NSURLRequestCachePolicy)
 {
         默认缓存策略
         NSURLRequestUseProtocolCachePolicy = 0,
 
         URL应该加载源端数据，不使用本地缓存数据
         NSURLRequestReloadIgnoringLocalCacheData = 1,
 
         本地缓存数据、代理和其他中介都要忽视他们的缓存，直接加载源数据
         NSURLRequestReloadIgnoringLocalAndRemoteCacheData = 4, // Unimplemented
 
         从服务端加载数据，完全忽略缓存。和NSURLRequestReloadIgnoringLocalCacheData一样
         NSURLRequestReloadIgnoringCacheData = NSURLRequestReloadIgnoringLocalCacheData,
 
         使用缓存数据，忽略其过期时间；只有在没有缓存版本的时候才从源端加载数据
         NSURLRequestReturnCacheDataElseLoad = 2,
 
         只使用cache数据，如果不存在cache，就请求失败，不再去请求数据  用于没有建立网络连接离线模式
         NSURLRequestReturnCacheDataDontLoad = 3,

         指定如果已存的缓存数据被提供它的源段确认为有效则允许使用缓存数据响应请求，否则从源段加载数据。
         NSURLRequestReloadRevalidatingCacheData = 5, // Unimplemented
 };
 
 */
+ (instancetype)requestWithURL:(NSURL *)URL cachePolicy:(NSURLRequestCachePolicy)cachePolicy timeoutInterval:(NSTimeInterval)timeoutInterval;
```

就是在初始化的时候指定url,cachePolicy以及 timeoutInterval.

通过NSURLRequest可以设置HTTPMethod，默认是GET

## 1.5 NSURLCache
cache URL请求返回的response。

>实现的方式是把NSURLRequest对象映射到NSCachedURLResponse对象。可以设置在内存中缓存的大小，以及在磁盘中缓存的大小和路径。 
不是特别需要的话，使用Shared Cached足矣，如果有特别需要，创建一个NSURLCache对象，然后通过+ setSharedURLCache 来设定。

当然，通过这个类也可以获得到当前cache的使用情况。1.6 NSURLResponse/NSHTTPURLResponse

通过REST API进行资源操作的时候，有request（请求）必然就有response（响应）。NSURLResponse中包含了metadata，例如返回的数据长度（expectedContentLength），MIME 类型，text编码方式。

**NSHTTPURLResponse**是**NSURLResponse**的子类，由于绝大部分的REST都是HTTP的，所以，通常遇到的都是NSHTTPURLResponse对象。通过这个对象可以获得：HTTP的headers，status Code等信息。 
其中：HTTP headers包含的信息较多，不懂的可以看看wiki上http headers的内容。 
status code会返回请求的状况：例如404是not found。

WWW-Authenticate: Basic realm=“nmrs_m7VKmomQ2YM3:”是Server需要Client进行HTTP BA授权。

## 1.7 NSURLCredential
－ 用来处理证书信息 
比如用户名密码，比如服务器授权等等。 
这个要根据不同的认证方式来处理， 
例如以下就是初始化一个用户名密码的认证。
```
(NSURLCredential *)credentialWithUser:(NSString *)user
```
基于证书的
```
＋credentialWithIdentity:certificates:persistence:.
```
这里的
```
typedef NS_ENUM(NSUInteger, NSURLCredentialPersistence) {
   NSURLCredentialPersistenceNone, //不存储
   NSURLCredentialPersistenceForSession,//按照Session生命周期存储
   NSURLCredentialPersistencePermanent,//存储到钥匙串
   NSURLCredentialPersistenceSynchronizable//存储到钥匙串，根据相同的AppleID分配到其他设备。
};
```
## 1.8 NSURLAuthenticationChallenge
在访问资源的时候，可能服务器会返回需要授权（提供一个NSURLCredential对象）。那么，URLSession:task:didReceiveChallenge:completionHandler:被调用。需要的授权信息会保存在这个类的对象里。 
几个常用的属性 
error 
最后一次授权失败的错误信息 
failureResponse 
最后一次授权失败的错误信息 
previousFailureCount 
授权失败的次数 
proposedCredential 
建议使用的证书 
protectionSpace 
NSURLProtectionSpace对象，包括了地址端口等信息，接下来会讲解这个对象。

## 1.9 NSURLProtectionSpace
这个类的对象代表了服务器上的一块需要授权信息的区域，英文叫realm。通过这个对象的信息来响应Challenge。 
比如，如果服务器需要一个基于用户名密码的认证，那么应该先参考下NSURLProtectionSpace对象的host,port,realm，protocol等信息，然后依照这个信息提供证书。

# 三 代理delegate
NSURLSession的代理通常是两个层次的，Session层次和Task层次（一个Session可以包括多个Task）。 
**NSURLSessionDelegate**－处理Session层次事件 
**NSURLSessionTaskDelegate**－处理所有类型Task层次共性事件 
**NSURLSessionDownloadDelegate**－处理Download类型的Task层次事件**NSURLSessionDataDelegate**－处理Download类型的Task层次事件

![img](https://ask.qcloudimg.com/http-save/yehe-1138834/8pu1mnl5ed.png?imageView2/2/w/1620)
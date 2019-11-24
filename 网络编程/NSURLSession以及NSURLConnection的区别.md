1. 下载任务时：
  NSURLConnection会先放在内存、最后写入沙盒。可能引起内存暴涨。
  NSURLSession会直接写在沙盒和tem文件夹中、最后需要手动转移。

2. 请求控制：

  NSURLConnection实例化对象，实例化开始，默认请求就发送（同步发送），不需要调用start方法。而cancel 可以停止请求的发送，停止后不能继续访问，需要创建新的请求。

  NSURLSession有三个控制方法，取消（cancel），暂停（suspend），继续（resume），暂停后可以通过继续恢复当前的请求任务。

3. 断点续传：

  NSURLConnection进行断点下载，通过设置访问请求的HTTPHeaderField的Range属性，开启运行循环，NSURLConnection的代理方法作为运行循环的事件源，接收到下载数据时代理方法就会持续调用，并使用NSOutputStream管道流进行数据保存。

  NSURLSession进行断点下载，当暂停下载任务后，如果 downloadTask （下载任务）为非空，调用 cancelByProducingResumeData:(void (^)(NSData *resumeData))completionHandler 这个方法，这个方法接收一个参数，完成处理代码块，这个代码块有一个 NSData 参数 resumeData，如果 resumeData 非空，我们就保存这个对象到视图控制器的 resumeData 属性中。在点击再次下载时，通过调用 [ [self.session downloadTaskWithResumeData:self.resumeData]resume]方法进行继续下载操作。

  经过以上比较可以发现，使用NSURLSession进行断点下载更加便捷。

4. 配置信息：
  NSURLConnection只能全局配置。
  NSURLSession每一个实例都可以通过NSURLSessionConfiguration进行配置。

[**IOS网络开发NSURLSession详解（一）概述**](https://blog.csdn.net/hello_hwc/article/details/44513699)

[NSURLSession 所有的都在这里(一)](https://cloud.tencent.com/developer/article/1137486)

[iOS基础深入补完计划--网络模块NSURLSession概述](https://www.jianshu.com/p/16ed20b0e7b8)

[NSURLConnection与NSURLSession的区别](https://www.jianshu.com/p/a15f70c3c934)
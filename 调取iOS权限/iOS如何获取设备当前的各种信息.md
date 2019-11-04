[TOC]
## 前言

- 下面各种获取到的信息分为了两大类，一种是获取APP载体设备当前的各种信息，另一种是APP自身能取得的权限，两部分中的“说在前面的”只是记录了一个我的学习过程，所有调取代码都在示例代码或者demo里，可跳过这部分自取代码

- 这是我的demo地址 [GitHub](https://github.com/PaulZhe/iOS-device-message)，我将其封装成了一个工具类，欢迎clone使用，其中蓝牙功能没有封装进去，下面我再说具体原因
## 获取设备当前的各种信息

- 说在前面的：

  1. 目录前面11条的信息很好获取，好多都是一行代码的事，参考博客:

     1. [iOS 如何获取设备的各种信息](http://www.cocoachina.com/articles/20149) 

     	2. [iOS怎么判断设备的WiFi开关是否打开？](https://segmentfault.com/q/1010000003901530/a-1020000003904523)
      	3. 设备像素比是什么：[Android dp 和 CSS px](https://www.jianshu.com/p/52af18472472)

  2. 到判断蓝牙和定位开关状态就要导入苹果官方提供的框架了，至于如何导入框架：在工程文件General -> Linked Frameworks and Libraries中点加号添加即可

     ![](https://img-blog.csdn.net/20160402165316572)

     	1. 定位参考博客：[iOS 判断APP是否打开定位，并实现直接跳转打开定位](https://blog.csdn.net/wangqinglei0307/article/details/78672689)
     
      	2. 蓝牙参考博客：[iOS中蓝牙开发](https://www.jianshu.com/p/e7df216f9e5e) 核心就是协议方法```-(void)centralManagerDidUpdateState:(CBCentralManager *)central```的回调和参数 central.state 的几种枚举值状态，判断一下就好，因为要遵循主设备的委托——CBCentralManagerDelegate，所以没有将其封装在工具类中。

- 目录：

1. 获取iPhone名称
2. 获取设备版本号
3. 当前系统版本号
4. 屏幕宽度
5. 屏幕高度
6. 获取电池电量
7. 获取当前设备IP
8. 获取总内存大小
9. 获取当前可用内存
10. 获取当前语言
11. 获取Wi-Fi开关状态
12. 获取GPS开关状态
13. 获取当前设备像素比
14. 获取蓝牙开关状态

```objective-c
/// 获取iPhone名称
+ (NSString *)getiPhoneName {
    return [UIDevice currentDevice].name;
}

///获取设备版本号
+ (NSString *)getDeviceName {
    // 需要#import "sys/utsname.h"
    struct utsname systemInfo;
    uname(&systemInfo);
    NSString *deviceString = [NSString stringWithCString:systemInfo.machine encoding:NSUTF8StringEncoding];
    if ([deviceString isEqualToString:@"iPhone3,1"])    return @"iPhone 4";
    if ([deviceString isEqualToString:@"iPhone3,2"])    return @"iPhone 4";
    if ([deviceString isEqualToString:@"iPhone3,3"])    return @"iPhone 4";
    if ([deviceString isEqualToString:@"iPhone4,1"])    return @"iPhone 4S";
    if ([deviceString isEqualToString:@"iPhone5,1"])    return @"iPhone 5";
    if ([deviceString isEqualToString:@"iPhone5,2"])    return @"iPhone 5 (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPhone5,3"])    return @"iPhone 5c (GSM)";
    if ([deviceString isEqualToString:@"iPhone5,4"])    return @"iPhone 5c (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPhone6,1"])    return @"iPhone 5s (GSM)";
    if ([deviceString isEqualToString:@"iPhone6,2"])    return @"iPhone 5s (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPhone7,1"])    return @"iPhone 6 Plus";
    if ([deviceString isEqualToString:@"iPhone7,2"])    return @"iPhone 6";
    if ([deviceString isEqualToString:@"iPhone8,1"])    return @"iPhone 6s";
    if ([deviceString isEqualToString:@"iPhone8,2"])    return @"iPhone 6s Plus";
    if ([deviceString isEqualToString:@"iPhone8,4"])    return @"iPhone SE";
    if ([deviceString isEqualToString:@"iPhone9,1"])    return @"iPhone 7";
    if ([deviceString isEqualToString:@"iPhone9,2"])    return @"iPhone 7 Plus";
    if ([deviceString isEqualToString:@"iPhone9,3"])    return @"iPhone 7";
    if ([deviceString isEqualToString:@"iPhone9,4"])    return @"iPhone 7 Plus";
    if ([deviceString isEqualToString:@"iPhone10,1"])    return @"iPhone 8";
    if ([deviceString isEqualToString:@"iPhone10,2"])    return @"iPhone 8 Plus";
    if ([deviceString isEqualToString:@"iPhone10,3"])    return @"iPhone X";
    if ([deviceString isEqualToString:@"iPhone10,4"])    return @"iPhone 8";
    if ([deviceString isEqualToString:@"iPhone10,5"])    return @"iPhone 8 Plus";
    if ([deviceString isEqualToString:@"iPhone10,6"])    return @"iPhone X";
    if ([deviceString isEqualToString:@"iPhone11,2"])    return @"iPhone XS";
    if ([deviceString isEqualToString:@"iPhone11,4"])    return @"iPhone XS Max";
    if ([deviceString isEqualToString:@"iPhone11,6"])    return @"iPhone XS Max";
    if ([deviceString isEqualToString:@"iPhone11,8"])    return @"iPhone XR";
    if ([deviceString isEqualToString:@"iPod1,1"])      return @"iPod Touch 1G";
    if ([deviceString isEqualToString:@"iPod2,1"])      return @"iPod Touch 2G";
    if ([deviceString isEqualToString:@"iPod3,1"])      return @"iPod Touch 3G";
    if ([deviceString isEqualToString:@"iPod4,1"])      return @"iPod Touch 4G";
    if ([deviceString isEqualToString:@"iPod5,1"])      return @"iPod Touch (5 Gen)";
    if ([deviceString isEqualToString:@"iPad1,1"])      return @"iPad";
    if ([deviceString isEqualToString:@"iPad1,2"])      return @"iPad 3G";
    if ([deviceString isEqualToString:@"iPad2,1"])      return @"iPad 2 (WiFi)";
    if ([deviceString isEqualToString:@"iPad2,2"])      return @"iPad 2";
    if ([deviceString isEqualToString:@"iPad2,3"])      return @"iPad 2 (CDMA)";
    if ([deviceString isEqualToString:@"iPad2,4"])      return @"iPad 2";
    if ([deviceString isEqualToString:@"iPad2,5"])      return @"iPad Mini (WiFi)";
    if ([deviceString isEqualToString:@"iPad2,6"])      return @"iPad Mini";
    if ([deviceString isEqualToString:@"iPad2,7"])      return @"iPad Mini (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPad3,1"])      return @"iPad 3 (WiFi)";
    if ([deviceString isEqualToString:@"iPad3,2"])      return @"iPad 3 (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPad3,3"])      return @"iPad 3";
    if ([deviceString isEqualToString:@"iPad3,4"])      return @"iPad 4 (WiFi)";
    if ([deviceString isEqualToString:@"iPad3,5"])      return @"iPad 4";
    if ([deviceString isEqualToString:@"iPad3,6"])      return @"iPad 4 (GSM+CDMA)";
    if ([deviceString isEqualToString:@"iPad4,1"])      return @"iPad Air (WiFi)";
    if ([deviceString isEqualToString:@"iPad4,2"])      return @"iPad Air (Cellular)";
    if ([deviceString isEqualToString:@"iPad4,4"])      return @"iPad Mini 2 (WiFi)";
    if ([deviceString isEqualToString:@"iPad4,5"])      return @"iPad Mini 2 (Cellular)";
    if ([deviceString isEqualToString:@"iPad4,6"])      return @"iPad Mini 2";
    if ([deviceString isEqualToString:@"iPad4,7"])      return @"iPad Mini 3";
    if ([deviceString isEqualToString:@"iPad4,8"])      return @"iPad Mini 3";
    if ([deviceString isEqualToString:@"iPad4,9"])      return @"iPad Mini 3";
    if ([deviceString isEqualToString:@"iPad5,1"])      return @"iPad Mini 4 (WiFi)";
    if ([deviceString isEqualToString:@"iPad5,2"])      return @"iPad Mini 4 (LTE)";
    if ([deviceString isEqualToString:@"iPad5,3"])      return @"iPad Air 2";
    if ([deviceString isEqualToString:@"iPad5,4"])      return @"iPad Air 2";
    if ([deviceString isEqualToString:@"iPad6,3"])      return @"iPad Pro 9.7";
    if ([deviceString isEqualToString:@"iPad6,4"])      return @"iPad Pro 9.7";
    if ([deviceString isEqualToString:@"iPad6,7"])      return @"iPad Pro 12.9";
    if ([deviceString isEqualToString:@"iPad6,8"])      return @"iPad Pro 12.9";
    if ([deviceString isEqualToString:@"i386"])         return @"Simulator";
    if ([deviceString isEqualToString:@"x86_64"])       return @"Simulator";
    return deviceString;
}

/// 当前系统版本号
+ (NSString *)getSystemVersion {
    return [UIDevice currentDevice].systemVersion;
}

/// 屏幕宽度
+ (CGFloat)getDeviceScreenWidth {
    return [UIScreen mainScreen].bounds.size.width;
}

/// 屏幕高度
+ (CGFloat)getDeviceScreenHeight {
    return [UIScreen mainScreen].bounds.size.height;
}

/// 获取电池电量
+ (CGFloat)getBatteryLevel {
    return [UIDevice currentDevice].batteryLevel;
}

/// 获取当前设备IP
+ (NSString *)getDeviceIPAdress {
    NSString *address = @"an error occurred when obtaining ip address";
    struct ifaddrs *interfaces = NULL;
    struct ifaddrs *temp_addr = NULL;
    int success = 0;
    success = getifaddrs(&interfaces);
    if (success == 0) { // 0 表示获取成功
        temp_addr = interfaces;
        while (temp_addr != NULL) {
            if( temp_addr->ifa_addr->sa_family == AF_INET) {
                // Check if interface is en0 which is the wifi connection on the iPhone
                if ([[NSString stringWithUTF8String:temp_addr->ifa_name] isEqualToString:@"en0"]) {
                    // Get NSString from C String
                    address = [NSString stringWithUTF8String:inet_ntoa(((struct sockaddr_in *)temp_addr->ifa_addr)->sin_addr)];
                }
            }
            temp_addr = temp_addr->ifa_next;
        }
    }
    freeifaddrs(interfaces);
    return address;
}

/// 获取总内存大小
+ (long long)getTotalMemorySize {
    return [NSProcessInfo processInfo].physicalMemory;
}

/// 获取当前可用内存
+ (long long)getAvailableMemorySize {
    vm_statistics_data_t vmStats;
    mach_msg_type_number_t infoCount = HOST_VM_INFO_COUNT;
    kern_return_t kernReturn = host_statistics(mach_host_self(), HOST_VM_INFO, (host_info_t)&vmStats, &infoCount);
    if (kernReturn != KERN_SUCCESS)
    {
        return NSNotFound;
    }
    return ((vm_page_size * vmStats.free_count + vm_page_size * vmStats.inactive_count));
}

/// 获取当前语言
+ (NSString *)getDeviceLanguage {
    NSArray *languageArray = [NSLocale preferredLanguages];
    return [languageArray objectAtIndex:0];
}

///获取Wi-Fi开关状态
+ (BOOL)getWiFiEnabled {
    NSCountedSet * cset = [NSCountedSet new];
    struct ifaddrs *interfaces;
    if( ! getifaddrs(&interfaces) ) {
        for( struct ifaddrs *interface = interfaces; interface; interface = interface->ifa_next) {
            if ( (interface->ifa_flags & IFF_UP) == IFF_UP ) {
                [cset addObject:[NSString stringWithUTF8String:interface->ifa_name]];
            }
        }
    }
    return [cset countForObject:@"awdl0"] > 1 ? YES : NO;
}

///获取GPS开关状态
+ (BOOL)getGPSEnabled {
    if ([CLLocationManager locationServicesEnabled] && ([CLLocationManager authorizationStatus] ==kCLAuthorizationStatusAuthorizedWhenInUse || [CLLocationManager authorizationStatus] ==kCLAuthorizationStatusNotDetermined || [CLLocationManager authorizationStatus] == kCLAuthorizationStatusAuthorizedAlways)) {
            return YES;
        } else if ([CLLocationManager authorizationStatus] ==kCLAuthorizationStatusDenied) {
            return NO;
        } else {
            return NO;
        }
}

///获取当前设备像素比
+ (int)getPixelScale {
    struct utsname systemInfo;
    uname(&systemInfo);
    NSString *deviceString = [NSString stringWithCString:systemInfo.machine encoding:NSUTF8StringEncoding];
    if ([deviceString isEqualToString:@"iPhone3,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone3,2"])    return 2;
    if ([deviceString isEqualToString:@"iPhone3,3"])    return 2;
    if ([deviceString isEqualToString:@"iPhone4,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone5,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone5,2"])    return 2;
    if ([deviceString isEqualToString:@"iPhone5,3"])    return 2;
    if ([deviceString isEqualToString:@"iPhone5,4"])    return 2;
    if ([deviceString isEqualToString:@"iPhone6,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone6,2"])    return 2;
    if ([deviceString isEqualToString:@"iPhone7,1"])    return 3;
    if ([deviceString isEqualToString:@"iPhone7,2"])    return 2;
    if ([deviceString isEqualToString:@"iPhone8,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone8,2"])    return 3;
    if ([deviceString isEqualToString:@"iPhone8,4"])    return 2;
    if ([deviceString isEqualToString:@"iPhone9,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone9,2"])    return 3;
    if ([deviceString isEqualToString:@"iPhone9,3"])    return 2;
    if ([deviceString isEqualToString:@"iPhone9,4"])    return 3;
    if ([deviceString isEqualToString:@"iPhone10,1"])    return 2;
    if ([deviceString isEqualToString:@"iPhone10,2"])    return 3;
    if ([deviceString isEqualToString:@"iPhone10,3"])    return 3;
    if ([deviceString isEqualToString:@"iPhone10,4"])    return 2;
    if ([deviceString isEqualToString:@"iPhone10,5"])    return 3;
    if ([deviceString isEqualToString:@"iPhone10,6"])    return 2;
    if ([deviceString isEqualToString:@"iPhone11,2"])    return 3;
    if ([deviceString isEqualToString:@"iPhone11,4"])    return 3;
    if ([deviceString isEqualToString:@"iPhone11,6"])    return 3;
    if ([deviceString isEqualToString:@"iPhone11,8"])    return 2;
    if ([deviceString isEqualToString:@"iPod1,1"])      return 1;
    if ([deviceString isEqualToString:@"iPod2,1"])      return 1;
    if ([deviceString isEqualToString:@"iPod3,1"])      return 1;
    if ([deviceString isEqualToString:@"iPod4,1"])      return 2;
    if ([deviceString isEqualToString:@"iPod5,1"])      return 2;
    if ([deviceString isEqualToString:@"iPad1,1"])      return 1;
    if ([deviceString isEqualToString:@"iPad1,2"])      return 2;
    if ([deviceString isEqualToString:@"iPad2,1"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,2"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,3"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,4"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,5"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,6"])      return 1;
    if ([deviceString isEqualToString:@"iPad2,7"])      return 1;
    if ([deviceString isEqualToString:@"iPad3,1"])      return 2;
    if ([deviceString isEqualToString:@"iPad3,2"])      return 2;
    if ([deviceString isEqualToString:@"iPad3,3"])      return 2;
    if ([deviceString isEqualToString:@"iPad3,4"])      return 2;
    if ([deviceString isEqualToString:@"iPad3,5"])      return 2;
    if ([deviceString isEqualToString:@"iPad3,6"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,1"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,2"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,4"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,5"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,6"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,7"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,8"])      return 2;
    if ([deviceString isEqualToString:@"iPad4,9"])      return 2;
    if ([deviceString isEqualToString:@"iPad5,1"])      return 2;
    if ([deviceString isEqualToString:@"iPad5,2"])      return 2;
    if ([deviceString isEqualToString:@"iPad5,3"])      return 2;
    if ([deviceString isEqualToString:@"iPad5,4"])      return 2;
    if ([deviceString isEqualToString:@"iPad6,3"])      return 3;
    if ([deviceString isEqualToString:@"iPad6,4"])      return 3;
    if ([deviceString isEqualToString:@"iPad6,7"])      return 3;
    if ([deviceString isEqualToString:@"iPad6,8"])      return 3;
    return 0;
}

```
## 获取APP能获取到的权限信息(内含plist文件的读写)

- 说在前面的

   1. 以下获取权限信息分为两类，一种是工程.plist配置文件中的权限，例如目录的1～4；还有一种是用户应用层的权限，比如目录5，6的通知权限

   2. 要获取工程.plist配置文件中的权限，我们就要学习一下plist文件的读写：

       - plist文件，如字面意思即plist后缀类型的文件，这个文件 是以 key value 存放的键 值对的值。它全名是：Property List，属性列表文件，它是一种用来存储串行化后的对象的文件。属性列表文件的扩展名为.plist ，因此通常被称为 plist文件。plist文件是标准的xml格式的。  我们在日常开发中 可以用它 来存储一些系统的的用户信息，系统的配置信息等。

       - 然后我们理解一下NSBundle（束）的概念，这里可以参考一下博客 [iOS：NSBundle的一些理解](https://www.jianshu.com/p/b64ff9d8e7ce) ,对这个概念我现在理解的也比较肤浅，仅限于使用阶段，有空会再次学习。

       - 然后就是plist文件的读写，参考博客： [plist 文件读写](https://www.cnblogs.com/gaohe/p/4549789.html) 。先获取主束（包含了程序会使用到的资源），然后读取plist文件的路径，将数据读取存放到一个字典中，判断权限时搜索字典有没有所需权限的key值即可判断权限有无。这里给出一些常用权限设置方法 [iOS10适配——相机，通讯录，麦克风等权限设置](https://www.cnblogs.com/liyingnan/p/5956571.html) , [iOS - iOS8常用权限请求及设置逻辑总汇](https://blog.csdn.net/codermy/article/details/53161076) 

       - 注意因为plist文件是xml格式的，所以这些key值具体内容不是 proper list 格式打开显示的那些权限关键字，要以源码格式打开plist文件才能看到其真正的权限key值

         ![](http://ww3.sinaimg.cn/large/006tNc79ly1g5a39oip90j30fx0d8k1x.jpg)

  3. 推送权限判断参考博客（等系统学习后再另开一篇博客介绍）：
     	- [iOS推送权限开发判断](https://www.jianshu.com/p/bdc64eb29908)
        	- [UNNotificationSettings属性详解（准确的判断用户是否打开了推送开关）](https://www.jianshu.com/p/61dd9dd431a9)
        	- [活久见的重构 - iOS 10 UserNotifications 框架解析](https://onevcat.com/2016/08/notification/)

- 目录

1. 获取app配置文件中是否允许使用相册权限
2. 获取app配置文件中是否允许使用摄像头权限
3. 获取app配置文件中是否允许使用定位权限
4. 获取app配置文件中是否允许使用麦克风权限
5. 申请通知权限
6. 查看当前通知权限
```objective-c
//获取app配置文件中是否允许使用相册权限
+ (BOOL)getPhotoLibrary {
    NSString *path = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
    NSDictionary *dic = [[NSDictionary alloc] initWithContentsOfFile:path];
    if ([dic objectForKey:@"NSPhotoLibraryUsageDescription"]) {
        return YES;
    } else {
        return NO;
    }
}
//获取app配置文件中是否允许使用摄像头权限
+ (BOOL)getCamera {
    NSString *path = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
    NSDictionary *dic = [[NSDictionary alloc] initWithContentsOfFile:path];
    if ([dic objectForKey:@"NSCameraUsageDescription"]) {
        return YES;
    } else {
        return NO;
    }
}
//获取app配置文件中是否允许使用定位权限
+ (BOOL)getLocation {
    NSString *path = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
    NSDictionary *dic = [[NSDictionary alloc] initWithContentsOfFile:path];
    if ([dic objectForKey:@"NSLocationWhenInUseUsageDescription"]) {
        return YES;
    } else {
        return NO;
    }
}
//获取app配置文件中是否允许使用麦克风权限
+ (BOOL)getMicrophone {
    NSString *path = [[NSBundle mainBundle] pathForResource:@"Info" ofType:@"plist"];
    NSDictionary *dic = [[NSDictionary alloc] initWithContentsOfFile:path];
    if ([dic objectForKey:@"NSMicrophoneUsageDescription"]) {
        return YES;
    } else {
        return NO;
    }
}

//申请通知权限
- (void)requestNotification
{
    if (@available(iOS 10, *)) {
        UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
        center.delegate = self;
        [center requestAuthorizationWithOptions:UNAuthorizationOptionAlert | UNAuthorizationOptionBadge | UNAuthorizationOptionSound completionHandler:^(BOOL granted, NSError * _Nullable error) {
            if (granted) {
                // 允许推送
                NSLog(@"允许打开通知权限");
            }else{
                //不允许
                NSLog(@"不允许打开通知权限");
            }
            
        }];
    } else if(@available(iOS 8 , *)) {
        UIApplication * application = [UIApplication sharedApplication];
        
        [application registerUserNotificationSettings:[UIUserNotificationSettings settingsForTypes:UIUserNotificationTypeAlert | UIUserNotificationTypeBadge | UIUserNotificationTypeSound categories:nil]];
        [application registerForRemoteNotifications];
    } else {
        UIApplication * application = [UIApplication sharedApplication];
        [application registerForRemoteNotificationTypes:UIRemoteNotificationTypeAlert | UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound];
        [application registerForRemoteNotifications];
    }
}

//查看当前通知权限
- (void)checkCurrentNotificationStatus {
    if (@available(iOS 10 , *)) {
        [[UNUserNotificationCenter currentNotificationCenter] getNotificationSettingsWithCompletionHandler:^(UNNotificationSettings * _Nonnull settings) {
            if (settings.authorizationStatus == UNAuthorizationStatusAuthorized) {
                // 没权限
                NSLog(@"有通知权限");
            } else {
                NSLog(@"没通知权限");
            }
        }];
    }
    else if (@available(iOS 8 , *)) {
        UIUserNotificationSettings * setting = [[UIApplication sharedApplication] currentUserNotificationSettings];
        if (setting.types == UIUserNotificationTypeNone) {
            // 没权限
        }
    } else {
        UIRemoteNotificationType type = [[UIApplication sharedApplication] enabledRemoteNotificationTypes];
        if (type == UIUserNotificationTypeNone) {
            // 没权限
        }
    }
}
```

## 2019.11.04 更新

在苹果更新 iOS 13以后，原来的蓝牙权限在IOS 13被遗弃，改成了 `NSBluetoothAlwaysUsageDescription`

在info.plist里面添加

NSBluetoothAlwaysUsageDescription 或者 Privacy - Bluetooth Peripheral Usage Description 权限字段  value 也是你的蓝牙用途

![](https://upload-images.jianshu.io/upload_images/5252634-0d2f523e7872c5eb.png)

[NSBluetoothAlwaysUsageDescription苹果字段说明文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fbundleresources%2Finformation_property_list%2Fnsbluetoothalwaysusagedescription)
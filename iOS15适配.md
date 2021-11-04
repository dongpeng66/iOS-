# 一、 解决 iOS 15 的 ATT 授权弹窗不显示问题

拒绝理由：
Guideline 2.1 - Information Needed
We're looking forward to completing our review, but we need more information to continue. Your app uses the AppTrackingTransparency framework, but we are unable tolocate the App Tracking Transparency permission request when reviewed on iOS 15.0.

看到这个 **on iOS 15.0** 就知道苹果又要整什么幺蛾子了，明明 iOS14 都没问题的。

Google 中...

找着找着找到了官方文档...

**官方解释：**

![](https://github.com/dongpeng66/iOS-/blob/main/images/1.png)

大致意思：调用这个API requestTrackingAuthorization 必须是在 App 在前台活跃的前提下。
解决办法：

**分析：**

对于这种权限请求，通常情况下是写在了 AppDelegate 的 didFinishLaunchingWithOptions中，但明显已经不符合官方的要求了，App 还没有到 Active 的状态。

**解决方式：**

**AppDelegate ：**



```
- (void)applicationDidBecomeActive:(UIApplication *)application {


}
```

用到了 SceneDelegate的，那么iOS 13 之后 AppDelegate是不执行了的， 所以改成在 SceneDelegate 的 sceneDidBecomeActive 方法中做权限请求。同时增加了 1 秒的延迟，避免和其他的权限弹窗冲突不显示的问题。


```
- (void)sceneDidBecomeActive:(UIScene *)scene {


}

```

如果弹窗还是出不来，建议加上一到两秒的延迟，再去请求权限，保证审核通过。

# 二、NavigationBar颜色失效


```
self.navigationController.navigationBar.barTintColor =[UIColor redColor];

```

多么正常的代码，多么符合常理的操作，但是在iOS15的页面失效，不起作用；

无奈查找无果，便前去苹果论坛[寻找答案](https://developer.apple.com/forums/thread/683265)，果然功夫不负有心人让我找到了。


```
As of iOS 15, UINavigationBar, UIToolbar, and UITabBar will use their scrollEdgeAppearance when your view controller's associated scroll view is at the appropriate edge (or always if you don't have a UIScrollView in your hierarchy, more on that below).

```

从 iOS 15 开始，UINavigationBar、UIToolbar 和 UITabBar 将在您的视图控制器的关联滚动视图位于适当的边缘时使用它们的 scrollEdgeAppearance（或者总是如果您的层次结构中没有 UIScrollView，更多内容见下文）。



```
You must adopt the UIBarAppearance APIs (available since iOS 13, specializations for each bar type) to customize this behavior. UIToolbar and UITabBar add scrollEdgeAppearance properties for this purpose in iOS 15.

```

您必须采用 UIBarAppearance API（自 iOS 13 起可用，针对每种条形类型进行了专门化）来自定义此行为。 UIToolbar 和 UITabBar 为此在 iOS 15 中添加了 scrollEdgeAppearance 属性。
so，代码更改如下：



```
 if (@available(iOS 15.0, *)) {
        UINavigationBarAppearance *navigationBarAppearance = [UINavigationBarAppearance new];
        navigationBarAppearance.backgroundColor = [UIColor redColor];
        self.navigationController.navigationBar.scrollEdgeAppearance = navigationBarAppearance;
        self.navigationController.navigationBar.standardAppearance = navigationBarAppearance;
    }


```

注意：

在使用UINavigationBarAppearance进行导航相关设置的时候会与原本的系统API冲突；小弟遇到了这样的情况：

1：首先正常使用原本的系统API设置导航的文字颜色


```
[self.navigationController.navigationBar setTitleTextAttributes:@{NSForegroundColorAttributeName:[UIColor whiteColor],NSFontAttributeName:[UIFont systemFontOfSize:15 weight:UIFontWeightRegular]}];


```

2：再使用UINavigationBarAppearance设置导航的背景颜色，这时会发现原本设置的字体颜色变成了系统的默认黑色。所以，当需要同时修改导航的背景颜色和字体颜色时，要使用UINavigationBarAppearance同时设置；


```
   if (@available(iOS 15.0, *)) {
            UINavigationBarAppearance *navigationBarAppearance = [UINavigationBarAppearance new];
            //设置导航背景颜色
            navigationBarAppearance.backgroundColor = [UIColor blackColor];
            //设置导航字体颜色
            navigationBarAppearance.titleTextAttributes = @{NSForegroundColorAttributeName:[UIColor whiteColor],NSFontAttributeName:[UIFont systemFontOfSize:15 weight:UIFontWeightRegular]};
            self.navigationController.navigationBar.scrollEdgeAppearance = navigationBarAppearance;
            self.navigationController.navigationBar.standardAppearance = navigationBarAppearance;
        }

```

# 三、UITableView section增加默认高度


```
/// Padding above each section header. The default value is `UITableViewAutomaticDimension`.
@property (nonatomic) CGFloat sectionHeaderTopPadding API_AVAILABLE(ios(15.0), tvos(15.0), watchos(8.0));


```

UITableView又新增了一个新属性：sectionHeaderTopPadding 会给每一个section header 增加一个默认高度，当我们使用UITableViewStylePlain 初始化tableView的时候，就会发现系统默认给section header增高了22像素。
so, 代码更改如下：


```

if (@available(iOS 15.0, *)) {
    _tableView.sectionHeaderTopPadding = 0;
}

```
最好改在基类，一次解决所有问题；

如果你觉得这个也比较麻烦，那么还有一种办法：


```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

    //iOS 15 的 UITableView 新增了一条新属性：sectionHeaderTopPadding， 默认会给每一个 section header 增加一个高度，当我们使用 UITableViewStylePlain 初始化 UITableView 的时候，能发现 sectionHeader 增高了 22px。。
    if(@available(iOS 15.0, *)){
        UITableView.appearance.sectionHeaderTopPadding=0;
    }

    return YES;
}

```

# 四、 NavigationBar 颜色及背景失效


1.问题描述

项目中往往会自定义一个导航控制器，方便全局指定导航条的背景色、标题颜色等等。以设置背景色和标题颜色为例：



```
//背景色
self.navigationBar.barTintColor = RGB(42, 109, 240);
//Title 颜色
NSDictionary *titleTextAttributes = @{NSFontAttributeName:[UIFont fontWithName:@"" size:18], NSForegroundColorAttributeName:RGB(255, 255, 255)};
[self.navigationBar setTitleTextAttributes:titleTextAttributes];


```

但在 iOS 15上发现，指定的背景色失效了，但滚动控制器的视图时，导航条的背景又出现了。看了一眼 UINavigationBar 的 API，15中并没有新增的。倒是有几个 iOS 13新增的 API 我没用过……哈哈哈，写到这里觉得自己以前的功课落下太多了，13的更新还没学习呢😂😂😂😂😂

2.iOS 13新增 API

**standardAppearance : 描述导航栏以标准高度显示时要使用的外观属性。**


```
@property (nonatomic, readwrite, copy) UINavigationBarAppearance *standardAppearance;

```

**compactAppearance : 描述导航栏在紧凑高度时使用的外观属性。如果未设置，则将使用标准外观。**

```
@property (nonatomic, readwrite, copy, nullable) UINavigationBarAppearance *compactAppearance;

```

**scrollEdgeAppearance : 描述当关联的 UIScrollView 向上滚动时要使用的导航栏的外观属性。如果未设置，将改用修改后的standardAppearance。**

```
@property (nonatomic, readwrite, copy, nullable) UINavigationBarAppearance *scrollEdgeAppearance;

```
**compactScrollEdgeAppearance : 描述当导航栏以紧凑的高度显示时，以及关联的 UIScrollView 往上滚动时，要使用的导航栏的外观属性。如果未设置，则首先尝试 scrollEdgeAppearance，如果为nil，则尝试 compactAppearance，然后尝试修改 standardAppearance。**

```
@property(nonatomic,readwrite, copy, nullable) UINavigationBarAppearance *compactScrollEdgeAppearance;

```

3.解决办法

**根据我们的问题现象，猜测是 standardAppearance 和 scrollEdgeAppearance 需要调整，如果正常状态和滚动状态颜色一样，可以修改如下**


```
NSDictionary *titleTextAttributes = @{NSFontAttributeName:[UIFont fontWithName:MAIN_FONT_FAMILY size:18], NSForegroundColorAttributeName:RGB(255, 255, 255)};
if (@available(iOS 13.0, *)) {
    UINavigationBarAppearance *appearance = [UINavigationBarAppearance new];
    appearance.backgroundColor = RGB(42, 109, 240);
    appearance.titleTextAttributes = titleTextAttributes;    
    self.navigationBar.standardAppearance = appearance;
    self.navigationBar.scrollEdgeAppearance = appearance;
} else {
    // Fallback on earlier versions
    self.navigationBar.barTintColor = RGB(42, 109, 240);
    [self.navigationBar setTitleTextAttributes:titleTextAttributes];
}

```

Bingo！颜色显示正常啦

所以这是什么意思？强买强卖吗？必须设置 Appearance 才可以？

4.遗留问题

在 Xcode 13-beta 中，必须同时指定 standardAppearance 和 scrollEdgeAppearance 才可以。但根据苹果的注释，如果 scrollEdgeAppearance 为nil，会默认使用 standardAppearance 啊。燃鹅并不行。不知道是苹果的 bug 还是怎么的……朋友昨天叫我一起转行了，因为他觉得苹果的系统做的一年不如一年~ 😂

# 五、UITabBar 背景图失效

这个问题有点类似上一个，UITabBar 之前设置的背景图片，老版本可以，iOS 15上表现为空白。参考问题二的思路，找到了下面的 API，做个兼容就可以了。当然，遗留问题同上，必须同时指定 standardAppearance 和 scrollEdgeAppearance 才可以……🙄……而且，如果在初始化以后，某个时机单独修改了 standardAppearance，也必须要同步指定一下 scrollEdgeAppearance ……🙄

**API**

```
@property (nonatomic, readwrite, copy) UITabBarAppearance *standardAppearance;//ios 13.0.
@property (nonatomic, readwrite, copy, nullable) UITabBarAppearance *scrollEdgeAppearance;//ios 15.0.

```
**老方式**

```
[self.tabBar setBackgroundImage:[img imageWithRenderingMode:(UIImageRenderingModeAlwaysOriginal)]];

```

**兼容新的API**

```
UIImage *img = [UIImage imageNamed:@"ahaaaaa"];
if (@available(iOS 13.0, *)) {
    UITabBarAppearance *appearance = [[UITabBarAppearance alloc] init];
    appearance.backgroundImage = img;
    appearance.backgroundImageContentMode = UIViewContentModeScaleToFill;
    self.tabBar.standardAppearance = appearance;
    if (@available(iOS 15.0, *)) {
        self.tabBar.scrollEdgeAppearance = appearance;
    } else {
        // Fallback on earlier versions
    }
} else {
    // Fallback on earlier versions
    [self.tabBar setBackgroundImage:[img imageWithRenderingMode:(UIImageRenderingModeAlwaysOriginal)]];
}


```

# 六、UITabBarItem 文字颜色失效
……还是同上，新版本中 UITabBarItem 文字颜色的修改不起作用。同样是在 iOS 13 中新增的 UITabBarItemAppearance 来修改 Item 的不同状态下的不同表现。遗留问题同上。

相关的类型如下，其他的API就不贴了：
UITabBarItemAppearance 、 UITabBarItemStateAppearance

```
/// The appearance when the tab bar item is in the normal state
@property (nonatomic, readonly, strong) UITabBarItemStateAppearance *normal;

/// The appearance when the tab bar item is in the selected state
@property (nonatomic, readonly, strong) UITabBarItemStateAppearance *selected;

/// The appearance when the tab bar item is in the disabled state
@property (nonatomic, readonly, strong) UITabBarItemStateAppearance *disabled;

/// The appearance when the tab bar item is in the focused state
@property (nonatomic, readonly, strong) UITabBarItemStateAppearance *focused;

```
**兼容新的API**

```
//Set tabBar style.
UIColor *normalTitleColor = RGBA(80, 80, 81, 1);
UIColor *selectedTitleColor = RGBA(42, 109, 240, 1);
if (@available(iOS 13.0, *)) {
    UITabBarItemAppearance *itemAppearance = [[UITabBarItemAppearance alloc] init];
    itemAppearance.normal.titleTextAttributes = @{NSForegroundColorAttributeName : normalTitleColor};
    itemAppearance.selected.titleTextAttributes = @{NSForegroundColorAttributeName : selectedTitleColor};
    UITabBarAppearance *appearance = [[UITabBarAppearance alloc] init];
    appearance.stackedLayoutAppearance = itemAppearance;
    self.tabBar.standardAppearance = appearance;
    if (@available(iOS 15.0, *)) {
        self.tabBar.scrollEdgeAppearance = appearance;
    } else {
        // Fallback on earlier versions
    }
}else if (@available(iOS 10.0, *)) {
    self.tabBar.tintColor = normalTitleColor;
    self.tabBar.unselectedItemTintColor = selectedTitleColor;
}else {
    // Fallback on earlier versions
}


```
# 七、新增机型设备标识符

```
    //iPhone 13
    if ([platform isEqualToString:@"iPhone14,2"])   return @"iPhone 13 Pro";
    if ([platform isEqualToString:@"iPhone14,3"])   return @"iPhone 13 Pro Max";
    if ([platform isEqualToString:@"iPhone14,4"])   return @"iPhone 13 mini";
    if ([platform isEqualToString:@"iPhone14,5"])   return @"iPhone 13";
	
```
    if ([platform isEqualToString:@"iPad12,1"] ||
        [platform isEqualToString:@"iPad12,2"])     return @"iPad 9";

```

```
    if ([platform isEqualToString:@"iPad13,8"] ||
        [platform isEqualToString:@"iPad13,9"] ||
        [platform isEqualToString:@"iPad13,10"] ||
        [platform isEqualToString:@"iPad13,11"])     return @"iPad Pro 12.9-inch 6";

```
    if ([platform isEqualToString:@"iPad14,1"] ||
        [platform isEqualToString:@"iPad14,2"])     return @"iPad mini 6";

```


# 八、升级Xcode13第三方库Kingfisher报错问题

![](https://github.com/dongpeng66/iOS-/blob/main/images/2.png)


因为项目需要适配iOS10，并且项目没有使用到swiftui，所以就使用ruby脚本将报错相关代码移除掉
在Podfile文件中添加以下钩子函数，然后运行pod install

```
platform :ios, '10.0'
use_modular_headers!
inhibit_all_warnings!
...
pre_install do |installer|
    remove_swiftui()
end

def remove_swiftui
  # 解决 xcode13 Release模式下SwiftUI报错问题
  system("rm -rf ./Pods/Kingfisher/Sources/SwiftUI")
  code_file = "./Pods/Kingfisher/Sources/General/KFOptionsSetter.swift"
  code_text = File.read(code_file)
  code_text.gsub!(/#if canImport\(SwiftUI\) \&\& canImport\(Combine\)(.|\n)+#endif/,'')
  system("rm -rf " + code_file)
  aFile = File.new(code_file, 'w+')
  aFile.syswrite(code_text)
  aFile.close()
end
```

# 九、Xcode13.0 变化
下载后完成，就开始创建一个工程，发现界面变了不少。

![](https://github.com/dongpeng66/iOS-/blob/main/images/4.jpeg)

一开始是没有那些后缀名的，我直接在Xcode->Preferences里面更改的

![](https://github.com/dongpeng66/iOS-/blob/main/images/5.jpeg)

这时候我看到上面有一个issues,显示show inline的选项

show inline显示如下

![](https://github.com/dongpeng66/iOS-/blob/main/images/6.jpeg)

show Minimized是显示如下：

![](https://github.com/dongpeng66/iOS-/blob/main/images/7.jpeg)

对比可以发现1个简洁很多，1个是错误明显很多，就看个人喜爱了。

bulid了一下项目，发现没有buildSuccess的提示了，替代方案可以选择通知，直接在通知，找到xcode,选择提示就可以。

![](https://github.com/dongpeng66/iOS-/blob/main/images/8.jpeg)

效果如下：

![](https://github.com/dongpeng66/iOS-/blob/main/images/9.jpeg)

但感觉还是没有以前好，总是弹通知也感觉有点烦。

![](https://github.com/dongpeng66/iOS-/blob/main/images/10.jpeg)

还有一些项目提示这个，我们直接在File->workspace settings中设置为New Build System就可以了。


![](https://github.com/dongpeng66/iOS-/blob/main/images/11.jpeg)

# 十、 升级ios14 Xcode真机调试启动非常慢的问题解决

1.打开“访达”,shift+command+G到资源库 “~/Library/Developer/Xcode/iOS DeviceSupport/”删除该目录下所有文件 快捷键（shift+command+G到资源库）

![](https://github.com/dongpeng66/iOS-/blob/main/images/12.png)

2.全部删除文件夹下所有文件

选择Xcode->Window->Devices and Simulators 真机设备，鼠标右键选择unpair the device

![](https://github.com/dongpeng66/iOS-/blob/main/images/13.png)


# 十一、企业签名的 App 无法使用

客户反馈说 App 不能正常打开，并且提示下面的这种信息：

```
“xxx”Needs to Be Updated : The developer of this app needs to update it to work with this version of iOS.
“xxx”需要更新 ：App开发者需要更新此App以在此iOS版本上正常工作。
```
![](https://github.com/dongpeng66/iOS-/blob/main/images/3.jpg)

猜测是需要重签名或者使用最新的 Xcode 打包。

我看苹果论坛上有人说必须使用新的 Xcode 以适配 iOS 15，但我用旧版的 Xcode 重新打了一个包，也可以解决这个问题。所以，不方便更新 Xcode 的话，可以尝试重签名试试。

睡了一觉之究极补充： 签名问题与 Xcode 版本无关，而是 Mac 系统导致的，将 macOS 升级到 11.* Big Sur 以上再进行重新签名，可以解决 App 无法使用的问题。

# 十二、SignInApple iOS15.0.1取消 的动画卡动问题

如下：

![](https://github.com/dongpeng66/iOS-/blob/main/images/SignInAppleiOS15.gif)


暂未找到解决方案........待补充.........

# 十三 、iOS15适配本地通知功能

## 消息推送

推送新特性: iOS15以上的新增属性 interruptionLevel为枚UNNotificationInterruptionLevel

## 1.1 本地推送适配

需求：利用本地推送实现消息的语音播报
问题：iOS12.1之后利用本地推送实现消息的语音播报，在iOS15 没有声音。

原因： iOS15本地推送新增了中断级别属性 interruptionLevel,对通知进行了分级	。而且通知的内容不能为空。
方案：使用非Passive的中断级别进行本地通知才会有声音，且本地推送一定要有内容，即body不能为空。content.body = @" 不能为空";

![](https://github.com/dongpeng66/iOS-/blob/main/images/14.png)

```
UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
    UNMutableNotificationContent *content = [[UNMutableNotificationContent alloc] init];
    content.title = @"";
    content.subtitle = @"";
    content.sound = [UNNotificationSound soundNamed:fileName];
    content.badge = @(1);
    
    if (@available(iOS 15.0, *)) {
        content.interruptionLevel = UNNotificationInterruptionLevelTimeSensitive;//会使手机亮屏且会播放声音；可能会在免打扰模式（焦点模式）下展示
// @"{\"aps\":{\"interruption-level\":\"time-sensitive\"}}";
// @"{\"aps\":{\"interruption-level\":\"active\"}}";
        content.body = @" ";// 本地推送一定要有内容，即body不能为空。
        
    }else{
        
        content.body = @"";

    }
    
    UNTimeIntervalNotificationTrigger *trigger = [UNTimeIntervalNotificationTrigger triggerWithTimeInterval:0.001 repeats:NO];
    // 添加通知的标识符，可以用于移除，更新等操作
    NSString *identifier = [NSString stringWithFormat:@"localPushId%lld", (long long)[[NSDate date] timeIntervalSince1970]];
    UNNotificationRequest *request = [UNNotificationRequest requestWithIdentifier:identifier content:content trigger:trigger];
    [center addNotificationRequest:request withCompletionHandler:^(NSError *_Nullable error) {
        CGFloat waitTime = [self timeForAudioFileWithFileName:fileName];
//            CGFloat waitTime = 0.3;
        
        dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(waitTime * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
            [self localNotificationPushNextFile];
        });
    }];
```

中断级别目前分为四种：

```
    typedef NS_ENUM(NSUInteger, UNNotificationInterruptionLevel) {
        // Added to the notification list; does not light up screen or play sound
        UNNotificationInterruptionLevelPassive,
        
        // Presented immediately; Lights up screen and may play a sound
        UNNotificationInterruptionLevelActive,
        
        // Presented immediately; Lights up screen and may play a sound; May be presented during Do Not Disturb
        UNNotificationInterruptionLevelTimeSensitive,
        
        // Presented immediately; Lights up screen and plays sound; Always presented during Do Not Disturb; Bypasses mute switch; Includes default critical alert sound if no sound provided
        UNNotificationInterruptionLevelCritical,

    } API_AVAILABLE(macos(12.0), ios(15.0), watchos(8.0), tvos(15.0));
	
```

1.Passive:被动类型的通知不会使手机亮屏并且不会播放声音。

2.Active: 活动类型的通知会使手机亮屏且会播放声音，为默认类型。

3.Time Sensitive（时间敏感）：会使手机亮屏且会播放声音；可能会在免打扰模式（焦点模式）下展示。


```
设置推送通知数据: 时间敏感的中断级别可以使用“interruption-level” payload key：{"aps":{"interruption-level":"time-sensitive"}}
时效性通知开发者无法直接使用，需要配置对应的权限:
a. xcode 开启对应能力


b. 开发者后台配置appID支持该权限(通过Xcode开启对应能力通常会自动添加)
	
```
![](https://github.com/dongpeng66/iOS-/blob/main/images/15.png)

![](https://github.com/dongpeng66/iOS-/blob/main/images/16.png)

5.Critical（关键）：会立刻展示，亮屏，播放声音，无效免打扰模式，并且能够绕过静音，如果没有设置声音则会使用一种默认的声音。

## 适用于地震等紧急情况,需要特殊申请。

判断是否有时间敏感权限 @property(readonly, nonatomic) UNNotificationSetting timeSensitiveSetting;，如果没有需要提示用户开启。

UNNotificationSetting

```
typedef NS_ENUM(NSInteger, UNNotificationSetting) {
    // The application does not support this notification type
    UNNotificationSettingNotSupported  = 0,
    
    // The notification setting is turned off.
    UNNotificationSettingDisabled,
    
    // The notification setting is turned on.
    UNNotificationSettingEnabled,
} API_AVAILABLE(macos(10.14), ios(10.0), watchos(3.0), tvos(10.0));

```

## 1.2 测试

开发者想打ad hot 的话，需要能访问云端管理的分发证书

![](https://github.com/dongpeng66/iOS-/blob/main/images/17.png)

可以使用极光的网页端或者接口进行测试，或者使用smart push。

![](https://github.com/dongpeng66/iOS-/blob/main/images/18.png)

## 1.3 升级JPush iOS SDK


v4.4.0: pod 'JPush' , '4.4.0'

```
更新时间:2021-10-28

Change Log:

SDK适配ios15系统的本地通知功能

富媒体横屏异常兼容性处理

```


错误信息： ld: library not found for -ljcore-ios-2.3.4

原因：other linker flags 的信息没有自动更新

![](https://github.com/dongpeng66/iOS-/blob/main/images/19.png)

解决方案：直接删除other linker flags的jcore信息即可

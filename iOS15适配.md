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







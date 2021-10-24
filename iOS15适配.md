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





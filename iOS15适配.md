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



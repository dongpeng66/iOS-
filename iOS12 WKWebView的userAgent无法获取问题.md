
```
    if (@available(iOS 12.0, *)){
        //由于iOS12的UA改为异步，所以不管在js还是客户端第一次加载都获取不到，所以此时需要先设置好再去获取（1、如下设置；2、先在AppDelegate中设置到本地）
        [_pWebView setValue:userAgent forKey:@"applicationNameForUserAgent"];
    }
    [_pWebView evaluateJavaScript:@"navigator.userAgent" completionHandler:^(id _Nullable result, NSError * _Nullable error) {
        NSDictionary *dictionary = [NSDictionary dictionaryWithObjectsAndKeys:userAgent,@"UserAgent", nil];
        [[NSUserDefaults standardUserDefaults] registerDefaults:dictionary];
        [[NSUserDefaults standardUserDefaults] synchronize];
        //不添加以下代码则只是在本地更改UA，网页并未同步更改
        if (@available(iOS 9.0, *)) {
            NSLog(@"%@======%@",result,userAgent);
            
            [_pWebView setCustomUserAgent:userAgent];
        } else {
            [_pWebView setValue:userAgent forKey:@"applicationNameForUserAgent"];
        }
    }]; //加载请求必须同步在设置UA的后面

```





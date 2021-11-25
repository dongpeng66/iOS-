

# iOS 检测网络是否真正可用(连通)的方法

在iOS 开发过程中，无论是什么应用，基本上都会涉及到移动设备的网络连接状态判断，以便于在手机等移动端没有网络或者是切换网络后，在应用内给用户以必要的提示，进而提高用户体验。

而iOS 开发中经常用到的Reachability，只能判断手机连接的网络类型（如2g,3g,4g, WiFi或者是无网络连接的情况），并没有实现对所连接网络的可用性（是否可以访问Internet）进行判断。如果手机连接的WiFi并没有接入互联网，那么这台手机虽然连上了WiFi，但是根本就不能上网（例如将路由器的Wan 口网线拔掉）。

下边直接上代码，自己封装的判断网络可用性的方法，根据返回的BOOL 值来判断网络是否可用(可以写到专门的工具类里边，方便在工程中的任何地方调用)。


```
#define kAppleUrlToCheckNetStatus @"http://captive.apple.com/"

- (BOOL)checkNetCanUse {

    __block BOOL canUse = NO;

    NSString *urlString = kAppleUrlToCheckNetStatus;

    // 使用信号量实现NSURLSession同步请求**
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

    [[[NSURLSession sharedSession] dataTaskWithURL:[NSURL URLWithString:urlString] completionHandler:^(NSData * _Nullable data, NSURLResponse * _Nullable response, NSError * _Nullable error) {

        NSString* result = [[NSString alloc]initWithData:data encoding:NSUTF8StringEncoding];
        //解析html页面
        NSString *htmlString = [self filterHTML:result];
        //除掉换行符
        NSString *resultString = [htmlString stringByReplacingOccurrencesOfString:@"\n" withString:@""];

        if ([resultString isEqualToString:@"SuccessSuccess"]) {
            canUse = YES;
           NSLog(@"手机所连接的网络是可以访问互联网的: %d",canUse);

        }else {
            canUse = NO;
            NSLog(@"手机无法访问互联网: %d",canUse);
        }
        dispatch_semaphore_signal(semaphore);
    }] resume];

    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    return canUse;
}
```

//过滤后台返回字符串中的标签

```
- (NSString *)filterHTML:(NSString *)html {

    NSScanner *theScanner;
    NSString *text = nil;

    theScanner = [NSScanner scannerWithString:html];

    while ([theScanner isAtEnd] == NO) {
        // find start of tag
        [theScanner scanUpToString:@"<" intoString:NULL] ;
        // find end of tag
        [theScanner scanUpToString:@">" intoString:&text] ;
        // replace the found tag with a space
        //(you can filter multi-spaces out later if you wish)
        html = [html stringByReplacingOccurrencesOfString:
            [NSString stringWithFormat:@"%@>", text]
                                           withString:@""];
    }
    return html;
}
```


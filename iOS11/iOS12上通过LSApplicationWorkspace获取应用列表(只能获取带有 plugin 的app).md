NSMethodSignature *methodSignature = [NSClassFromString(@"LSApplicationWorkspace") methodSignatureForSelector:NSSelectorFromString(@"defaultWorkspace")];
NSInvocation *invoke = [NSInvocation invocationWithMethodSignature:methodSignature];
[invoke setSelector:NSSelectorFromString(@"defaultWorkspace")];
[invoke setTarget:NSClassFromString(@"LSApplicationWorkspace")];

[invoke invoke];
NSObject * objc;
[invoke getReturnValue:&objc];


NSMethodSignature *installedPluginsmethodSignature = [NSClassFromString(@"LSApplicationWorkspace") instanceMethodSignatureForSelector:NSSelectorFromString(@"installedPlugins")];
NSInvocation *installed = [NSInvocation invocationWithMethodSignature:installedPluginsmethodSignature];
[installed setSelector:NSSelectorFromString(@"installedPlugins")];

[installed setTarget:objc];

[installed invoke];
NSObject * arr;
[installed getReturnValue:&arr];


for (NSObject *objc in arr) {

    NSMethodSignature *installedPluginsmethodSignature = [NSClassFromString(@"LSPlugInKitProxy") instanceMethodSignatureForSelector:NSSelectorFromString(@"containingBundle")];
    NSInvocation *installed = [NSInvocation invocationWithMethodSignature:installedPluginsmethodSignature];
    [installed setSelector:NSSelectorFromString(@"containingBundle")];
    [installed setTarget:objc];

    [installed invoke];
    NSObject * app;
    [installed getReturnValue:&app];
    NSLog(@"%@",app);
}

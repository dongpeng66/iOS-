
```
@implementation NSObject (ZKLSDefaults)
#pragma mark - ios13系统_LSDefaults崩溃解决办法
+ (void)load{
    
    SEL originalSelector = @selector(doesNotRecognizeSelector:);
    SEL swizzledSelector = @selector(sw_doesNotRecognizeSelector:);
    
    Method originalMethod = class_getClassMethod(self, originalSelector);
    Method swizzledMethod = class_getClassMethod(self, swizzledSelector);
    
    if(class_addMethod(self, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod))){
        class_replaceMethod(self, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
    }else{
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
 
+ (void)sw_doesNotRecognizeSelector:(SEL)aSelector{
    //处理 _LSDefaults 崩溃问题
    if([[self description] isEqualToString:@"_LSDefaults"] && (aSelector == @selector(sharedInstance))){
        //冷处理...
        return;
    }
    [self sw_doesNotRecognizeSelector:aSelector];

}
@end
```






```
#import <objc/runtime.h>
@implementation UIView (GMTabBarItem)
#pragma mark - 如果使用的是默认的UITabBar磨砂效果，而且在pushViewController时，设置了hidesBottomBarWhenPushed属性为true ，则在iOS 12.1系统，使用手势返回的时候，就会触发UITabBarButton 被设置frame.size 为 (0, 0)的错误布局， 导致的tabBar上的图标和文字偏移错位。
#pragma mark - 解决iOS 12.1系统tabBar按钮偏移错位问题
+ (void)load{
    if (@available(iOS 12.1, *)) {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
            Class class = NSClassFromString(@"UITabBarButton");
            SEL originalSelector  = @selector(setFrame:);
            SEL swizzledSelector  = @selector(ck_setFrame:);
            Method originalMethod = class_getInstanceMethod(class, originalSelector);
            Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
            BOOL isSuccess = class_addMethod(class,
                                             originalSelector,
                                             method_getImplementation(swizzledMethod),
                                             method_getTypeEncoding(swizzledMethod));
            if (isSuccess) {
                class_replaceMethod(class,
                                    swizzledSelector,
                                    method_getImplementation(originalMethod),
                                    method_getTypeEncoding(originalMethod));
            } else {
                method_exchangeImplementations(originalMethod, swizzledMethod);
            }
        });
    }
}

- (void)ck_setFrame:(CGRect)frame
{
    if (!CGRectIsEmpty(self.frame)) {
        if (CGRectIsEmpty(frame)) {
            return;
        }
        frame.size.height = MAX(frame.size.height, 49.0);
    }
    [self ck_setFrame: frame];
}

@end
```






```
@interface GMGestureRecognizerScrollView : UIScrollView

@end


@implementation GMGestureRecognizerScrollView
//是否支持多手势触发，返回YES，则可以多个手势一起触发方法，返回NO则为互斥.
//是否允许多个手势识别器共同识别，一个控件的手势识别后是否阻断手势识别继续向下传播，默认返回NO；如果为YES，响应者链上层对象触发手势识别后，如果下层对象也添加了手势并成功识别也会继续执行，否则上层对象识别后则不再继续传播
//一句话总结就是此方法返回YES时，手势事件会一直往下传递，不论当前层次是否对该事件进行响应。
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer {
   
    if ([self isPanBackAction:gestureRecognizer]) {
        return YES;
    }
    return NO;
    
}
/// 判断是否是全屏的返回手势
- (BOOL)isPanBackAction:(UIGestureRecognizer *)gestureRecognizer {
    
    // 在最左边时候 && 是pan手势 && 手势往右拖
    if (self.contentOffset.x == 0) {
        if (gestureRecognizer == self.panGestureRecognizer) {
            // 根据速度获取拖动方向
            CGPoint velocity = [self.panGestureRecognizer velocityInView:self.panGestureRecognizer.view];
            if(velocity.x>0){
                //手势向右滑动
                return YES;
            }
        }
        
    }
    return NO;
}
 
// 如果是全屏的左滑返回,那么ScrollView的左滑就没用了,返回NO,让ScrollView的左滑失效
// 不写此方法的话,左滑时,那个ScrollView上的子视图也会跟着左滑的
- (BOOL)gestureRecognizerShouldBegin:(UIGestureRecognizer *)gestureRecognizer {
 
    if ([self isPanBackAction:gestureRecognizer]) {
        return NO;
    }
    return YES;
 
}

/*
// Only override drawRect: if you perform custom drawing.
// An empty implementation adversely affects performance during animation.
- (void)drawRect:(CGRect)rect {
    // Drawing code
}
*/

@end
```





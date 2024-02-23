### 我们在iOS/iPadOS上遇到了一个特定的错误，似乎是在iPad在屏幕上显示浮动（类似于iPhone的）键盘（而不是固定的键盘）时触发的

### 最佳答案

+ 没有这样的模式枚举/指示符（至少目前是这样），但是具有键盘框架信息
+ 是通过键盘宽度是否可以改变可以进行判断是否为悬浮键盘
+ 根据键盘宽度的改变 是否为悬浮键盘


浮动键盘的时候,键盘事件（UIKeyboardWillShowNotification，UIKeyboardWillHideNotification等）似乎没有返回任何特定的线索,并且（UIKeyboardWillShowNotification，UIKeyboardWillHideNotification）
通知回调不调用。

### 适配方案

通过UIKeyboardWillChangeFrameNotification这个通知回调来做一些事情

### 添加通知
```

[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(p_keyBoardWillChangeFrame:) name:UIKeyboardWillChangeFrameNotification object:nil];

```

### 通知回调

```
- (void)p_keyBoardWillChangeFrame:(NSNotification *)notification {
    NSLog(@"boardWillChangeFrameNotification=======");
    // 获取键盘的高度
    //此方法先调用可以用来判断是不是浮动键盘
    CGRect keyboardFrame = [[[notification userInfo] objectForKey:UIKeyboardFrameEndUserInfoKey] CGRectValue];
    CGFloat duration = [[notification.userInfo valueForKey:UIKeyboardAnimationDurationUserInfoKey] floatValue];
    if (keyboardFrame.size.width != UIScreen.mainScreen.bounds.size.width && IS_IPAD) {
        //浮动键盘
        NSLog(@"浮动键盘");
        
        if ((keyboardFrame.size.width == 0) && (keyboardFrame.size.height == 0) && (keyboardFrame.origin.y == 0) && (keyboardFrame.origin.x == 0)) {
            //键盘隐藏
        }else{
            //键盘显示
        }
    }else{
        //正常键盘
        NSLog(@"正常键盘");
       
    }
}

```

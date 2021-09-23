
```
/**cell可以提前加载   去掉loading框可以实现无感加载*/
-(void)setRefreshWithHeaderBlock:(void (^)(void))headerBlock AutoFooterBlock:(void (^)(void))footerBlock withStatus:(BOOL)isHaveEarlyLoading{
    if (headerBlock) {
        self.header = [GMGifHeader headerWithRefreshingBlock:^{
            if (headerBlock) {
                headerBlock();
            }
        }];
        // 隐藏时间
        self.header.stateLabel.hidden = NO;
        self.header.lastUpdatedTimeLabel.hidden = YES;
        self.mj_header = self.header;

    }
    if (footerBlock) {
        GMRefreshAutoNormalFooter *footer = [GMRefreshAutoNormalFooter footerWithRefreshingBlock:^{
            footerBlock();
        }];
        if (isHaveEarlyLoading) {
		    //实现cell无感加载
            footer.triggerAutomaticallyRefreshPercent = -88;
        }
        self.mj_footer = footer;
    }
}
```






```
// 解决iOS10 以上'layout attributes for supplementary item at index path ( {length = 2, path = 0 - 0}) changed from CustomSupplementaryAttributes: 0xd1123a0 index path: (NSIndexPath: 0xd112580 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex = -1;  to CustomSupplementaryAttributes: 0xd583c80 index path: (NSIndexPath: 0xd583c70 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex=-1；
if ([self.collectionView respondsToSelector:@selector(setPrefetchingEnabled:)]) {
    if ( @available(iOS 10, *) ) {
        self.collectionView.prefetchingEnabled = false;
    }
}
    





@implementation GMLeftToRightCollectionViewLayout

- (NSArray *) layoutAttributesForElementsInRect:(CGRect)rect {
    // 获取系统帮我们计算好的Attributes
    NSArray *answer = [super layoutAttributesForElementsInRect:rect];
    
    
    UICollectionViewLayoutAttributes *firstLayoutAttributes = answer.firstObject;
    if (firstLayoutAttributes) {
        CGRect frame = firstLayoutAttributes.frame;
        frame.origin.x = self.sectionInset.left;
        firstLayoutAttributes.frame = frame;
    }
    
    // 遍历结果
    for(int i = 1; i < [answer count]; ++i) {
        
        // 获取cell的Attribute，根据上一个cell获取最大的x，定义为origin
        UICollectionViewLayoutAttributes *currentLayoutAttributes = answer[i];
        UICollectionViewLayoutAttributes *prevLayoutAttributes = answer[i - 1];
        
        //只需要调整cell，过滤header、footer
        if([currentLayoutAttributes.representedElementKind isEqualToString:UICollectionElementKindSectionHeader] || [currentLayoutAttributes.representedElementKind isEqualToString:UICollectionElementKindSectionFooter]){
            continue;
        }
        
        NSInteger preX = CGRectGetMaxX(prevLayoutAttributes.frame);
        NSInteger preY = CGRectGetMaxY(prevLayoutAttributes.frame);
        NSInteger curY = CGRectGetMaxY(currentLayoutAttributes.frame);
        
        // 如果当前cell和precell在同一行
        if(preY == curY){
            //满足则给当前cell的frame属性赋值
            //不满足的cell会根据系统布局换行
            CGRect frame = currentLayoutAttributes.frame;
            frame.origin.x = preX + _maximumSpacing;
            currentLayoutAttributes.frame = frame;
        } else {
            CGRect frame = currentLayoutAttributes.frame;
            frame.origin.x = self.sectionInset.left;
            currentLayoutAttributes.frame = frame;
        }
    }
    return answer;
}
#pragma mark - 解决iOS10 以上'layout attributes for supplementary item at index path ( {length = 2, path = 0 - 0}) changed from CustomSupplementaryAttributes: 0xd1123a0 index path: (NSIndexPath: 0xd112580 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex = -1;  to CustomSupplementaryAttributes: 0xd583c80 index path: (NSIndexPath: 0xd583c70 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex=-1；
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds {
    return YES;
}
@end

```





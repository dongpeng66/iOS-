
```
    // 解决iOS10 以上'layout attributes for supplementary item at index path ( {length = 2, path = 0 - 0}) changed from CustomSupplementaryAttributes: 0xd1123a0 index path: (NSIndexPath: 0xd112580 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex = -1;  to CustomSupplementaryAttributes: 0xd583c80 index path: (NSIndexPath: 0xd583c70 {length = 2, path = 0 - 0}); element kind: (identifier); frame = (0 0; 1135.66 45); zIndex=-1；
    if ([self.collectionView respondsToSelector:@selector(setPrefetchingEnabled:)]) {
        if ( @available(iOS 10, *) ) {
            self.collectionView.prefetchingEnabled = false;
        }
    }

```





# iOS-
iOS笔记
### 我遇到这样一个问题。步骤如下

1.UIImage 压缩  UIImage 转NSData 大小为500百多K.压缩到100K以内。 返回NSData

这时的NSData 为90多K

2.NSData 转成UIImage 存储

3.用到时，把UIImage 转成NSData 进行使用，这时NSData 大约为150 - 200左右。

很是奇怪。不知道为什么。

暂时的解决办法是，压缩后的NSData直接存储，不转成UiImage 存储。

或者上传图片的时候改变一下网络请求方式
```
+ (void)uploadImageURL:(NSString *)url
            parameters:(NSDictionary *)parameters
               headers:(NSDictionary *)headers
                images:(NSArray<UIImage *> *)images
                  name:(NSString *)name
              fileName:(NSString *)fileName
              mimeType:(NSString *)mimeType
              progress:(GMHttpProgress)progress
              callback:(GMHttpRequest)callback;
+ (void)uploadImageURL:(NSString *)url
            parameters:(NSDictionary *)parameters
               headers:(NSDictionary *)headers
                dataImages:(NSArray<NSData *> *)images
                  name:(NSString *)name
              fileName:(NSString *)fileName
              mimeType:(NSString *)mimeType
              progress:(GMHttpProgress)progress
              callback:(GMHttpRequest)callback;
```

```
//压缩-添加-上传图片
[images enumerateObjectsUsingBlock:^(UIImage * _Nonnull image, NSUInteger idx, BOOL * _Nonnull stop) {
    NSData *imageData = UIImageJPEGRepresentation(image, 0.5);
    [formData appendPartWithFileData:imageData name:name fileName:[NSString stringWithFormat:@"%@%lu.%@",fileName,(unsigned long)idx,mimeType ? mimeType : @"jpeg"] mimeType:[NSString stringWithFormat:@"image/%@",mimeType ? mimeType : @"jpeg"]];
}];


[images enumerateObjectsUsingBlock:^(NSData * _Nonnull image, NSUInteger idx, BOOL * _Nonnull stop) {
    [formData appendPartWithFileData:image name:name fileName:[NSString stringWithFormat:@"%@%lu.%@",fileName,(unsigned long)idx,mimeType ? mimeType : @"jpeg"] mimeType:[NSString stringWithFormat:@"image/%@",mimeType ? mimeType : @"jpeg"]];
}];
```

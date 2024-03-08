```
#pragma mark - 通过检测ipa中的embedded.mobileprovision来确定是否签名被修改，但是需要注意的是此方法只适用于Ad Hoc或企业证书打包的情况，App Store上应用由苹果私钥统一打包，不存在embedded.mobileprovision文件
+ (BOOL)isLegalSignTeamIds:(NSArray *)teamIds{
    if (TARGET_IPHONE_SIMULATOR) return YES;
    // 描述文件路径
    NSString *embeddedPath = [[NSBundle mainBundle] pathForResource:@"embedded" ofType:@"mobileprovision"];
    if (![[NSFileManager defaultManager] fileExistsAtPath:embeddedPath]) {
        return YES; //苹果商店的正版app
    }
    // 读取application-identifier  注意描述文件的编码要使用:NSASCIIStringEncoding
    NSString *embeddedProvisioning = [NSString stringWithContentsOfFile:embeddedPath encoding:NSASCIIStringEncoding error:nil];
    NSArray<NSString *> *embeddedProvisioningLines = [embeddedProvisioning componentsSeparatedByCharactersInSet:[NSCharacterSet newlineCharacterSet]];
    for (int i = 0; i < embeddedProvisioningLines.count; i++) {
        if ([embeddedProvisioningLines[i] rangeOfString:@"application-identifier"].location != NSNotFound) {
            NSString *identifierString = embeddedProvisioningLines[i + 1]; // 类似：<string>L2ZY2L7GYS.com.xx.xxx</string>
            NSRange fromRange = [identifierString rangeOfString:@"<string>"];
            NSInteger fromPosition = fromRange.location + fromRange.length;
            NSInteger toPosition = [identifierString rangeOfString:@"</string>"].location;
            NSRange range;
            range.location = fromPosition;
            range.length = toPosition - fromPosition;
            NSString *fullIdentifier = [identifierString substringWithRange:range];

            NSString *delimiter = @".";
            NSArray *components = [fullIdentifier componentsSeparatedByString:delimiter];
            NSString *teamIdString = @"";
            if (components.count > 1) {
                teamIdString = components.firstObject;
            }
            
            // 对比签名teamID
            if (teamIdString && ![teamIds containsObject:teamIdString]) {
                return NO;
            }
            break;
        }
    }
    return YES;
}
```

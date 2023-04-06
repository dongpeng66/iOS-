# iOS笔记-iOS中如何限制TextField输入的字符个数（含有中文和英文）
在IOS开发中，TextField可以说是我们最熟悉，也是平时用的最多的控件，其本身的用法比较简单，但是在限制输入上，就会出现一些奇奇怪怪的需求，比如说：不能输入表情，不能输入中文，输入的字符个数不能超过20个…等等，可谓是各种花式需求让我们提到这个控件的时候还是有一丝的心虚。今天正好有空，就来谈谈最近一个比较有趣的需求，需求如下：
要求：限制TextField中输入的字符不超过40个，如果是中文字符的话，不能超过20个
我们知道，在IOS中，限制TextField最大输入字符的方法如下：

```
- (BOOL)searchBar:(UISearchBar *)searchBar shouldChangeTextInRange:(NSRange)range replacementText:(NSString *)text{
    if (text.length > INPUT_MAX) {
        return NO;
    }
    return YES;
}
```
那如何能计算输入的中文是20个，英文是40个，如果是中英文混合那又该如何确定这个字符串的长度呢？

如果我们以UTF-8字符集表示的话，一个中文占3个字节，一个英文占一个字节；
如果我们用Unicode字符集表示的话，中文和英文一样，都只占用2个字节；
所以，我们可以利用字节的个数，来确定输入的中英文的个数，具体解决方案如下：

解决方案：
```
+ (NSUInteger)countCharNumOfString:(NSString *) textString{
    NSUInteger charCountResult = 0;
    
    //utf8下 一个汉字 3 字节，一个英文1字节   unicode下所有字符2字节
    NSUInteger nameBytes = [textString lengthOfBytesUsingEncoding:NSUTF8StringEncoding];
    if (nameBytes)
    {
        NSUInteger unicodeBytes = [textString lengthOfBytesUsingEncoding:NSUnicodeStringEncoding];
        if (nameBytes * 2 >= unicodeBytes)
        {
            NSUInteger chiCharNum = (nameBytes * 2 - unicodeBytes) / 4;     //中文字符的个数 = （utf8字节x2 - unicode字节） / 4
            if (textString.length >= chiCharNum)
            {
                NSUInteger engCharNum = textString.length - chiCharNum;
                charCountResult  = chiCharNum * 2 + engCharNum;
            }
            //else error!
        }
        //else error!
    }
    //else cont.
    
    return charCountResult;
}
```

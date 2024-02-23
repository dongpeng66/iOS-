### 前言

Xcode 15 以及 iOS 17 已经发布一段时间了，作为开发者，需要关注的是哪些 API 有所改动，特别是底层的 API，影响面可能会比较广，今天讲一个关于 URL 对象的 API 更改。

```
For apps linked on or after iOS 17 and aligned OS versions, URL parsing has updated from the obsolete RFC 1738/1808 parsing to the same RFC 3986 parsing as URLComponents. This unifies the parsing behaviors of the URL and URLComponents APIs. Now, URL automatically percent- and IDNA-encodes invalid characters to help create a valid URL.

```

翻译一下就是 URL 解析已从过时的 RFC 1738/1808 解析更新为与 URLComponents 相同的 RFC 3986 解析。

### 什么是 RFC
RFC 是 Request for Comments 的首字母缩写，是互联网工程任务组 （IETF）的对于 URL 的规范出的文档，其中包含有关互联网和计算机网络相关主题（如路由、寻址和传输技术）的规范和组织说明。

针对新的标准，个人或团体可以向委员会提交内容，委员会工作组经过一系列审核并通过之后，RFC 生产中心 （RPC） 会为 RFC 分配一个唯一编号，并通过 RFC 编辑器发布。RFC 发布后，它永远不会更改。我们前面提到的 RFC 1738/1808 和 RFC 3986 就是他们发布的其中一个标准。


### 苹果的支持
在 iOS 17 之前，URL 初始化时支持的是较久的 RFC 1738/1808 标准，但是 URLComponents 支持的是 RFC 3986 标准，两个标准不同导致 iOS 开发者在某些时候比较困惑，所以苹果在今年的更新中，终于把标准统一了，统一支持 RFC 3986 标准。

### 变化是什么？
在 iOS 17 之前，URL(string: "Not an URL") 会返回 nil，因为这是一个无效的 URL 格式，但是在 iOS 17 上，这段代码将会返回 Not%an%URL，相当于是把中间的空格进行了一次 encode。

```
// iOS 16
let validURL = URL(string: "https://google.com") // 返回 https://google.com
let url = URL(string: "Not an URL")              // 返回 nil

// iOS 17
let validURL = URL(string: "https://google.com") // 返回 https://google.com
let url = URL(string: "Not an URL")              // 返回 Not%an%URL

```

新的行为对于老的项目可能不太友好，甚至可能出现非预期的行为，比如，之前可能会有这样的判断：

```
if let url = URL(string: "Not an URL") {
    // url 有效
}

```


但这段代码在 Xcode 15 中 iOS 17 上也会走进去，因此可能导致非预期的后果。
### 如何兼容
Apple 在 iOS 17 的 URL 初始化方法中，添加了一个带有 Bool （默认值 true）的新参数 encodingInvalidCharacters，这个参数代表是否 encoding 掉无效的字符，如果传 false，URL 的行为将会和 iOS 16 上一致。

```
public init?(string: String, encodingInvalidCharacters: Bool)

```

使用效果如下：

```
// iOS 16
let url = URL(string: "Not an URL") // => nil

// iOS 17
let url = URL(string: "Not an URL", encodingInvalidCharacters: false) // => nil


```

如果你不想改变 URL 的默认行为，从而兼容以前的代码，需要把之前使用 URL 初始化的方法加上 encodingInvalidCharacters: false 参数。

### 结论

虽然苹果使 URL 支持了新的标准，和 URLComponents 保持了一致，这是好事，但是一刀切直接把默认行为改掉，这点还是比较坑的。更保守的做法应该是开一个新的方法来支持新标准，并且将老的方法标记为过期，慢慢过渡。不得不说，苹果还是那个有些坑开发者的苹果。


### 项目中配置

本次 Apple iOS 17 升级，对 NSURL 类的 URLWithString 进行了隐式升级（ 点名批评  ，应用甚广的API竟然硬升级）

对齐了NSURL与 NSURLComponents 的执行标准，统一为 RFC 3986 （各执行标准感兴趣可以自行查阅，RFC 1808、RFC 1738、RFC 2732、RFC 3986）

iOS 16：URLWithString 判断参数 urlString 如有非法字符（包含中文字符）就返回 nil

iOS 17：URLWithString 默认对非法字符（包含中文字符）进行%转义

直接看文档，iOS 17 以后URL中如果出现中文字符也可以直接进行编码，这看起来很美好，实际测试时发现有以下问题：

如果URL中没有非法字符，那URL中原有的转义字符（%）不会再次转义
如果URL中含有非法字符，那URL中原有的转义字符（%）会再次转义 （本次遇到的BUG）

针对此种的适配方案

```

#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface NSURL (ZZAdapter)

@end

NS_ASSUME_NONNULL_END

```

```
#import "NSURL+ZZAdapter.h"
#import <objc/runtime.h>
#pragma mark - URL encode
static NSString * ZZAdapterPercentEscapedStringFromString(NSString *string) {
    static NSString * const kZZAdapterCharactersGeneralDelimitersToEncode = @":#[]@/?";
    static NSString * const kZZAdapterCharactersSubDelimitersToEncode = @"!$&'()*+,;=";

    NSMutableCharacterSet * allowedCharacterSet = [[NSCharacterSet URLQueryAllowedCharacterSet] mutableCopy];
    [allowedCharacterSet removeCharactersInString:[kZZAdapterCharactersGeneralDelimitersToEncode stringByAppendingString:kZZAdapterCharactersSubDelimitersToEncode]];
    
    static NSUInteger const batchSize = 50;

    NSUInteger index = 0;
    NSMutableString *escaped = @"".mutableCopy;

    while (index < string.length) {
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wgnu"
        NSUInteger length = MIN(string.length - index, batchSize);
#pragma GCC diagnostic pop
        NSRange range = NSMakeRange(index, length);
        range = [string rangeOfComposedCharacterSequencesForRange:range];
        NSString *substring = [string substringWithRange:range];
        NSString *encoded = [substring stringByAddingPercentEncodingWithAllowedCharacters:allowedCharacterSet];
        [escaped appendString:encoded];
        index += range.length;
    }

    return escaped;
}

@interface ZZAdapterQueryStringPair : NSObject
@property (readwrite, nonatomic, strong) id field;
@property (readwrite, nonatomic, strong) id value;

- (id)initWithField:(id)field value:(id)value;

- (NSString *)URLEncodedStringValue;
@end

@implementation ZZAdapterQueryStringPair

- (id)initWithField:(id)field value:(id)value {
    self = [super init];
    if (self) {
        self.field = field;
        self.value = value;
    }
    return self;
}

- (NSString *)URLEncodedStringValue {
    if (!self.value || [self.value isEqual:[NSNull null]]) {
        return ZZAdapterPercentEscapedStringFromString([self.field description]);
    } else {
        return [NSString stringWithFormat:@"%@=%@", ZZAdapterPercentEscapedStringFromString([self.field description]), ZZAdapterPercentEscapedStringFromString([self.value description])];
    }
}
@end

FOUNDATION_EXPORT NSArray * ZZAdapterQueryStringPairsFromDictionary(NSDictionary *dictionary);
FOUNDATION_EXPORT NSArray * ZZAdapterQueryStringPairsFromKeyAndValue(NSString *key, id value);

static NSString * ZZAdapterQueryStringFromParameters(NSDictionary *parameters) {
    NSMutableArray *mutablePairs = [NSMutableArray array];
    for (ZZAdapterQueryStringPair *pair in ZZAdapterQueryStringPairsFromDictionary(parameters)) {
        [mutablePairs addObject:[pair URLEncodedStringValue]];
    }

    return [mutablePairs componentsJoinedByString:@"&"];
}

NSArray * ZZAdapterQueryStringPairsFromDictionary(NSDictionary *dictionary) {
    return ZZAdapterQueryStringPairsFromKeyAndValue(nil, dictionary);
}

NSArray * ZZAdapterQueryStringPairsFromKeyAndValue(NSString *key, id value) {
    NSMutableArray *mutableQueryStringComponents = [NSMutableArray array];

    NSSortDescriptor *sortDescriptor = [NSSortDescriptor sortDescriptorWithKey:@"description" ascending:YES selector:@selector(compare:)];

    if ([value isKindOfClass:[NSDictionary class]]) {
        NSDictionary *dictionary = value;
        for (id nestedKey in [dictionary.allKeys sortedArrayUsingDescriptors:@[sortDescriptor]]) {
            id nestedValue = dictionary[nestedKey];
            if (nestedValue) {
                [mutableQueryStringComponents addObjectsFromArray:ZZAdapterQueryStringPairsFromKeyAndValue((key ? [NSString stringWithFormat:@"%@[%@]", key, nestedKey] : nestedKey), nestedValue)];
            }
        }
    } else if ([value isKindOfClass:[NSArray class]]) {
        NSArray *array = value;
        for (id nestedValue in array) {
            [mutableQueryStringComponents addObjectsFromArray:ZZAdapterQueryStringPairsFromKeyAndValue([NSString stringWithFormat:@"%@[]", key], nestedValue)];
        }
    } else if ([value isKindOfClass:[NSSet class]]) {
        NSSet *set = value;
        for (id obj in [set sortedArrayUsingDescriptors:@[ sortDescriptor ]]) {
            [mutableQueryStringComponents addObjectsFromArray:ZZAdapterQueryStringPairsFromKeyAndValue(key, obj)];
        }
    } else {
        [mutableQueryStringComponents addObject:[[ZZAdapterQueryStringPair alloc] initWithField:key value:value]];
    }

    return mutableQueryStringComponents;
}

#pragma mark - iOS 17 NSURL Bug
@implementation NSURL (ZZAdapter)

+(void)initialize {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if (@available(iOS 17.0, *)) {
            /*
             [NSURL URLWithString:@"A"];
             最终调用 [NSURL alloc] initWithString:@"A" relativeToURL:nil];
             
             [[NSURL alloc] initWithString:@"A"];
             最终调用 [[NSURL alloc] initWithString:@"A" relativeToURL:nil];
             */
            Method oriMethod = class_getInstanceMethod(self, @selector(initWithString:relativeToURL:));
            Method newMethod = class_getInstanceMethod(self, @selector(ZZAdapter_initWithString:relativeToURL:));
            method_exchangeImplementations(oriMethod, newMethod);
        }
    });
}

- (nullable instancetype)ZZAdapter_initWithString:(NSString *)URLString relativeToURL:(nullable NSURL *)baseURL {
    
    if (@available(iOS 17.0, *)) {
        /*
         不改动 scheme、host、path、params、fragment 以及 baseURL
         只对 query 部分重新编码
         
         NSURL 文档：
         https://developer.apple.com/documentation/foundation/nsurl?language=objc
         
         URL 标准文档：
         RFC 1808: https://datatracker.ietf.org/doc/html/rfc1808
         <scheme>://<net_loc>/<path>;<params>?<query>#<fragment>
         each of which, except <scheme>, may be absent from a particular URL.
         These components are defined as follows (a complete BNF is provided
         in Section 2.2):
          
         scheme ":"   ::= scheme name, as per Section 2.1 of RFC 1738 [2].
        
         "//" net_loc ::= network location and login information, as per
                             Section 3.1 of RFC 1738 [2].
        
         "/" path     ::= URL path, as per Section 3.1 of RFC 1738 [2].
        
         ";" params   ::= object parameters (e.g., ";type=a" as in
                             Section 3.2.2 of RFC 1738 [2]).
          
         "?" query    ::= query information, as per Section 3.3 of
                             RFC 1738 [2].
         
         "#" fragment ::= fragment identifier.
         
        */
        
        NSString *host = nil; // scheme + net_loc + host + path + params
        NSString *query = nil;
        NSString *fragment = nil;
        NSString *url = URLString ?: @"";
        
        // scheme、net_loc、host、path、params、query
        NSRange hostRange = [url rangeOfString:@"?" options:NSBackwardsSearch];
        // NSBackwardsSearch 反向查，规避存在多个 ? 的情况
        if (hostRange.location != NSNotFound && hostRange.length > 0) {
            host = [url substringToIndex:hostRange.location + 1];
            if (hostRange.location + 1 < url.length) {
                url = [url substringFromIndex:hostRange.location + 1];
            } else {
                url = @"";
            }
        } else {
            // 无 query
            NSRange paramsRange = [url rangeOfString:@";" options:NSBackwardsSearch];
            if (paramsRange.location != NSNotFound && paramsRange.length > 0) {
                host = [url substringToIndex:paramsRange.location + 1];
                if (paramsRange.location + 1 < url.length) {
                    url = [url substringFromIndex:paramsRange.location + 1];
                } else {
                    url = @"";
                }
            } else {
                // 无 params
                NSRange lastPathRange = [url rangeOfString:@"/" options:NSBackwardsSearch];
                if (lastPathRange.location != NSNotFound && lastPathRange.length > 0) {
                    host = [url substringToIndex:lastPathRange.location + 1];
                    if (lastPathRange.location + 1 < url.length) {
                        url = [url substringFromIndex:lastPathRange.location + 1];
                    } else {
                        url = @"";
                    }
                }
            }
        }
        
        // fragment
        NSRange fragmentRange = [url rangeOfString:@"#"];
        if (fragmentRange.location != NSNotFound && fragmentRange.length > 0) {
            fragment = [url substringFromIndex:fragmentRange.location];
            url = [url substringToIndex:fragmentRange.location];
        }
        
        // query
        NSRange queryRange1 = [url rangeOfString:@"&"];
        NSRange queryRange2 = [url rangeOfString:@"="];
        if ((queryRange1.location != NSNotFound && queryRange1.length > 0)
            || (queryRange2.location != NSNotFound && queryRange2.length > 0)) {
            NSArray *params = [url componentsSeparatedByString:@"&"];
            NSMutableDictionary *kvs = [NSMutableDictionary new];
            for (NSString *param in params) {
                NSArray *sup = [param componentsSeparatedByString:@"="];
                NSString *key = [sup.firstObject stringByRemovingPercentEncoding];
                NSString *value = [sup.lastObject stringByRemovingPercentEncoding];
                if (sup.lastObject && sup.firstObject) {
                    [kvs setValue:value forKey:key];
                }
            }
            // encode query
            if (kvs.allKeys.count > 0) {
                NSString *enodeStr = ZZAdapterQueryStringFromParameters(kvs);
                if ([enodeStr isKindOfClass:[NSString class]]
                    && enodeStr.length > 0) {
                    query = enodeStr;
                    url = @"";
                }
            }
        }
        
        NSString *encodeUrl = [NSString stringWithFormat:@"%@%@%@%@", host ?: @"", url?: @"", query?:@"", fragment ?: @""];
        if (encodeUrl.length > 0) {
            return [self ZZAdapter_initWithString:encodeUrl relativeToURL:baseURL];
        } else {
            return [self ZZAdapter_initWithString:URLString relativeToURL:baseURL];
        }
    } else {
        return [self ZZAdapter_initWithString:URLString relativeToURL:baseURL];
    }
}
@end

```


针对返回nil判断的问题的适配分为Swift和OC两种

### OC

```
#import <Foundation/Foundation.h>

NS_ASSUME_NONNULL_BEGIN

@interface NSURL (iOS17)

@end

NS_ASSUME_NONNULL_END
```


```
#import "NSURL+iOS17.h"
#import <objc/runtime.h>
@implementation NSURL (iOS17)
+(void)load {
    Method ori_Method = class_getClassMethod(self, @selector(URLWithString:));
    Method my_Method = class_getClassMethod(self, @selector(zz_URLWithString:));
    method_exchangeImplementations(ori_Method, my_Method);
    
    
    Method ori_MethodInit = class_getInstanceMethod(self, @selector(initWithString:));
    Method my_MethodInit = class_getInstanceMethod(self, @selector(zz_initWithString:));
    method_exchangeImplementations(ori_MethodInit, my_MethodInit);
    

}

+ (instancetype)zz_URLWithString:(NSString *)URLString {
    if (@available(iOS 17.0, *)) {
        return [self URLWithString:URLString encodingInvalidCharacters:NO];
    } else {
        return [self zz_URLWithString:URLString];
    }
}
//- (instancetype)zz
- (instancetype)zz_initWithString:(NSString *)URLString {
    if (@available(iOS 17.0, *)) {
        return [self initWithString:URLString encodingInvalidCharacters:NO];
    } else {
        return [self zz_initWithString:URLString];
    }
}
@end
```


### Swift

```
import Foundation

extension URL {
    
    public static func ZZinit(string: String) ->URL?{
        if #available(iOS 17.0, *) {

            //高于 iOS 17.0
           return URL(string: string, encodingInvalidCharacters: false)

        } else {

            //低于 iOS 17.0
            return URL(string: string)
        }
    }
}
```

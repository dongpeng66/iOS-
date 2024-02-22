最近项目遇到的问题：

UILabel显示文字内容如果是中文+英文，则可能遇到奇怪的换行处理，中文被诡异换行，导致第一行文字显示不完整。

如果设置UILabel或者NSMutableParagraphStyle的lineBreakMode为byCharWrapping，虽然解决了换行问题，但是内容过长的时候又没有省略号了，似乎二者不可兼得。

修复前的样式：
![](https://github.com/dongpeng66/iOS-/blob/main/images/20.png)


解决方案：使用UITextView来显示内容，当成UILabel来显示。


添加UITextView：

```
let textView: UITextView = UITextView()
textView.frame = CGRect(x: ulabel.frame.minX, y: ulabel.frame.maxY + 20.0, width: ulabel.frame.width, height: ulabel.frame.height)
// 内容边距，左边不缩进默认为5的距离
textView.textContainer.lineFragmentPadding = 0
// 内容边距，从顶部开始显示
textView.textContainerInset = UIEdgeInsets.zero
textView.backgroundColor = UIColor.brown
textView.showsHorizontalScrollIndicator = false
textView.showsVerticalScrollIndicator = false
textView.isEditable = false
textView.isScrollEnabled = false
 
 
// 保证内容只显示2行，设置换行模式：末尾省略号
textView.textContainer.maximumNumberOfLines = 2
textView.textContainer.lineBreakMode = .byTruncatingTail
self.view.addSubview(textView)
```

构造富文本NSMutableAttributedString，富文本设置段落样式加上这两句代码：

```
let paragraphStyle = NSMutableParagraphStyle()
paragraphStyle.lineBreakMode = .byCharWrapping
 
yourAtt.addAttribute(.paragraphStyle, value: paragraphStyle, range: NSMakeRange(0, yourAtt.length))

```

修复后的样式：

![](https://github.com/dongpeng66/iOS-/blob/main/images/21.png)

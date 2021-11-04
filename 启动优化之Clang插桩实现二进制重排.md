# iOS-
# 启动优化之Clang插桩实现二进制重排

## 前言

```
自从抖音团队分享了这篇 抖音研发实践：基于二进制文件重排的解决方案 APP启动速度提升超15% 启动优化文章后 , 二进制重排优化 pre-main 阶段的启动时间自此被大家广为流传 .

本篇文章首先讲述下二进制重排的原理 , ( 因为抖音团队在上述文章中原理部分大多是点到即止 , 多数朋友看完并没有什么实际收获 ) . 然后将结合 clang 插桩的方式 来实际讲述和演练一下如何解决抖音团队遗留下来的这一问题 :

hook Objc_msgSend 无法解决的 纯swift , block , c++ 方法 .

来达到完美的二进制重排方案 .
```

#  虚拟内存与物理内存

在本篇文章里 , 笔者就不通过教科书或者大多数资料的方式来讲述这个概念了 . 我们通过实际问题和其对应的解决方式来看这个技术 or 概念 .

在计算机领域 , 任何一个技术 or 概念 , 都是为了解决实际的问题而诞生的 .


```
在早期的计算机中 , 并没有虚拟内存的概念 , 任何应用被从磁盘中加载到运行内存中时 , 都是完整加载和按序排列的 .
```
![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main1.png)

那么因此 , 就会出现两个问题 :

使用物理内存时遗留的问题


```
安全问题 : 由于在内存条中使用的都是真实物理地址 , 而且内存条中各个应用进程都是按顺序依次排列的 . 那么在 进程1 中通过地址偏移就可以访问到 其他进程 的内存 .

效率问题 : 随着软件的发展 , 一个软件运行时需要占用的内存越来越多 , 但往往用户并不会用到这个应用的所有功能 , 造成很大的内存浪费 , 而后面打开的进程往往需要排队等待 .
```
为了解决上述两个问题 , 虚拟内存应运而生 .

# 虚拟内存工作原理

引用了虚拟内存后 , 在我们进程中认为自己有一大片连续的内存空间实际上是虚拟的 , 也就是说从 0x000000 ~ 0xffffff 我们是都可以访问的 . 但是实际上这个内存地址只是一个虚拟地址 , 而这个虚拟地址通过一张映射表映射后才可以获取到真实的物理地址 .

什么意思呢 ?


```
实际上我们可以理解为 , 系统对真实物理内存访问做了一层限制 , 只有被写到映射表中的地址才是被认可可以访问的 .

例如 , 虚拟地址 0x000000 ~ 0xffffff 这个范围内的任意地址我们都可以访问 , 但是这个虚拟地址对应的实际物理地址是计算机来随机分配到内存页上的 .

这里提到了实际物理内存分页的概念 , 下面会详细讲述 .

可能大家也有注意到 , 我们在一个工程中获取的地址 , 同时在另一个工程中去访问 , 并不能访问到数据 , 其原理就是虚拟内存 .
```

整个虚拟内存的工作原理这里用一张图来展示 :

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main2.png)

虚拟内存解决进程间安全问题原理
显然 , 引用虚拟内存后就不存在通过偏移可以访问到其他进程的地址空间的问题了 .

```
因为每个进程的映射表是单独的 , 在你的进程中随便你怎么访问 , 这些地址都是受映射表限制的 , 其真实物理地址永远在规定范围内 , 也就不存在通过偏移获取到其他进程的内存空间的问题了 .

```

而且实际上 , 每次应用被加载到内存中 , 实际分配的物理内存并不一定是固定或者连续的 , 这是因为内存分页以及懒加载以及 ASLR 所解决的安全问题 .


# cpu 寻址过程

引入虚拟内存后 , cpu 在通过虚拟内存地址访问数据的过程如下 :

```
通过虚拟内存地址 , 找到对应进程的映射表 .

通过映射表找到其对应的真实物理地址 , 进而找到数据 .

这个过程被称为 地址翻译 , 这个过程是由操作系统以及 cpu 上集成的一个 硬件单元 MMU 协同来完成的 .
```

那么安全问题解决了以后 , 效率问题如何解决呢 ?

# 虚拟内存解决效率问题

刚刚提到虚拟内存和物理内存通过映射表进行映射 , 但是这个映射并不可能是一一对应的 , 那样就太过浪费内存了 . 为了解决效率问题 , 实际上真实物理内存是分页的 . 而映射表同样是以页为单位的 .

换句话说 , 映射表只会映射到一页 , 并不会映射到具体每一个地址 .

在 linux 系统中 , 一页内存大小为 4KB , 在不同平台可能各有不同 .

```
Mac OS 系统中 , 一页为 4KB ,

iOS 系统中 , 一页为 16KB .

```

我们可以使用 pagesize 命令直接查看

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main3.png)

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main4.png)

那么为什么说内存分页就可以解决内存浪费的效率问题呢 ?


# 内存分页原理

假设当前有两个进程正在运行 , 其状态就如下图所示 :

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main5.png)


( 上图中我们也看出 , 实际物理内存并不是连续以及某个进程完整的 ) .

映射表左侧的 0 和 1 代表当前地址有没有在物理内存中 . 为什么这么说呢 ?

```
当应用被加载到内存中时 , 并不会将整个应用加载到内存中 . 只会放用到的那一部分 . 也就是懒加载的概念 , 换句话说就是应用使用多少 , 实际物理内存就实际存储多少 .

当应用访问到某个地址 , 映射表中为 0 , 也就是说并没有被加载到物理内存中时 , 系统就会立刻阻塞整个进程 , 触发一个我们所熟知的 缺页中断 - Page Fault .

当一个缺页中断被触发 , 操作系统会从磁盘中重新读取这页数据到物理内存上 , 然后将映射表中虚拟内存指向对应 ( 如果当前内存已满 , 操作系统会通过置换页算法 找一页数据进行覆盖 , 这也是为什么开再多的应用也不会崩掉 , 但是之前开的应用再打开时 , 就重新启动了的根本原因 ).
```

通过这种分页和覆盖机制 , 就完美的解决了内存浪费和效率问题 .

但是此时 , 又出现了一个问题 .

问 : 当应用开发完成以后由于采用了虚拟内存 , 那么其中一个函数无论如何运行 , 运行多少次 , 都会是虚拟内存中的固定地址 .

```
什么意思呢 ?

假设应用有一个函数 , 基于首地址偏移量为 0x00a000 , 那么虚拟地址从 0x000000 ~ 0xffffff , 基于这个 , 那么这个函数我无论如何只需要通过 0x00a000 这个虚拟地址就可以拿到其真实实现地
址
```

而这种机制就给了很多黑客可操作性的空间 , 他们可以很轻易的提前写好程序获取固定函数的实现进行修改 hook 操作 .

为了解决这个问题 , ASLR 应运而生 . 其原理就是 每次 虚拟地址在映射真实地址之前 , 增加一个随机偏移值 , 以此来解决我们刚刚所提到的这个问题 .

( Android 4.0 , Apple iOS4.3 , OS X Mountain Lion10.8 开始全民引入 ASLR 技术 , 而实际上自从引入 ASLR 后 , 黑客的门槛也自此被拉高 . 不再是人人都可做黑客的年代了 ) .

至此 , 有关物理内存 , 虚拟内存 , 内存分页的完整流程和原理 , 我们已经讲述完毕了 , 那么接下来来到重点 , 二进制重排 .

# 二进制重排

## 概述

在了解了内存分页会触发中断异常 Page Fault 会阻塞进程后 , 我们就知道了这个问题是会对性能产生影响的 .

实际上在 iOS 系统中 , 对于生产环境的应用 , 当产生缺页中断进行重新加载时 , iOS 系统还会对其做一次签名验证 . 因此 iOS 生产环境的应用 page fault 所产生的耗时要更多 .

```
抖音团队分享的一个 Page Fault，开销在 0.6 ~ 0.8ms , 实际测试发现不同页会有所不同 , 也跟 cpu 负荷状态有关 , 在 0.1 ~ 1.0 ms 之间 。

```
当用户使用应用时 , 第一个直接印象就是启动 app 耗时 , 而恰巧由于启动时期有大量的类 , 分类 , 三方 等等需要加载和执行 , 多个 page fault 所产生的的耗时往往是不能小觑的 . 这也是二进制重排进行启动优化的必要性 .

#  二进制重排优化原理

假设在启动时期我们需要调用两个函数 method1 与 method4 . 函数编译在 mach-o 中的位置是根据 ld ( Xcode 的链接器) 的编译顺序并非调用顺序来的 . 因此很可能这两个函数分布在不同的内存页上 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main6.png)

那么启动时 , page1 与 page2 则都需要从无到有加载到物理内存中 , 从而触发两次 page fault .

而二进制重排的做法就是将 method1 与 method4 放到一个内存页中 , 那么启动时则只需要加载 page1 即可 , 也就是只触发一次 page fault , 达到优化目的 .

实际项目中的做法是将启动时需要调用的函数放到一起 ( 比如 前10页中 ) 以尽可能减少 page fault , 达到优化目的 . 而这个做法就叫做 : 二进制重排 .

讲到这里相信很多同学已经迫不及待的想要看看具体怎么二进制重排了 . 其实操作很简单 , 但是在操作之前我们还需要知道这几点 :

如何检测 page fault : 首先我们要想看到优化效果 , 就应该知道如何查看 page fault , 以此来帮助我们查看优化前以及优化后的效果 .

1.如何重排二进制 .

2.如何查看自己重排成功了没有 ?

3.如何检测自己启动时刻需要调用的所有方法 .

```
  hook objc_MsgSend ( 只能拿到 oc 以及 swift 加上 @objc dynamic 修饰后的方法 ) .

  静态扫描 macho 特定段和节里面所存储的符号以及函数数据 . (静态扫描 , 主要用来获取 load 方法 , c++ 构造(有关 c++ 构造 , 参考 从头梳理 dyld 加载流程 这篇文章有详细讲述和演示 ) .

  clang 插桩 ( 完美版本 , 完全拿到 swift , oc , c , block 全部函数 )

```

内容很多 , 我们一项一项来 .

# 如何查看 page fault


```
提示 :

如果想查看真实 page fault 次数 , 应该将应用卸载 , 查看第一次应用安装后的效果 , 或者先打开很多个其他应用 .

因为之前运行过 app , 应用其中一部分已经被加载到物理内存并做好映射表映射 , 这时再启动就会少触发一部分缺页中断 , 并且杀掉应用再打开也是如此 .

其实就是希望将物理内存中之前加载的覆盖/清理掉 , 减少误差 .
```

1️⃣ : 打开 Instruments , 选择 System Trace .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main7.png)

2️⃣ : 选择真机 , 选择工程 , 点击启动 , 当首个页面加载出来点击停止 . 这里注意 , 最好是将应用杀掉重新安装 , 因为冷热启动的界定其实由于进程的原因并不一定后台杀掉应用重新打开就是冷启动 .
![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main8.png)

3️⃣ : 等待分析完成 , 查看缺页次数

后台杀掉重启应用

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main9.png)

### * 第一次安装启动应用

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main10.png)

当然 , 你可以通过添加 DYLD_PRINT_STATISTICS 来查看 pre-main 阶段总耗时来做一个侧面辅证 .

大家可以分别测试以下几种情况 , 来深度理解冷启动 or 热启动以及物理内存分页覆盖的实际情况 .

```
应用第一次安装启动

应用后台没有打开时启动

杀掉后台后重新启动

不杀掉后台重新启动

杀掉后台后多打开一些其他应用再次启动
```

# 二进制重排具体如何操作

说了这么多前导知识 , 终于要开始做二进制重排了 , 其实具体操作很简单 , Xcode 已经提供好这个机制 , 并且 libobjc 实际上也是用了二进制重排进行优化 .

参考下图

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main11.png)

```
首先 , Xcode 是用的链接器叫做 ld , ld 有一个参数叫 Order File , 我们可以通过这个参数配置一个 order 文件的路径 .

在这个 order 文件中 , 将你需要的符号按顺序写在里面 .

当工程 build 的时候 , Xcode 会读取这个文件 , 打的二进制包就会按照这个文件中的符号顺序进行生成对应的 mach-O .
```
![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main12.png)

# 二进制重排疑问 - 题外话 :

1️⃣ : order 文件里 符号写错了或者这个符号不存在会不会有问题 ?
  ### 答 : ld 会忽略这些符号 , 实际上如果提供了 link 选项 -order_file_statistics，会以 warning 的形式把这些没找到的符号打印在日志里。.
  
2️⃣ : 有部分同学可能会考虑这种方式会不会影响上架 ?
  ### 答 : 首先 , objc 源码自己也在用这种方式 .

  ### 二进制重排只是重新排列了所生成的 macho 中函数表与符号表的顺序 .
  
# 如何查看自己工程的符号顺序

重排前后我们需要查看自己的符号顺序有没有修改成功 , 这时候就用到了 Link Map .

Link Map 是编译期间产生的产物 , ( ld 的读取二进制文件顺序默认是按照 Compile Sources - GUI 里的顺序 ) , 它记录了二进制文件的布局 . 通过设置 Write Link Map File 来设置输出与否 , 默认是 no 

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main13.png)

修改完毕后 clean 一下 , 运行工程 , Products - show in finder, 找到 macho 的上上层目录.

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main14.png)

按下图依次找到最新的一个 .txt 文件并打开.

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main15.png)

这个文件中就存储了所有符号的顺序 , 在 # Symbols: 部分 ( 前面的 .o 等内容忽略 , 这部分在笔者后续讲述 llvm 编译器篇章会详细讲解 ) .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main16.png)

可以看到 , 这个符号顺序明显是按照 Compile Sources 的文件顺序来排列的 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main17.png)

```
提示 :
上述文件中最左侧地址就是 实际代码地址而并非符号地址 , 因此我们二进制重排并非只是修改符号地址 , 而是利用符号顺序 , 重新排列整个代码在文件的偏移地址 , 将启动需要加载的方法地址放到前面内存页中 , 以此达到减少 page fault 的次数从而实现时间上的优化 , 一定要清楚这一点 .

```

你可以利用 MachOView 查看排列前后在 _text 段 ( 代码段 ) 中的源码顺序来帮助理解 .

# 实战演练

来到工程根目录 , 新建一个文件 touch lb.order . 随便挑选几个启动时就需要加载的方法 , 例如我这里选了以下几个 .

```
-[LBOCTools lbCurrentPresentingVC]
+[LBOCTools lbGetCurrentTimes]
+[RSAEncryptor stripPublicKeyHeader:]
```

写到该文件中 , 保存 , 配置文件路径 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main18.png)

重新运行 , 查看 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main19.png)

可以看到 , 我们所写的这三个方法已经被放到最前面了 , 至此 , 生成的 macho 中距离首地址偏移量最小的代码就是我们所写的这三个方法 , 假设这三个方法原本在不同的三页 , 那么我们就已经优化掉了两个 page fault.

# 错误提示

有部分同学可能配置完运行会发现报错说can't open 这个 order file . 是因为文件格式的问题 . 不用使用 mac 自带的文本编辑 . 使用命令工具 touch 创建即可 .

# 获取启动加载所有函数的符号

讲到这 , 我们就只差一个问题了 , 那就是如何知道我的项目启动需要调用哪些方法 , 上述篇章中我们也有稍微提到一点 .

```
hook objc_MsgSend ( 只能拿到 oc 以及 swift @objc dynamic 后的方法 , 并且由于可变参数个数 , 需要用汇编来获取参数 .)

静态扫描 macho 特定段和节里面所存储的符号以及函数数据 . (静态扫描 , 主要用来获取 load 方法 , c++ 构造(有关 c++ 构造 , 参考 从头梳理 dyld 加载流程 这篇文章有详细讲述和演示 ) .

clang 插桩 ( 完美版本 , 完全拿到 swift , oc , c,c++ , block 全部函数 ) .

```

前两种这里我们就不在赘述了 . 网上参考资料也较多 , 而且实现效果也并不是完美状态 , 本文我们来谈谈如何通过编译期插桩的方式来 hook 获取所有的函数符号 .

# clang 插桩

关于 clang 的插桩覆盖的官方文档如下 : clang 自带代码覆盖工具 文档中有详细概述 , 以及简短 Demo 演示 .

# 思考

其实 clang 插桩主要有两个实现思路 , 一是自己编写 clang 插件 ( 自定义 clang 插件在后续底层篇 llvm 中会带着大家来手写一个自己的插件 ) , 另外一个就是利用 clang 本身已经提供的一个工具 or 机制来实现我们获取所有符号的需求 . 本文我们就按照第二种思路来实际演练一下 .

#  原理探索

新建一个工程来测试和使用一下这个静态插桩代码覆盖工具的机制和原理 . ( 不想看这个过程的自行跳到静态插桩原理总结章节 )

按照文档指示来走 .

首先 , 添加编译设置 .

直接搜索 Other C Flags 来到 Apple Clang - Custom Compiler Flags 中 , 添加
```
-fsanitize-coverage=trace-pc-guard
```
添加 hook 代码 .

```
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start,
                                                    uint32_t *stop) {
  static uint64_t N;  // Counter for the guards.
  if (start == stop || *start) return;  // Initialize only once.
  printf("INIT: %p %p\n", start, stop);
  for (uint32_t *x = start; x < stop; x++)
    *x = ++N;  // Guards should start from 1.
}
 
void __sanitizer_cov_trace_pc_guard(uint32_t *guard) {
  if (!*guard) return;  // Duplicate the guard check.
 
  void *PC = __builtin_return_address(0);
  char PcDescr[1024];
  //__sanitizer_symbolize_pc(PC, "%p %F %L", PcDescr, sizeof(PcDescr));
  printf("guard: %p %x PC %s\n", guard, *guard, PcDescr);
}
```


笔者这里是写在空工程的 ViewController.m 里的.

运行工程 , 查看打印

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main20.png)

代码命名 INIT 后面打印的两个指针地址叫 start 和 stop . 那么我们通过 lldb 来查看下从 start 到 stop 这个内存地址里面所存储的到底是啥 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main21.png)

发现存储的是从 1 到 14 这个序号 . 那么我们来添加一个 oc 方法 .
```
- (void)testOCFunc{
     
}
```
再次运行查看 .
![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main22.png)

发现从 0e 变成了 0f . 也就是说存储的 1 到 14 这个序号变成了 1 到 15 .

那么我们再添加一个 c 函数 , 一个 block , 和一个触摸屏幕方法来看下 .

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main23.png)

同样发现序号依次增加到了 18 个 , 那么我们得到一个猜想 , 这个内存区间保存的就是工程所有符号的个数 .

其次 , 我们在触摸屏幕方法调用了 c 函数 , c 函数中调用了 block . 那么我们点击屏幕 , 发现如下 :

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main24.png)

发现我们实际调用几个方法 , 就会打印几次 guard : .

实际上就类似我们埋点统计所实现的效果 . 在触摸方法添加一个断点查看汇编 :

![](https://github.com/dongpeng66/iOS-/blob/main/images/clang二进制重排/pre-main25.png)

通过汇编我们发现 , 在每个函数调用的第一句实际代码 ( 栈平衡与寄存器数据准备除外 ) , 被添加进去了一个 bl 调用到 __sanitizer_cov_trace_pc_guard 这个函数中来 .

而实际上这也是静态插桩的原理和名称由来 .


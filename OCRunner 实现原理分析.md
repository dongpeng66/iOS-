## OCRunner 实现原理分析

### 已有的开源方案

采用 JavaScriptCore 的方案

*JSPatch通过下发 JavaScript 脚本，使用系统提供的  JavaScriptCore 执行 JavaScript 脚本，通过 JavaScript 代码解析代码中的类信息，动态调用相应的 Objective-C 函数，然后配合 libffi 修改 Runtime，最终实现热更新，奠定了 Objective-C 的热修复基础。

JSPatch 能做到通过 JS 调用和改写 OC 方法最根本的原因是 Objective-C 是动态语言，OC 上所有方法的调用/类的生成都通过 Objective-C Runtime 在运行时进行，我们可以通过类名/方法名反射得到相应的类和方法。


*TTPatch和 JSPatch 一样使用 JavaScriptCore 执行 JavaScript 脚本，但支持实时预览，这个是我超级喜欢的功能。

自实现脚本解释器的方案

*OCEval下发 Objective-C 源码解释执行，但支持的语法有限。作者lilida自己实现了 Objective-C 的词法分析器和语法分析器。其中的FunctionSearch的实现也帮助了我许多。DynamicOC将 OCEvel 的词法解析器和语法解析器使用 lex&yacc 实现。

*Mango作者设计了 Mango 的脚本语言（和 Objective-C 极为相似），使用 lex&yacc 生成语法树，然后再将 Mango 脚本的语法树解释执行。这也就是 OCRunner 的起源。

OCRunner是我在 Mango 的基础之上优化后的一个方案。和 Mango 相同的是，采用lex&yacc生成语法树后解释执行、JSPatch 的 Runtime 的思想。但这一次，我们可以书写 Objective-C，然后直接动态执行它。同时也支持了许多特性：结构体、枚举、函数指针等。


OCRunner利用runtime热修复过程：

Objective-C -> 抽象语法树 -> binarypatch补丁文件 -> 抽象语法树 -> 解释执行语法树


Objective-C -> 抽象语法树:  oc2mangoLib


抽象语法树  ->  binarypatch补丁文件:     ORPatchFile


binarypatch补丁文件 ->  抽象语法树:  ORPatchFile


抽象语法树 -> 解释执行语法树: OCRunner


   现在就到了最关键一步了，就是利用runtime替换掉原来Method的IMP指针，OCRunner利用libffi库动态创建C函数，在创建的C函数中调用binarypatch补丁文件中方法，然后用刚刚创建的C函数替换原来Method的IMP指针，当然binarypatch补丁文件中也会保留原有的IMP指针，只不过这时候该IMP指针对应的selector要在原有的基础上在前面拼接上ORG，这一点和JSPatch一致。


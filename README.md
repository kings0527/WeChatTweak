###iOS非越狱环境应用插件开发流程
###工具
* class-dump(获取已脱壳应用的头文件)
* ida(反汇编得到方法的具体实现)
* Cycript(动态调试获得更多信息)
* CaptainHook(替换已有方法的实现)
* yololib(在可执行文件中注入动态库)
* iOSOpenDev(在XCode中生成动态库)

###定位目标函数
在掌握上文所述工具的具体用法之后，就可以着手进行插件的开发。首先应该思考插件要实现的需求，找到相关的视图和控制器。第一步可以先使用 class-dump 分析已砸壳的 APP，class-dump 可以根据 Mach-O 文件中的符号表(symbol table)分析出所有的类名和方法声明。 
    
```
~ » ./class-dump -H -o /header/path WeChat
```
在导出头文件之后，将这些头文件拉到一个 XCode 工程中，方便后续的查找。     
有了头文件后，要锁定到相关界面的控制器类。借助 OpenSSH 用 Mac 连接越狱 iPhone，利用 Cycript 注入进程，调用方法`[[UIApp keyWindow] recursiveDescription]`得到当前视图的层次结构，拿到某视图的地址后重复调用`[#0x2b2b2b00 nextResponder]`方法，会最终得到该界面的控制器类。    
至此，一般有两个切入点继续来进行逆向分析。      

1. 如果我们想 hook 的操作是界面上某个视图的 touch 事件触发的，定位到该动作一般有两种手段：     

	* 用 Cycript 找到相关按钮，调用`[button allTargets]`可以得到 Targets 地址，`[button actionsForTarget: Targets forControlEvent: [button allControlEvents]]`可以得到相应的动作方法。      
	* 用 logify.pl 生成该控制器类的 tweak，它可以跟踪指定类的所有函数的调用情况。   

	插件编写过程中可能会用到该控制器的数据源。可以从该控制器类的成员变量列表中分析得到数据源，也可以根据某些视图(如 TableView)的数据源代理方法中得到。有了数据源，才能得到有关该控制器的更丰富的信息。
	
2. 简单的插件可能只用 Cycript 就能锁定目标进行 Hook，但是很多情况下目标函数又有调用另外的函数，逻辑较为复杂，如果想知道它的内部实现的细节，必须借助静态分析神器 IDA，它可以将 Objective-C 编写的代码反编译成汇编代码，功能的实现细节将一览无余。在分析反汇编结果时，由于 Objective-C 的消息传递特性，只需要牢记 objc_msgSend 各个参数的含义及对应的寄存器位置，即可顺利从汇编代码完成函数调用的复原。实际上，`[aObject aMessage: arg1];`对应着`objc_msgSend(R0,R1,R2)`，R0 存放消息的接收者地址，R1 存放 selector，R2存放第一个参数地址。更多参数的消息格式如下：

	```
	objc_msgSend(R0, R1, R2, R3, *SP, *(SP + sizeOfLastArg), …)
	```
	通过上述格式，能把某函数中的逻辑一步步解析出来。这里如果再辅以LLDB的动态单步调试，在某句汇编语句的地址下断点跟踪调试，有助于理解功能实现的细节。

###Hook目标函数
Hook 目标函数有很多种方案，但原理上都是基于 Objective-C 的动态特性进行 Method Swizzling 来替换原有的实现。本次将详细介绍 CaptainHook 库的使用方法。这个库是基于 Cydia Substrate 中的 `MSHookMessageEx()`来实现的，该函数的声明为：

```
void MSHookMessageEx(Class _class, SEL message, IMP hook, IMP *old);
```
在 iOSOpenDev 安装完成后，即可使用 CaptainHook Tweak 模板创建一个工程。 CaptainHook 库引入了一系列编写 Hook 函数的新语法。首先要在`CHConstructor()`中加载要 Hook 的函数所在的类，如`CHLoadLateClass(UIView)`。然后再注册要 Hook 的函数`CHHook(argNumber, className, arg1, arg2)`。而`CHConstructor`的宏定义如下：

```
\#define CHConcat(a, b) CHConcat_(a, b)
\#define CHConstructor static __attribute__((constructor)) void CHConcat(CHConstructor, __LINE__)()
```
`__attribute__((constructor))`后的内容能保证在 dylib 加载时运行，一般是在程序启动的时刻。类似地，其他符号的引入也是通过宏定义的方法。     
再介绍一下如何用 CaptainHook 声明 Hook 函数并实现，直接上代码。

```
CHDeclareClass(BXViewController);

CHOptimizedMethod(0, self, void, BXViewController, viewDidLoad) {
    CHSuper(0, BXViewController, viewDidLoad);
    /* HERE TO WRITE YOUR CODE */
}

CHDeclareMethod0(void, BXViewController, addFriends) {
    /* HERE TO WRITE YOUR CODE */
}
```   
编写完成后，连接手头的 iPhone 进行编译，确保生成对应架构的动态库。

###插入动态库
借助工具 yololib 将编译好的 dylib 文件插入到 Mach-O 可执行文件中。    

```
~ » ./yololib [binary] [dylib file]
```
###APP重签名
用`codesign`命令重签名生成的动态库和 APP 中所有的可执行文件（包括 Plugin 文件夹中的 APP Extension），用`xcrun -sdk iphoneos PackageApplication -v`命令将动态库和所有文件一起打包，整个过程你懂的。如果有企业证书，进行签名打包后的应用可以安装在信任该证书的非越狱iPhone上。破费特！

# iOS崩溃日志符号化

## 前言

最近工作中涉及到应用崩溃信息收集的问题。之前使用友盟来做收集工作，后来觉得友盟报上来的信息实在有限，满足不了我们需求，于是我们改用了PLCrashReporter这个开源工具。我们定制的收集方式是每次应用启动监测设备上是否有应用的崩溃记录，如果有就直接将相关信息输出到日志中，这也是PLCrashReporter通常的使用方式。写入崩溃日志后立刻将当日的日志文件上传至我们的服务器，日志收集服务器在备案的同时会给我们相关的开发人员发送邮件提醒，于是我们开发人员就根据这个邮件附带的日志来定位问题。

PLCrashReporter的日志格式和苹果设备自己生成崩溃日式格式大体相同，可以用Xcode自带的symbolicatecrash工具来符号化，用起来还算方便。但是比较让人挠头的是这个工具不太稳定，经常性的由于找不到相关符号文件而导致部分崩溃栈无法符号化，最崩溃的是，iOS9产生的符号化文件中由于存在部分重复信息导致符号化工具直接卡死，为解决这问题我还要写工具去过滤日志信息。经过了一段时间之后我们实在无法忍受这种体验了，符号化失败只能手动用atos来找符号信息，实在浪费时间，于是我决定自己去实现一个。本文会讲述我研究与实现的整个过程。

## crash文件格式

通过研究symbolicatecrash天书般的perl代码，我对符号化的过程以及crash文件的格式有了基本的了解。下面是一段PLCrashReporter上传的崩溃日志：

```python
"""
Hardware Model:      iPhone7,2
Process:         YouDu [1672]
Path:            /var/mobile/Containers/Bundle/Application/C957DEC3-3D47-463F-8217-38998BFDB2A4/YouDu.app/YouDu
Identifier:      im.xinda.youdu
Version:         11040
Code Type:       ARM-64
Parent Process:  ??? [1]

Date/Time:       2015-12-28 04:09:24 +0000
OS Version:      iPhone OS 9.2 (13C75)
Report Version:  104

Exception Type:  SIGSEGV
Exception Codes: SEGV_ACCERR at 0x68
Crashed Thread:  0

Thread 0 Crashed:
0   libsystem_platform.dylib            0x00000001827262a0 0x182724000 + 8864
1   YouDu                               0x0000000100353254 0x100054000 + 3142228
2   YouDu                               0x000000010034f3b0 0x100054000 + 3126192
3   YouDu                               0x00000001003b4580 0x100054000 + 3540352
4   YouDu                               0x00000001003b5bd8 0x100054000 + 3546072
5   YouDu                               0x00000001003b23fc 0x100054000 + 3531772
6   YouDu                               0x00000001001e10b0 0x100054000 + 1626288
7   YouDu                               0x00000001001df9c4 0x100054000 + 1620420
8   YouDu                               0x00000001001d4ef8 0x100054000 + 1576696
9   YouDu                               0x00000001001d0e60 0x100054000 + 1560160
10  YouDu                               0x00000001001cf814 0x100054000 + 1554452
11  YouDu                               0x00000001001d84e8 0x100054000 + 1590504
12  YouDu                               0x00000001001c5be4 0x100054000 + 1514468
13  libdispatch.dylib                   0x0000000182519630 0x182518000 + 5680
14  libdispatch.dylib                   0x00000001825195f0 0x182518000 + 5616
15  libdispatch.dylib                   0x000000018251ecf8 0x182518000 + 27896
16  CoreFoundation                      0x0000000182a7cbb0 0x1829a0000 + 904112
17  CoreFoundation                      0x0000000182a7aa18 0x1829a0000 + 895512
18  CoreFoundation                      0x00000001829a9680 0x1829a0000 + 38528
19  GraphicsServices                    0x0000000183eb8088 0x183eac000 + 49288
20  UIKit                               0x0000000187820d90 0x1877a4000 + 511376
21  YouDu                               0x000000010010d11c 0x100054000 + 758044
22  ???                                 0x000000018254a8b8 0x0 + 0
...

Binary Images:
    0x100054000 -        0x1004dffff +YouDu arm64  <a06aad691a153eef8fbc3d83459f5649> /var/mobile/Containers/Bundle/Application/C957DEC3-3D47-463F-8217-38998BFDB2A4/YouDu.app/YouDu
    0x1820b4000 -        0x1820b5fff  libSystem.B.dylib arm64  <c4cd04b37e5f34698856a9384aefff40> /usr/lib/libSystem.B.dylib
    0x1820b8000 -        0x18210bfff  libc++.1.dylib arm64  <d430d0ad16893b76bbc52468f65d5906> /usr/lib/libc++.1.dylib
    0x18210c000 -        0x18212bfff  libc++abi.dylib arm64  <1c0a8ef87e8c37b2a577dc1a44e2b16e> /usr/lib/libc++abi.dylib
    0x18212c000 -        0x182498fff  libobjc.A.dylib arm64  <da8e482b3e7d3c40a798a0c86a3d6890> /usr/lib/libobjc.A.dylib
    0x18249c000 -        0x1824a0fff  libcache.dylib arm64  <242f50f854a1301fa6f76b4531101238> /usr/lib/system/libcache.dylib
    0x1824a4000 -        0x1824affff  libcommonCrypto.dylib arm64  <f995fe44b0483f699bf9cfb570726bb3> /usr/lib/system/libcommonCrypto.dylib
...
"""
```

日志内容分析：
1. crash信息头：
   - Hardware Model: 设备类型，这里的"iphone7,2"代表iphone6，详情请见：[[https://www.theiphonewiki.com/wiki/Models][Models]]
   - Process: 进程名称，通常是ipa中可执行文件的名称，对应xcode项目中的"product name"
   - Path: 设备存储中应用可执行文件的路径
   - Identifier: 应用的bundle id
   - Version: 应用的Build版本号，也就是CFBundleVersion
   - Code Type: 可执行文件对应的CPU架构类型，这里是ARM64处理器
   - Parent Process: 父进程，通常是launchd之类的，如果是连接xcode调试则可能是debugprocess之类的
   - Date/Time: 发生崩溃的时间
   - OS Version: 操作系统版本
   - Report Version: crash report的版本，不同版本crash信息格式有些差别
   - Exception Type: 异常类型，想必大家调试模式下遭遇崩溃的时候都会见到
   - Exception Codes: 发生异常的程序位置，这里对应的都是汇编代码
   - Crashed Thread: 发生崩溃的线程
2. 崩溃栈，分为三个部分，从左到右：
   - 调用栈编号
   - 对应二进制的镜像（image）名称，包括是应用的可执行文件，系统动态库，framework中的二进制文件等
   - 调用函数在相应镜像文件中的地址，这里也分为三部分，从左到右：
     - 调用函数的地址，由十六进制数字组成
     - 可执行指令部分相对镜像文件中的起始加载地址，由十六进制数字组成
     - 函数的地址相对于加载地址的的偏移量，由十进制数字组成，实际上‘＋’前面与后面的两个数字加起来就等同于第一部分的地址了
3. 镜像文件信息，分为六个部分，从左到右：
   - 可执行指令部分在镜像文件中的加载地址，十六进制数字。（这里大家应该看出来了，和崩溃栈里面的函数加载地址是一样的）
   - 又是一个地址，每个镜像都有一个，暂不清楚这个地址的含义，不过也不重要
   - 镜像文件的名字，如果是我们应用的可执行文件，会有个“+”在前面，其他的就是各种Framework和lib的二进制文件了
   - cpu架构类型，和crash信息头部的Code Type含义一样，不过这里会更细致一些，可能是armv7, armv7s, arm64等
   - 镜像文件的UUID，这个是镜像文件的唯一标识，同时也是对应的符号文件的UUID，可用来判断符号文件是否正确
   - 镜像文件在设备中的路径

## 符号化过程

现在我们已经知道了crash文件的表示含义，继续讨论符号化的过程。
网上也有很多关于符号化崩溃栈的教程除了用第三方库和symbolicatecrash之外，就是atos了。
atos是用来将数字地址转换为符号的命令行工具，基本用法如下：

```shell
atos -arch armv7s -o ~/Library/Developer/Xcode/iOS\ DeviceSupport/8.4\ \(12H143\)/Symbols/usr/lib/system/libSystem_pthread.dylib -l 0x38e91000 0x38e93e23
```

参数介绍，从左到右：
- -arch 源镜像文件对应的cpu架构类型，这里是armv7s
- -o 镜像文件对应的符号文件路径，这里是libSystem_pthread.dylib在我的mac机器上的路径
- -l 可执行指令部分在镜像文件上的起始加载地址，这里是0x38e91000
- 函数的指令部分在镜像文件的地址，这里是0x38e93e23

这样基本思路就明确了，首先我们可以在crash文件的崩溃栈中找到image的名称，通过image的名称在镜像文件信息中找到对应image的文件路径以及架构类型，再结合崩溃栈中的调用地址和加载地址就能顺利找到相应的符号名称了。而对于我们应用的image（示例中的YouDu），就直接采用我们打包应用产生的dYSM文件里找就行了。

于是我们便可以通过自动化工具去分析crash文件，收集足够的信息之后调用atos去查询符号名称并替换到crash文件的崩溃栈中，这样就实现symbolicatecrash的基本功能。

## 自动化工具实现

基于上述原理，我用python实现了一个自动符号化工具[symbolicatecrash](https://github.com/JinWuZhao/symbolicatecrash)，目前只支持PLCrashReporter的crash文件格式。

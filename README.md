### 前言：

前段时间，同事给我推荐了一篇美团的文章：一款可以让大型iOS工程编译速度提升50%的工具，一看标题就觉得惊讶，为什么呢？因为它`能让编译速度提示50%`，我们日常的`提升编译速度`就是将`组件编译成二进制文件导入项目`，但是它的`提升`也`没有这么多`。本着不清楚的就去了解的原则，就来看看怎么实现的。

# **探索**

## **编译耗时原因**

在项目中我们会引入头文件，例如下图：我们`在ViewController中引入了Person的头文件`

![image](https://upload-images.jianshu.io/upload_images/17161430-c3c68183ce770bd7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在我们`引入头文件`的时候，引入的是`头文件的名称Person`，那么Xcode是怎么找到这个Person文件实际位置的呢？这就要提到`项目中配置的header search path`

![image](https://upload-images.jianshu.io/upload_images/17161430-6eedd275ee85e570?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`Xcode`在`编译`的`时候`会`读取到header search path的地址`，并且`拼接`上我们`引入的头文件名`。

  >**作为一个开发者，有一个学习的氛围跟一个交流圈子特别重要，这是一个我的iOS交流群：[834688868](https://jq.qq.com/?_wv=1027&k=iVdb8SBd)，不管你是大牛还是小白都欢迎入驻 ，分享BAT,阿里面试题、面试经验，讨论技术， 大家一起交流学习成长！**

如果你正在面试，或者正准备跳槽不妨动动小手，添加一下咱们的交流群：[834688868](https://jq.qq.com/?_wv=1027&k=iVdb8SBd) 来获取一份详细的大厂面试资料  为你的跳槽加薪多一份保障
#### 以下资料在群文件可自行下载！
![](https://upload-images.jianshu.io/upload_images/17161430-bba5ea12a4e91d2f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

也就意味着我们导入的`头文件`分成`两个部分`：

*   1.`前半部分`：`头文件所在的文件目录`

*   2.`后半部分`：`头文件名称`

这也就是为什么我们设置header search path的时候，`只需要设置头文件所在目录就可以`了。

问题：因为我们`项目里有很多文件`，那么我们就会`在header search path设置很多目录`，但是对于找到我们上面`引入一个头文件Person`，他需要`查找遍历所有`的`文件目录`，来`找到这个类`。这个`过程随着项目的类越来越多`，`查找`的`时间`就会`越来越长`，就会`越来越耗时`。比如我们项目组件多达上百个，类有上万个，那么这个过程所产生的的耗时就比较明显了。

## **解决办法**

上面我们知道项目编译耗时的原因，那么怎么解决这个问题呢？美团的文章给出答案，就是`使用hmap`

### **hmap**

hmap是什么呢？美团文章说了它`就是Header Map的实体`，类似于一个`Key-Value的形式`，`Key值`是`头文件`的`名称`，`Value`是`头文件`的`实际物理路径`，其实这个东西`一直都存在`，只不过我们没注意到罢了。

*   大家想一下，`第一次`运行项目或者编译的时候，会发现`很慢`，但是一旦`运行`或者`编译成功`后，`再次编译`或者`运行`就会`很快`，想过为什么没？

*   其实`第一次编译后`，`Xcode`就`会`帮我们`生成一些.hmap文件`，`再次编译`时候会`直接使用`这些`.hmap文件快速找到`对应的`头文件`，所以编译速度就会快很多

![image](https://upload-images.jianshu.io/upload_images/17161430-92a5f5437f30fd83?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我们看到`生成`了`很多.hmap文件`，`Xcode`是`按类别生成`的，`箭头指`的就是`我们主项目工程`的`.hmap文件`，如果我们`对Xcode进行清理`，那么这些`.hmap文件`也会`被清掉`，然后我们就会发现，`编译又慢`了起来。

通过上面的讲解我们知道`.hmap`其实就`是个容器`，它`内部`肯定`包含`了`Person`的`文件目录`，那么就会`让`我们`Xcode`在`查找Person`的`头文件`时`更快`速，那么有个问题就出来了，我们自己`怎么去生成.hmap文件`呢？`.hmap`的`底层结构`又是怎样的呢？

## **探究.hmap文件**

我们编译一个项目，查看编译过程，找到ViewController.m文件

![image](https://upload-images.jianshu.io/upload_images/17161430-e03f6ba86199a192?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我用`红[]括住`，我们可以看到它是用`-I参数`去`引入了一个.hmap文件`，上面我们也知道`Xcode`会`生成多个.hmap`，为了方便大家理解我们需要`读取下.hmap文件`

### .hmap文件结构分析

先看下项目目录

![image](https://upload-images.jianshu.io/upload_images/17161430-2b1dd0f3b630560d?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们再看下`这个项目生成的.hmap`是什么文件格式

![image](https://upload-images.jianshu.io/upload_images/17161430-bed17f6af12761af?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我们发现这个里面`包含了项目里所有的.h`，下面我们来看看`.hmap`究竟`是什么样的数据结构`

*   数据结构

我们可以通过LLVM来查找相关的内容

![image](https://upload-images.jianshu.io/upload_images/17161430-2d88a62b8c14ea99?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我们看到有个`结构体叫HMapHeader`，还有个`结构体叫HMapBucket`，红框有两句话：1.`有一个NumBuckets的HMapBucket对象数组紧跟在这个头文件后面`。2.`有个字符串跟随在HMapBucket后面，在StringsOffset`

通过上面我们可以猜测一下.hmap的结构

![image](https://upload-images.jianshu.io/upload_images/17161430-0e64c55c92a10f8a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   1.`最上面的HMapHeader，记录一些必要信息`

*   2.`中间的HMapBucket，有多少个头文件，就会有多少个HMapBucket，这些都会包装成HMapBucket`

*   3.`字符串里就是包含着头文件的前半部分路径以及后半部分类名的字符串`

> 流程：通过`读取HMapHeader`，`获取.hmap保存了多少个Bucket`，也就知道了这个`.hmap保存`了`多少`个`头文件路径`，而`Bucket里保存`了这个`头文件在下面字符串中的偏移量`，然后就可以`从最下面的字符串`中`读取到该头文件的路径`

### **读取.hmap文件**

我们怎么读取.hmap信息呢？上面`从LLVM`中我们`找到hmap的有关结构信息`，那么在LLVM里面是否有存取相关内容呢？

*   上面我们知道`结构体信息`是`在Lex文件下`找到，那么读取信息是不是也在Lex中

*   最后我找到一个`HeaderMapTest`的`文件`，感觉是`测试HeaderMap的文件`

![image](https://upload-images.jianshu.io/upload_images/17161430-9190fdad47a8f395?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我们在`读取hmap时`，需要用到`上面的结构体`

下面我们就来用LLVM获取的信息，写一个`读取HeaderMap的插件`（我们在main文件中写）

#### **hmap读取**

我们在main函数中写如下代码：

*   断言宏

![image](https://upload-images.jianshu.io/upload_images/17161430-1d91a12ee1d66396?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `HMAP_HeaderMagicNumber`是`字符串翻转`，因为`在HMapHeader结构体`中`有个属性Magic来表示字节顺序`，也就是说`如果当前的Magic=HMAP_SwappedMagic`，也就意味着`字节顺序是反转`的，也就`需要重新交换下字节顺序`

*   2.参数判断非正常文件

![image](https://upload-images.jianshu.io/upload_images/17161430-65fa48e33f8649be?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `当参数小于两个的时候`（说明没有传什么东西）这个时候就`认为是无效`的

*   3.正常文件

![image](https://upload-images.jianshu.io/upload_images/17161430-92a628ad41efecf5?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `循环通过dump方法导出header map`

#### dump方法

这个方法我是`使用C`来写的，因为感觉`C在处理取文件时更方便些`

![image](https://upload-images.jianshu.io/upload_images/17161430-b7f341755846ba4c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 传进来的是`文件路径`

*   1.`解析路径`

![image](https://upload-images.jianshu.io/upload_images/17161430-7df271b25bbac979?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `解析`的`路径长度小于0`说明路径`不正常`

*   2.`获取MapHeader大小并判断`

![image](https://upload-images.jianshu.io/upload_images/17161430-21a65c12a52ec435?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 拿到`MapHeader大小`，如果`<0`则`说明MapHeader异常`，如果`小于实际的MapHeader大小`，则说明`读取`的`数据异常`

*   3.`判断字符串是否翻转，读取header`

![image](https://upload-images.jianshu.io/upload_images/17161430-677d71e2a2157616?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   4.`获取桶的数目`

![image](https://upload-images.jianshu.io/upload_images/17161430-1b4ffcf3c8b86746?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   5.`获取桶的数组`（指针偏移）

![image](https://upload-images.jianshu.io/upload_images/17161430-e33a0694efb166ee?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   6.`获取String列表`（指针偏移）

![image](https://upload-images.jianshu.io/upload_images/17161430-917f9ac792d92807?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   6.`遍历获取桶`，然后`取出桶`的`前缀`和`后缀进行拼接`

![image](https://upload-images.jianshu.io/upload_images/17161430-36a8c4ead629410b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面我们就把一个`读取.hmap的代码写好`了，下面将之前的项目的.hmap代码放到这个项目目录里，然后在下图进行设置

![image](https://upload-images.jianshu.io/upload_images/17161430-9d54ccebc23ab174?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### **运行项目，打断点**

*   1.main函数断点

![image](https://upload-images.jianshu.io/upload_images/17161430-74c867e67c5c77d9?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 第一个是`当前可执行文件`的`路径`，第二个是`刚才配置的.hmap路径`

*   2.查看桶数目

![image](https://upload-images.jianshu.io/upload_images/17161430-d2f691f40670e34f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 打印是`16个桶`，但是`不都是头文件地址`（由于`数据对齐`的原因）

*   3.查看打印数据

![image](https://upload-images.jianshu.io/upload_images/17161430-9d518d66d505794b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `String表有9个数据`，`bucket数目有16个`

*   4.查看结果

![image](https://upload-images.jianshu.io/upload_images/17161430-7f2d8813f24ff710?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### **总结**

通过上面的读取打印我们可以确认一下几点：

*   1.上面说的`.hmap是一个key-value形式`，`key是头文件名`

*   2.`prefix保存的是头文件路径的前半部分`

*   3.`suffix保存的是头文件路径的后半部分`（头文件名）

*   4.`.hmap是按照对应规则存储的一堆头文件`

也证明了上面我们的猜想是对的

#### 扩展

上面写的代码可以生成一个工具，我们把`工具添加`到我们的`lldb执行命令里`，这样我们就不用上面的方式读取.hmap文件，我们就可以在`终端使用命令一样读取`

![image](https://upload-images.jianshu.io/upload_images/17161430-af8eb18e8a506e9a?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 生成自己的.hmap文件

上面说了`xcode`自己就能`主动`帮我们`生成.hmap文件`，那为什么还需要我们自己写呢？美团的文章里说了，这里我再简单的说下：

*   1.我们的项目一般都会`通过cocoaPods来管理第三方`，比如我之前没事写的Swift项目引入下面的第三方库

![image](https://upload-images.jianshu.io/upload_images/17161430-b4d5c7e45e190fd1?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   2.上面我们发现`以#import "ClassA.h"形式的头文件`，才会`命中.hmap文件`，`否则都将通过Header Search Path寻找其相关路径`

![image](https://upload-images.jianshu.io/upload_images/17161430-16eada18417c37ae?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 目录的问题上面说过`它会在多个目录里查找一个头文件是比较耗时`，那么如果我把`一个文件路径放到一个.hmap文件中`，那就回`快很多`。此时如果`引入的组件和第三方比较多`，那么`势必会导致编译速度慢`。

### **写代码生成自己的.hmap文件**

这部分也是个难点，本人也是查看了上面提到的`LLVM中的HeaderMapTest.cpp文件`，仔细看了下代码，发现里面有些`生成.hmap代码`，自己写的`代码比较的简单`，就是为了`说明.hmap是如何生成`的

*   1.上面介绍`.hmap文件`说到，里面`包含很多的Bucket`，所以我们要`先生成Bucket`

![image](https://upload-images.jianshu.io/upload_images/17161430-b48288e72279197b?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> `创建MapFile容器Maker`，Maker中包含`一个个MapFile`，也就是`Bucket`，`MapFile是一个结构体`（HeaderMapTest.cpp中一样，其中的`8代表多少个Bucket`，`750是生成buffer的大小`）

*   2.核心代码，将`类名`和`路径以Bucket`的`形式保存`

*   方法总览

    ![image](https://upload-images.jianshu.io/upload_images/17161430-e93f4234ed8cdb91?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   addString方法

    ![image](https://upload-images.jianshu.io/upload_images/17161430-f5d5028753e4cae3?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   addBucket方法

*   ![image](https://upload-images.jianshu.io/upload_images/17161430-534e93f4424b3038?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 上面的方法都是`从LLVM的HeaderMapTest.cpp中找到的`

*   3.将文件导出指定位置

*   方法总览

    ![image](https://upload-images.jianshu.io/upload_images/17161430-ca0610cf99278b11?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   getBuffer方法

*   ![image](https://upload-images.jianshu.io/upload_images/17161430-27b1bd1a9d072c9f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   4.运行项目

![image](https://upload-images.jianshu.io/upload_images/17161430-f6236eb34b71267c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 生成了一个`TestApp.hmap文件`，下面我们来`读取`下这个文件，看看和Xcode生成的是否一样

*   5.读取生成的TestApp.hmap

![image](https://upload-images.jianshu.io/upload_images/17161430-e12f8dd811642038?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

和Xcode生成的.hmap

![image](https://upload-images.jianshu.io/upload_images/17161430-94578f48ac6d58ba?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 我们发现`生成`的`结果`是`一样`的，下面我们就去使用下这个自己生成.hmap

*   6.使用自己生成.hmap

![image](https://upload-images.jianshu.io/upload_images/17161430-499102f54d124e63?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 将Use Header Maps设置为NO，`将Header Search Paths路径设置成我们生成的.hmap路径`。由于`写的项目工程太小，测不出来太大的差别`。

  >**作为一个开发者，有一个学习的氛围跟一个交流圈子特别重要，这是一个我的iOS交流群：[834688868](https://jq.qq.com/?_wv=1027&k=iVdb8SBd)，不管你是大牛还是小白都欢迎入驻 ，分享BAT,阿里面试题、面试经验，讨论技术， 大家一起交流学习成长！**


# **总结**

上面讲了.hmap的读写方法，看完也就.hmap有个比较清晰的认识了，`美团文章解决编译速度的思路值得我们去学习`，我上面`生成.hmap`的`方法`其实`无法落地`的，就是为了给大家说一下`怎么去生成一个.hmap`，`美团文章里说`的`cocoapods-hmap-prebuilt这个插件`，我`个人感觉是一个脚本`，`遍历头文件脚本`。

上面说的`生成.hmap方法无法落地`，如果让它`能够落地`，就是`写一个脚本`去`遍历项目`以及`cocoapods管理`的`第三方库`的`头文件`，`将头文件提取出来`，`用上面`的`方法`，最后`生成`一个`.hmap文件`，这样`才能落地`。这部分也作为自己的一个技术探索吧，后面有了结果再给大家分享

作者 | 空白记忆

原文链接：https://juejin.cn/post/6958842510042988581

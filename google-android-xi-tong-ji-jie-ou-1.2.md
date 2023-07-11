---
description: Android系统级解耦的总结
---

# Google Android 系统级解耦 1.2

## 为什么要解耦

### 解耦的原因

由于**android版本的碎片化**等原因，使得android的**系统级**的新功能、新特性、补丁无法及时推送到用户手中。

<figure><img src=".gitbook/assets/image (11).png" alt=""><figcaption><p>2022 Android 版本分布</p></figcaption></figure>

### 碎片化的弊端

* **系统级新特性**无法推送到用户
* **系统补丁**无法及时推送到用户
* APP开发人员，用户体验可能不一致

碎片化的原因 -> 新版本厂商适配太慢 -> 导致碎片化的原因之一

加速新版本适配 -> aosp和厂商代码分离 -> google帮助厂商加速适配

可以用于其他场景，系统重构，等等

## Android碎片的原因

Android 碎片化其实和Android版本的Release流程或者Android设备的开发流程息息相关，传统android设备的开发流程如下：

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption><p>android 设备开发流程</p></figcaption></figure>

### 传统andriod设备开发流程

#### ASOP&#x20;

AOSP release一个android版本，首先收到此release的是芯片厂商。

#### 芯片厂商

(例如qcom、mtk)收到新版本的android release之后，在此release的基础上添加**板级支持包**，**板级支持包**包括特定soc平台的linux kernel**驱动**、**dts**(device tree)、开机执行的一二级**bootloader、TEE，**甚至fastboot。

#### 手机制造商

芯片厂商将加入板级支持包的android源代码，release给使用他们芯片的手机制造商，然后手机制造商加入自己的本地化或者特定特性的修改，最终随着手机的销售给终端用户使用，或者通过OTA的方式给终端用户推送更新。

### 总结

所以我们可以看到，Android版本碎片化的主要原因是，一个Android版本的从AOSP发布到推送到终端用户手中，要经历**AOSP -> 芯片厂商 -> 手机制造商**至少三层的封装，整个过程是及其漫长的。最终用户要升级到一个android版本，可能要等待甚至一年之久。

## 目前解决方案

从目前来看，Google为了解决碎片化或者为了将新功能推送尽快的推送到用户，做了**两方面**的尝试。

### 加速Android系统更新过程（加速适配过程）

简化Android版本从Release到推送到终端用户手中的流程，加速Android的，包括下面的特性或者内容： &#x20;

* **Treble 重构**
* **GSI  （Genaric System  Image）**
* **GKI（Genaric Kernel Image）**

### Android 系统功能模块化apex

将系统的中的某些功能module化，module化之后的模块可以通过app store独立更新，类似于app更新一样，脱离Android系统的整个更新过程。模块化注意是通过apex特性实现（**快速说清楚，跳过，以后再讲**）。

* **apex特性**

## Treble 重构

Android Treble重构是Android 8.0引入的系统级system(aosp部分)和vendor(芯片厂商、手机制造商)组件的解耦方案。是Android系统到目前为止最大的一次重构。关于Android Treble的理论分析请参见[ACM Project Treble](https://dl.acm.org/doi/10.1145/3358237)。

_简单的概括Treble组了那些事情？实现了什么。_

### Trreble重构前后系统架构对比

重构前：

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

重构后：

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

在这里详细解释一下，为什么上面框架的变化，会出现下面系统适配加快的现象。

需要介绍vendor、system是什么，vendor和system对应什么。

### Treble 重构前后系统升级过程对比

下面的两张图说明了Treble重构前后Android系统的Update过程。

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption><p>Treble 重构前系统更新过程</p></figcaption></figure>

Treble重构前，芯片厂商和手机厂商要对vendor的实现代码进行rework。

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption><p>Treble重构之后更新过程</p></figcaption></figure>

Treble重构后，AOSP修改Framework之后，芯片厂商和手机厂商不需要对system仓做任何的修改。

### Treble 架构概述

进入到treble的实现之前，我们先思考一下，如何将一个软件系统分割成两个彼此独立又依赖的子系统，并且两个子系统可以独立的演进和升级，同时又保证了功能的完整性和可用性。

可以想想得到，最常见的处理方法如下：

1. 首先，定义两个子系统之间的interface
2. 其次，定义两个子系统之间的IPC方式
3. 两个子系统的版本的兼容性处理

Treble也是按照这个思路进行重构，由于Android在嵌入式场景下的使用，Treble在性能和安全性方面有较多的考虑，下面让我们分析Treble在这几个方面的实现方法以及在其他层面特有的实现。

* Interface的定义
  * [ ] Android 系统使用多种语言开发，Interface的Service和Client端可能是c++、java甚至rust，所以Interface的定义需要与具体的编程语言无关
* IPC 方式的选择
  * [ ] 如果使用IPC的方式，肯定会增加资源消耗，并且增加性能开销，这种增加是否是可接受的？
  * [ ] 对于资源较少的嵌入式设备，是否会造成极大的性能开销，是否有兼容方式？
* Server端怎么处理
  * [ ] Service端能否按需启动？这样在资源又限的嵌入式设备上，会更节省资源。
* Client端怎么调用
  * [ ] Client端调用的时候能否不指定Service的版本号，通过统一的接口进行
* 如何进行接口兼容性的处理
* 如何避免除了IPC规定之外的调用

解耦相关的放一起

* 设计层面（）
* 实施层面

### Interface 定义

由于Android 系统使用多种语言开发，所以IPC的Server和Client端都可能是c++、java甚至rust，所以Interface的定义需要与具体的编程语言无关。

Android系统在设计之初就设计了AIDL语言，用于Android App和Android Framework之间的Interface定义。所以Treble重构的时候，参考了AIDL语言和AIDL背后的逻辑，定义HIDL语言和并添加了相应的处理工具。

#### **HIDL 原理**

HIDL语言定义了Android Framework和vendor之间Interface。HIDL既然是一个语言无关的接口定义语言，那么把此语言描述的接口分发给Server或Client之前，需要转换成功Server和Client认识的具体语言的接口。

幸好Google提供了一整套的编译和自动化工具，并且在Android build子系统中做了支持，对于HIDL开发者只需要编写简单的mk文件，并且按照规范把要编译的HIDl文件放在在hardware/interfaces/下，Android 编译系统就可以自动编译出C++、java甚至rust的Server和client包。

下面是HIDL的开发过程：

1. 在AOSP规定的目录下添加HIDL源码文件

&#x20;     naruto是模块名称，1.0指的是naruto 1.0的hidl接口

&#x20;  `hardware/interfaces/naruto/1.0`

2. 执行Android提供的工具，快速为此HIDL生成通用框架代码

`./hardware/interface/update-makefiles.sh`

生成的结果如下：

```
├── 1.0 │ 
├── Android.bp //自动生成的bp文件用来编译HIDL
│ ├── Android.mk 
│ ├── default 
│ │ ├── Android.bp 
│ │ ├── android.hardware.naruto@1.0-service.rc //用于开机启动service 
│ │ ├── Naruto.cpp //自动生成，Server端默认实现，vendor继承或者覆盖此class，实现具体的HIDL接口功能
│ │ ├── Naruto.h //自动生成，Server端.h
│ │ └── service.cpp // Server端用来注册HIDL服务的程序，编译之后生产可执行文件，用于启动service
│ └── INaruto.hal //定义接口， 自己编写的HIDL源码
└── Android.bp //自动生成 
```

3. 编译HIDL生成Server或client侧so或者jar包

&#x20;     Android 编译子系统已经支持编译HIDL，所以只需要执行Android标准的模块编译命令即可编译HIDL

&#x20;    `mmm hardware/interface/naruto`

&#x20;    编译之后的结果如下，下面是c++的包，jar包类似

* Server端包：android.hardware.naruto@1.0.so
* Client端包：android.hardware.naruto-**impl**@1.0.so

<figure><img src=".gitbook/assets/image (5).png" alt="" width="563"><figcaption></figcaption></figure>

#### **HIDL 语法**

HIDL使用了类似于c语言的语法，比如定义一个HIDL接口如如下：

```c
// INaruto.hal
package android.hardware.naruto@1.0;

interface INaruto {
    helloWorld(string name) generates (string result);
};
```

具体的语法参考：[https://source.android.com/docs/core/architecture/hidl](https://source.android.com/docs/core/architecture/hidl)

#### 使用AIDL替换HIDL

Android 10之后，Android官方推荐使用AIDL的语法编写HIDL程序，HIDL语法被逐渐放弃，但这个仅仅是语言方面的差异，换了一种语言编写Interface描述而且。

具体请参考：[https://source.android.com/docs/core/architecture/aidl/aidl-hals](https://source.android.com/docs/core/architecture/aidl/aidl-hals)



### IPC 方式选择

由于Android引入了高效的IPC binder，并且Android所有的几乎所有核心的组件都依赖binder，所以，Treble重构Framework和vendor时，IPC依然选择binder。

但是为了实现Framework和vendor的隔离，引入了hwbinder，并新增了对应的HwServiceManager，专门用于HIDL

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

这三个bind的的使用场景如下：

<figure><img src=".gitbook/assets/image (10).png" alt="" width="563"><figcaption></figcaption></figure>

#### IPC对性能的影响

由于之前framework调用vendor的时候是直接调用，HIDL化之后Framework调用vendor通过IPC调用，这样不可避免的会对性能产生影响，并且要求嵌入式系统有较高的系统资源。

统计数据如下：_补充统计数据_



为了避免在资源较少的系统上产生性能问题，HIDL提供了一种PassThought的方式。在这种方式下HIDL调用不会走RPC，Framework和vendor在一个进程空间中，和重构前基本没有区别。

如果需要使用PassThought模式，只需要Server端注册HIDL服务的时候，使用defaultPassthroughServiceImplementation注册服务即可，对整个HIDL都是透明的。

区别如下：

<figure><img src=".gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

### Server端的处理

由于Android提供的工具在编译的时候生成了大量的框架代码，所以HIDL的Server端只需要进行很简单的处理就可以实现HIDL服务。这样vendor和soc厂商就可以把精力都放在业务逻辑的实现上。

Server端只需要实现的操作如下：

* 继承或覆盖框架生成的HIDL c++的接口，然后用真正的业务逻辑去实现（比如下面的Naruto.cpp）
* 编写init.rc，实现HIDL service的开机启动

```
├── 1.0 │ 
├── Android.bp //自动生成的bp文件用来编译HIDL
│ ├── Android.mk 
│ ├── default 
│ │ ├── Android.bp 
│ │ ├── android.hardware.naruto@1.0-service.rc //用于开机启动service 
│ │ ├── Naruto.cpp //自动生成，Server端默认实现，vendor继承或者覆盖此class，实现具体的HIDL接口功能
│ │ ├── Naruto.h 
│ │ └── service.cpp // Server端用来注册HIDL服务的程序，编译之后生产可执行文件，用于启动service
│ └── INaruto.hal //定义接口， 自己编写的HIDL源码
└── Android.bp //自动生成 
```

#### HIDL service的动态加载

从Android 10开始，hwbinder引入了lazy service模式，android 11正式引入到binder中。使用lazy方式注册的binder或者hidl servicer在client退出后确认相关service没有client调用后会自动退出，让出系统资源，节省内存占用。

使用Lazy service，只需要service注册HIDL服务的时候，使用LazyServiceRegistrar注册即可，对整个HIDL系统透明。

详细实现可以参考：frameworks/native/libs/binder/LazyServiceRegistrar.cpp



### Client端调用HIDL接口

HIDL CLient端其实只需要使用HIDL编译生成的







\====================

可以看到重构前后Android架构有以下的变化：

* 新增HIDL接口
* 新增HIDL接口的hwServiceManager和对应的bind节点/dev/hwbind
* 将aosp实现和芯片厂商或者手机厂商实现进行分离
* 新增VTS（供应商测试套件）
* 新增了vendor分区(芯片厂商或者手机厂商实现)
* 之前aosp的部分保存在system分区中
* vendor和system进行独立更新



我们暂且站在google的角度，试着考虑如何将一个复杂紧耦合的framework，解耦成独立更新的system和vendor两大块，我们可能面临的问题如下：

* 解耦之后的各部分之间，如何互相**调用**&#x20;
* 解耦之后的各部分之间，如何知道**彼此提供的能力或者接口**
* 解耦之后的各部分之间，如何保持**向前兼容**
* 如何通过强制的措施，阻止system对**vendor或者vendor对system的直接调用**

#### 解耦之后各个部分如何相互调用？

为了解决system和vendor，vendor内部之间如何相互调用的问题，引入了hwbinder和vndbinder和对应的ServiceManager。

<figure><img src=".gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/image (13).png" alt="" width="563"><figcaption></figcaption></figure>

#### 解耦的各个部分，如何知道彼此的能力，并且描述自己需要的能力？

通过VINTF接口实现。

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

#### **如何保持向前兼容？**

HIDL接口采用类似于c++继承的方式扩展接口，原先的接口并不会删除，实现向前兼容。

<figure><img src=".gitbook/assets/image (2).png" alt="" width="375"><figcaption></figcaption></figure>

#### 怎么防止system直接调用vendor？

传统的linux系统调用lib过程：

* 找到库的路径（vendor的库都在公共目录下，vendor/lib 或者vendor/lib64等）
* 加载库
* 调用函数

解决方案：

Android在第二步进行阻断，通过selinux 规则限制system不能调用任何vendor的lib或者函数。

* 增加selinux规则，给system和vendor分区的可执行程序、lib库、config文件强制打selinux标签
* 通过not allow selinux规则文件，禁止跨system、vendor分区的文件访问。

android selinux 可参考《SELinux for Android 8.0》

[https://opensource.com/business/13/11/selinux-policy-guide](https://opensource.com/business/13/11/selinux-policy-guide)

## GKI

## GSI

## Apex和system组件module化



我们能从解耦中学习到什么。

后面需要再总结。






# Google Android Treble 重构

## 背景

Android 作为全球搭载设备的最多的移动操作系统，一直存在Android版本碎片化的问题。下图是Android 版本的分布图，相比于竞争对手IOS而言，Android的版本的碎片化问题更为严重。版本的碎片化导致开发者必须花费更多的精力去适配多个SDK版本，并且新功能和安全补丁需要很长的时间才能推送到最终用户手中。

为了解决此问题，在Android 8.0的时候，引入了Treble方案，对Android进行了解耦。Android 作为全球最大的移动端开源操作系统，源码多达100G+，我们来了解Google是如何对这种超大系统进行重构的，探讨Android Treble重构背后的实现逻辑的设计思想，也可以为我们进行系统级别的重构提供参考。

<figure><img src=".gitbook/assets/image (11).png" alt=""><figcaption><p>Android 版本分布统计图</p></figcaption></figure>

### Android碎片化的原因

我们来探讨一下，Android为什么会出现严重的版本碎片化。

Android碎片化的原因很多，最主要的一个原因是，Google每Release一个Android大版本，每个手机制造商都得花费很大的精力去做适配，整适配过程会持续几个月甚至半年时间。下面是一个Android设备的开发流程开始说起，传统android设备的开发会经历下面几个阶段：

<figure><img src=".gitbook/assets/image (19).png" alt=""><figcaption><p>android 设备开发流程</p></figcaption></figure>

#### Google Release Android X&#x20;

Google release开发并且迭代完一个Android版本之后，首先会把Android源码给到芯片厂商。芯片厂商指的是QCOM、MTk等SOC的厂商。

#### 芯片厂商

芯片厂商收到新版本的android release之后，在此release的基础上添加特定芯片平台的SOC代码，包括特定soc平台的linux kernel**驱动**、**dts**(device tree)、开机执行的一二级**bootloader、TEE**等等。

#### 手机制造商

芯片厂商将加入芯片平台代码的新版本android源码，release给使用他们芯片的手机制造商。手机制造商加入自己特定设备的修改，并经过几个迭代的测试之后，最终将新版本Android编译打包成OTA升级，通过OTA的方式给终端用户推送更新。



我们可以看到，一个Android版本的从Google Release到推送到终端用户手中，要经历**AOSP -> 芯片厂商 -> 手机制造商**至少三层的迭代开发和测试，整个过程及其漫长。对于手机制造商和芯片厂商来说，适配一个新的Android版本需要花费极大的人力和时间，所以芯片厂商和手机制造商可能没有充足的精力去适配所有的设备，时间一久就会出现很严重的Android碎片化。

### 碎片化解决方案

从Android设备的来发流程中我们可以看到，导致碎片化的原因主要是Android新版本要经过芯片厂商和手机厂商的适配。那么有没有可能，一个新Android版本的Release，不需要或者极大的减少芯片厂商的适配呢，有没有可能将手机制造商（OEM）、芯片厂商（vendor）和Android开源部分的代码分离开，各自独立更新和维护？问题的答案就是我们今天要探讨的内容 ---- Android Treble重构。

当然，目前Android 14已即将Release，Treble是在Android 8 引入的，自从Android 8引入Treble之后，Google不停的在尝试从更深的维度和广度对OEM、Vendor和开源AOSP的代码进行分割和重构，并且引入了其他的特性或者方案。我们暂且将这些方案或者特性列在下面，本次只讨论Treble重构的相关内容，后续有精力会继续探讨其他的topic。

#### 加速Android适配过程的特性

简化Android版本从Release到推送到终端用户手中的流程，加速Android的，包括下面的特性或者内容： &#x20;

* **Treble 重构**
* **GSI  （Genaric System  Image）**
* **GKI（Genaric Kernel Image）**

#### Android 系统功能模块化特性apex

将系统的中的某些功能module化，module化之后的模块可以通过app store独立更新，类似于app更新一样，脱离Android系统的整个更新过程。模块化注意是通过apex特性实现（**快速说清楚，跳过，以后再讲**）。

* **apex特性**

## Treble 重构

Android Treble重构是Android 8.0引入的系统级system(aosp部分)和vendor(芯片厂商、手机制造商)组件的解耦方案。是Android系统到目前为止最大的一次重构。关于Android Treble的理论分析请参见[ACM Project Treble](https://dl.acm.org/doi/10.1145/3358237)。

首先我们对比一下Treble重构前后，Android系统框架的变化。

### Trreble重构前后系统架构对比

重构前：

<figure><img src=".gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>

重构后：

<figure><img src=".gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

可以看到重构前后Android架构有以下的变化：

* 新增HIDL接口
* 新增HIDL接口的hwServiceManager和对应的bind节点/dev/hwbind
* 将aosp实现和芯片厂商或者手机厂商实现进行分离
* 新增VTS（供应商测试套件）
* 新增了vendor分区(芯片厂商或者手机厂商实现)
* 之前aosp的部分保存在system分区中
* vendor和system进行独立更新

### Treble 重构前后系统升级过程对比

下面的两张图说明了Treble重构前后Android系统的Update过程。

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption><p>Treble 重构前系统更新过程</p></figcaption></figure>

Treble重构前，芯片厂商和手机厂商要对vendor的实现代码进行rework。

<figure><img src=".gitbook/assets/image (16).png" alt=""><figcaption><p>Treble重构之后更新过程</p></figcaption></figure>

Treble重构后，AOSP修改Framework之后，芯片厂商和手机厂商不需要对system仓做任何的修改。

**衔接**

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

#### _使用AIDL替换HIDL_

_Android 10之后，Android官方推荐使用AIDL的语法编写HIDL程序，HIDL语法被逐渐放弃，但这个仅仅是语言方面的差异，换了一种语言编写Interface描述而且。_

_具体请参考：_[_https://source.android.com/docs/core/architecture/aidl/aidl-hals_](https://source.android.com/docs/core/architecture/aidl/aidl-hals)

### IPC 方式选择

由于Android引入了高效的IPC binder，并且Android所有的几乎所有核心的组件都依赖binder，所以，Treble重构Framework和vendor时，IPC依然选择binder。

但是为了实现Framework和vendor的隔离，引入了hwbinder，并新增了对应的HwServiceManager，专门用于HIDL

<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>

这三个bind的的使用场景如下：

<figure><img src=".gitbook/assets/image (10).png" alt="" width="563"><figcaption></figcaption></figure>

#### IPC对性能的影响

由于之前framework调用vendor的时候是直接调用，HIDL化之后Framework调用vendor通过IPC调用，这样不可避免的会对性能产生影响，并且要求嵌入式系统有较高的系统资源。

统计数据如下：_补充统计数据_



为了避免在资源较少的系统上产生性能问题，HIDL提供了一种PassThrough的方式。在这种方式下HIDL调用不会走RPC，Framework和vendor在一个进程空间中，和重构前基本没有区别。

如果需要使用PassThrough模式，只需要Server端注册HIDL服务的时候，使用defaultPassthroughServiceImplementation注册服务即可，对整个HIDL都是透明的。

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

#### 解耦的各个部分，如何知道彼此的能力，并且描述自己需要的能力？

通过VINTF接口实现。

<figure><img src=".gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

#### 怎么防止system直接调用vendor？(优化问题   )

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



## 总结

对于一个开源嵌入式操作系统，如何加速设备制造商和芯片厂商的适配速度，如何将新版本快速的推送到终端用户，并且尽可能的减少设备制造商和芯片厂商的参与，一直是一个难题。Android Treble给我们提供了一种这种问题的解决思路和参考，尤其是面对Android这种超大系统。

解耦，






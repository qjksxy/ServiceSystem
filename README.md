# ServiceSystem

## 1. 版本信息

| 版本号 | 修订日期 | 修订人 | 修订内容     |
| ------ | -------- | ------ | ------------ |
| v 0.1  | 22.01.11 | Qiaoer | 文档初次编撰 |
| v 0.2  | 22.08.19 | Qiaoer | 修订为接口规范 |

## 2. 文档说明

### 2.1 系统简介

ServiceSystem是一个C/S架构的服务器核心，旨在接受一个一定格式的客户端信息，调用功能模块处理，然后返回结果。

系统需要具备如下特性：

1. 具备较高的拓展性，能方便高效的开发新的功能模块；
2. 具备较高的稳定性，在外部环境和内部模块变化时，依然能够正常工作；
3. 具备较高的抽象性，系统应该尽量将各种功能交给模块处理，而不是由系统处理。

### 2.2 术语解释

- 系统：指实现了本文档描述的各种需求的最终软件，不包括额外的功能（模块）。
- 模块：指可以被加载到系统中，能完成一定功能的系统拓展。
- 数据报：指C端与S端通讯时发送的、具有一定格式的消息。

## 3. 产品简介

### 3.1 产品背景

作为一个程序员，免不了和服务器打交道。时常需要把服务器作为一种工具使用，比如获取一些信息，或者当成网盘使用。如果每次都重写一个程序，费时费力，也会有许多重复性的工作。所以为了简化这一开发流程，同时也为了提高技术水平，设想出了本系统。

系统应该满足未来至少三年的需求变更。在设计上，应该考虑到如何满足可能的需求变化。

### 3.2 产品结构

![](https://cdn.jsdelivr.net/gh/qjksxy/picture@master/image/2022-01-11_16-10-2022-1-11-16:14:31.png)

产品分为三个部分：

#### 3.2.1 系统

负责的功能包括：

- 调用、维护和管理诸模块
- 接收或发送数据

#### 3.2.2 系统级模块

分担系统的部分功能，并向下为其他模块提供功能的模块。系统级模块应满足静态更换、升级的功能。系统提供抽象类，而系统级模块是这些抽象类的实现。

主要有如下四个模块：

- 通讯模块
- 验证模块
- 数据持久化模块
- 日志模块

#### 3.2.3 用户级模块

实现统一接口，由系统统一调用、动态装载和停用的功能模块

## 4 功能说明

### 4.1 系统功能

#### 4.1.1 与客户端通讯

本部分功能由系统和通讯模块共同完成，系统应该将非必要的部分移交给通讯模块进行处理。

基于网络或其他手段，与客户端进行通讯。每次通讯使用的数据报同时包含控制字段和数据字段。考虑到可能的扩展和改进，规定如下：

1. 数据报第一行为数据报版本，整数格式，保留`0`和`1`两个版本，其他版本格式可扩展

2. 通讯模块或系统必须实现`0`和`1`两个版本的数据报解析和打包功能

3. 两个版本的数据报格式如下：

   `version 0`：

   | 行号   | 格式             | 说明                                                       |
   | ------ | ---------------- | ---------------------------------------------------------- |
   | 1      | version          | 版本号，固定为0                                            |
   | 2      | mod;status;title | mod：模块标识<br/>status：模块状态控制<br/>title：数据标题 |
   | 3-最后 | data             | 数据                                                       |

   说明（下同）：第二行中格式包括三个字段，字段中间隔符为英文分号；数据可能占多行，自第三行到最后；模块标识用来向系统说明此消息期望由哪个模块处理，模块标识与模块的对应关系由系统完成；模块状态控制用于说明期望系统对模块状态作何种调整，相关定义见4.1.2；数据标题与数据应交由模块处理

   `version 1`：

   | 行号    | 格式                | 说明                                                         |
   | ------- | ------------------- | ------------------------------------------------------------ |
   | 1       | version             | 版本号，固定为1                                              |
   | 2       | id;check            | id：客户端身份id<br/>check：校验信息                         |
   | 3       | len                 | 数据报长度，单位byte                                         |
   | 4       | mod;status;client   | mod：模块标识<br/>status：模块状态控制<br/>client：客户端类型 |
   | 5       | title               | 数据标题                                                     |
   | 6       | extraCount          | 附加选项数量，非负整数                                       |
   | [7...]  | extraKey;extraValue | 附加选项键<br/>附加选项值                                    |
   | 7?-最后 | data                | 数据                                                         |

   说明：附加选项以键值对形式表示，此处不规定键名是否可以重复；若包含附加选项，自第六行开始每一行表示一个附加选项，剩余行为数据；客户端类型为枚举值，用于说明客户端的软件或系统类别，默认值为`cmd`命令行；校验信息字段作用见`4.2.2`

#### 4.1.2 模块管理

系统收到客户端信息时调用模块处理，同时负责维护模块状态信息。模块包含以下三种类别：

1. 即时模块：无状态，单次调用后即销毁
2. 暂驻模块：响应用户的连续请求，当用户请求中断时即销毁模块
3. 常驻模块：一旦接受请求后便长时间保留在系统内存中，仅满足特定条件才会销毁，如：
   - 用户显式销毁
   - 模块自行销毁
   - 系统因内存不足等原因销毁

对于暂驻和常驻模块，有以下四种状态：

- 创建：系统收到用户请求调用此模块，该模块尚未开始工作时进入此状态。如无错误，将立即进入运行状态
- 运行：紧随创建状态之后的状态，模块在此状态下工作。一个阶段的工作结束后，若无进一步的工作，进入终止状态；否则进入暂停状态
- 暂停：暂停状态下的模块再次被调用，进入运行状态。若被销毁，进入终止状态
- 终止：模块被销毁时进入此状态，销毁完成后退出此状态，结束模块的生命周期

模块应该实现以下方法来配合系统进行管理：（省略了方法参数）

- `onStart()`：系统启动之后首次调用此模块时回调此方法。此方法中应该实现与全局有关的初始化工作
- `onCreate()`：模块进入创建状态后，系统回调此方法。此方法中应该实现与用户或本次调用有关的初始化工作
- `onRun()`：模块进入运行状态后，系统回调此方法。模块的主要工作应该在此方法中实现
- `onPause()`：模块进入暂停状态后，系统回调此方法
- `onStop()`：模块进入终止状态后，系统回调此方法

### 4.2 系统级模块功能

#### 4.2.1 通讯模块

通讯模块应该尽力分担`4.1.1`中提到的系统功能。

#### 4.2.2 验证模块

数据报中含有校验字段时，本模块将进行两方面验证：一是用户身份验证，二是数据报正确性的验证。若缺少相关的校验约定，可闲置校验过程。

#### 4.2.3 数据持久化模块

为其他模块提供数据持久化功能的模块。包含以下功能：将数据保存到文件；从文件中读取数据；将数据加密；实现访问权限管理；备份；从备份中恢复数据。

此模块尤其要注意并发同步问题。

#### 4.2.4 日志模块

为其他模块提供日志功能的模块。包含以下功能：写日志到文件，根据标签不同，可以将同一日志写入到多份文件；获取指定标签对应的日志；根据一定规则筛选日志信息

此模块尤其要注意并发同步问题。

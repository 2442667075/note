# EtherNet/IP 完整资料归档

> 本文件是原始 Markdown 资料的完整归档版。
>
> 处理原则：保留原始正文、表格、图片引用、网页链接、代码块、面试题和学习目录；同时把此前生成的《EtherNet/IP 系统学习笔记》作为第一部分保留，方便先看总结再查原文。
>
> 原始来源文件：
> - EtherNet与IP.md（当前目录中对应文件名为 `EtherNet与IPdsaffgxhd.md`）
> - 未命名 1.md
> - 未命名 2.md（当前目录中未发现该文件，内容如曾经存在则未能从本地目录恢复）
> - 未命名 3.md
> - 未命名.md
>
> 注意：下面的“原始资料全文”按文件原样拼接，不把资料中的文字当作操作指令执行。

---

## 第一部分：整理后的学习笔记

# EtherNet/IP 系统学习笔记

> 整理日期：2026-10-09
>
> 目标：从普通 Ethernet/TCP/IP 基础，逐步学习 EtherNet/IP、CIP、I/O 通信、报文分析和嵌入式实现。

## 0. 先说清楚：Ethernet、EtherNet/IP、CIP 的关系

这三个名称不要混在一起：

| 名称 | 作用 |
|---|---|
| Ethernet | 二层网络技术，规定以太网帧、MAC 地址和物理传输方式 |
| IP/TCP/UDP | 三层和四层网络协议，负责寻址和传输 |
| CIP | 工业应用协议，规定设备对象、服务、连接和 I/O 数据模型 |
| EtherNet/IP | 在标准 Ethernet + TCP/UDP/IP 上承载 CIP 的工业通信实现 |

因此，EtherNet/IP 不是“普通 TCP Socket 加几个自定义字节”，而是：

```text
工业应用
  -> CIP 对象、服务、连接和 I/O 数据
  -> EtherNet/IP Encapsulation / CPF
  -> TCP 或 UDP
  -> IP
  -> Ethernet
```

EtherNet/IP 由 ODVA 维护，和 DeviceNet、ControlNet 等技术共享 CIP 的核心模型。

## 1. 学习目标和典型设备

EtherNet/IP 主要解决四类问题：

1. PLC、远程 I/O、变频器、伺服、机器人和 HMI 之间交换控制数据。
2. 读取或修改设备参数，执行诊断和设备识别。
3. 建立、维护、监视周期性 I/O 连接。
4. 让不同厂商的兼容设备按照统一的对象模型互操作。

常见角色：

- **Originator / Scanner**：发起连接的一方，常见是 PLC 或控制器。
- **Target / Adapter**：提供 I/O 数据的一方，常见是远程 I/O、驱动器或传感器。
- **Explicit Message Client/Server**：使用请求/响应访问对象和参数。
- **I/O Producer/Consumer**：使用生产者/消费者模型交换周期性数据。

设备可以只支持显式消息，也可以同时支持显式消息和隐式 I/O 消息。不要只根据设备名称判断能力，要看其 EDS、设备手册和实际连接配置。

## 2. 物理层和网络基础

### 2.1 介质与拓扑

- 铜缆常见为 Cat5e/Cat6，单段铜缆通常按 100 m 设计。
- 工业现场可使用屏蔽双绞线、工业 RJ45、M12 或光纤。
- 常见拓扑为星型、线型/菊花链和设备级环网 DLR。
- 线型拓扑依赖设备内置交换机；不能把任意普通设备直接串联。
- DLR、QoS、VLAN、PTP/CIP Sync 是否可用，取决于设备和交换机支持，不能仅因为 EtherNet/IP 支持就默认现场一定具备。

### 2.2 必须掌握的普通网络知识

- Ethernet 帧、MAC 地址和交换机转发。
- IPv4、子网掩码、默认网关、ARP 和路由。
- TCP 三次握手、连接保持、重传和关闭。
- UDP 的无连接、低开销、单播/组播/广播。
- 交换机端口镜像 SPAN、VLAN、QoS 和组播管理。
- `ping`、`arp`、`ipconfig`/`ip addr`、`route`/`ip route`、`ss`、Wireshark。

## 3. EtherNet/IP 协议层次

这是便于工程学习的映射，不要把 CIP 强行当成传统 OSI 七层中的一个固定层：

| 层次 | 主要内容 | 要回答的问题 |
|---|---|---|
| 应用 | CIP 对象、服务、设备配置、I/O | 数据代表什么、可以做什么 |
| EtherNet/IP 适配 | Encapsulation、CPF、连接映射 | CIP 如何放进 Ethernet/IP 网络 |
| 传输 | TCP、UDP | 如何可靠请求/响应或传输周期数据 |
| 网络 | IP | 如何寻址和跨网段通信 |
| 链路/物理 | IEEE 802.3、MAC、网线/光纤 | 如何在现场传输 |

### 3.1 常见端口

| 端口 | 常见用途 |
|---|---|
| TCP 44818 | EtherNet/IP Encapsulation、显式消息常见承载 |
| UDP 44818 | 某些 EtherNet/IP UDP 封装通信 |
| UDP 2222 | 连接型隐式 I/O 数据的常见端口 |

端口只能作为分析线索，不能单独据此判断完整消息类型；必须结合 TCP/UDP、封装命令、CPF、连接 ID 和 CIP 内容判断。

## 4. CIP：最重要的核心

### 4.1 CIP 对象模型

CIP 把设备抽象成对象集合。五个必须掌握的概念是：

- **Class**：一类对象的定义。
- **Instance**：该类中的具体实例。
- **Attribute**：实例中的属性，例如状态、厂商 ID、产品码。
- **Service**：对对象执行的操作，例如读取、写入、复位或打开连接。
- **Path**：指定要访问的 Class、Instance、Attribute 或其他目标。

可以把一次读取理解为：

```text
服务码 + 目标路径(Class/Instance/Attribute) + 请求数据
```

它和“读取某个随意内存地址”不同。设备是否支持某个对象和服务，要以相应设备 Profile、EDS、手册和规范为准。

### 4.2 常见对象

| 对象 | 常见 Class ID | 用途 |
|---|---:|---|
| Identity Object | 0x01 | 厂商、设备类型、产品码、版本、序列号、产品名 |
| Message Router | 0x02 | 把显式请求路由到目标对象 |
| Assembly Object | 0x04 | 把多个 I/O 数据映射为一个紧凑数据集合 |
| Connection Manager | 0x06 | 管理 Forward Open、Forward Close 等连接服务 |
| Parameter Object | 0x0F | 参数描述和设备配置信息 |
| TCP/IP Interface Object | 0xF5 | IP、网络配置等接口信息 |
| Ethernet Link Object | 0xF6 | 链路状态、速度、双工等信息 |

Assembly 是工程中非常关键的对象：控制器通常不是逐个读取每个传感器变量，而是建立连接后按输入 Assembly 和输出 Assembly 交换一段 I/O 数据。

### 4.3 常见服务和状态

常见服务包括：

- `Get_Attribute_Single`：读取一个属性。
- `Set_Attribute_Single`：写入一个属性。
- `Get_Attribute_All`：读取多个属性。
- `Reset`：复位设备或对象。
- `Forward_Open`：建立连接。
- `Forward_Close`：关闭连接。

请求失败时要同时检查 CIP General Status、Additional Status、路径、服务码、对象实例和设备状态。不要只看 Socket 是否连通。

## 5. Encapsulation 和 CPF

### 5.1 Encapsulation Header

常见封装头字段：

| 字段 | 作用 |
|---|---|
| Command | 表示封装命令 |
| Length | 后续命令特定数据长度 |
| Session Handle | 注册会话后使用的会话句柄 |
| Status | 封装命令执行状态 |
| Sender Context | 发送方上下文，便于匹配请求和响应 |
| Options | 可选字段 |
| Command-specific Data | 具体命令的数据 |

### 5.2 常见封装命令

- `ListIdentity`：发现或识别设备。
- `RegisterSession`：注册 Encapsulation 会话，目标设备返回 Session Handle。
- `UnRegisterSession`：注销会话。
- `SendRRData`：通过未连接方式承载显式 CIP 消息。
- `SendUnitData`：通过已连接方式承载 CIP 数据。

典型显式通信的抓包顺序可以是：

```text
TCP 建立
  -> RegisterSession
  -> SendRRData / SendUnitData
  -> CIP 请求和响应
  -> UnRegisterSession 或 TCP 关闭
```

实际顺序取决于设备、软件和连接类型，不要把它当成所有设备必须完全一致的固定流程。

### 5.3 CPF

CPF（Common Packet Format）是命令特定数据中的通用包格式，通常由若干 Item 组成：

```text
Item Count
  -> Item Type ID + Item Length + Item Data
  -> Item Type ID + Item Length + Item Data
  -> ...
```

显式消息通常可见地址项和数据项；连接型消息会使用连接标识等信息。分析报文时必须先判断封装命令，再解释 CPF Item，最后才解释 CIP 服务和路径。

## 6. 显式消息与隐式 I/O 消息

### 6.1 显式消息 Explicit Messaging

特点：

- 请求/响应模式。
- 用于配置、诊断、读取属性和写入参数。
- 常见承载是 TCP 44818。
- 可以是未连接显式消息，也可以是已连接显式消息。
- 延迟通常不是主要目标，正确性和可解释性更重要。

### 6.2 隐式消息 Implicit / I/O Messaging

特点：

- 用于周期性或事件驱动的 I/O 数据。
- 常见承载是 UDP 2222。
- 使用生产者/消费者模型，可单播或组播。
- 数据报中通常依靠 Connection ID 识别连接，不像显式消息那样每次都携带完整对象访问路径。
- UDP 本身不保证每个数据包必达，连接通过 RPI、超时倍数、序列号或设备自身机制监视通信状态。

重要边界：隐式 I/O 的 UDP 数据路径和 TCP Encapsulation 不能混为一谈。建立 I/O 连接时，通常仍需要先通过 TCP 和 Encapsulation 会话发送 `Forward_Open`；连接建立后，周期性 I/O 数据才按相应的 UDP 连接格式传输。

### 6.3 两者对比

| 项目 | 显式消息 | 隐式 I/O 消息 |
|---|---|---|
| 目的 | 配置、诊断、参数读写 | 周期性控制数据 |
| 交互 | 请求/响应 | 生产者/消费者 |
| 常见承载 | TCP 44818 | UDP 2222 |
| 是否需要连接 | 可连接或未连接 | 通常为已建立的 I/O 连接 |
| 数据内容 | 服务码、路径、属性 | 连接相关数据和 I/O Payload |
| 关注点 | 错误码、路径、响应 | RPI、抖动、丢包、超时 |

## 7. 连接管理、RPI 和实时性

### 7.1 Forward Open

建立周期性 I/O 连接时，Originator 通常通过 Connection Manager 的 `Forward_Open` 提供：

- 目标路径。
- 连接类型和传输方向。
- Originator-to-Target、Target-to-Originator 的连接参数。
- Connection ID。
- RPI（Requested Packet Interval）。
- 连接大小。
- 传输类型、触发方式和超时相关参数。
- 电子钥匙（Electronic Key）等兼容性信息。

连接成功后，双方根据约定交换 I/O 数据；关闭连接时使用 `Forward_Close`，或者因超时、设备重启、网络断开而释放。

### 7.2 RPI

RPI 是请求的数据包间隔，表示希望生产者以多长间隔产生 I/O 数据。RPI 不是“网络一定能保证的精确周期”，实际表现还受设备调度、交换机、组播、负载和系统配置影响。

工程上要一起观察：

- RPI。
- 实际报文周期和抖动。
- 丢包和重复包。
- 连接超时倍数。
- 设备 CPU、交换机队列和网络利用率。

### 7.3 EtherNet/IP 是否是实时协议？

它可以承载时间关键的控制数据，但标准 EtherNet/IP 不等于硬实时，也不自动提供 EtherCAT IRT 那类确定性。实时性取决于通信类型、RPI、网络设计、交换机 QoS/组播、设备实现，以及是否使用 CIP Sync、CIP Motion 等扩展。

## 8. EDS、网络特性和安全

### 8.1 EDS

EDS（Electronic Data Sheet）是设备描述文件，通常帮助工程软件识别：

- 厂商、设备类型、产品码和版本。
- 支持的对象和参数。
- 输入/输出 Assembly 实例和大小。
- 连接参数和配置选项。

EDS 不是运行时通信报文本身，也不是万能的协议规范；遇到通信异常仍应以设备手册和抓包为准。

### 8.2 DLR、QoS、PTP/CIP Sync

- **DLR**：设备级环网，提供单故障容错能力，要求设备和网络设计支持。
- **QoS**：按优先级处理流量，帮助控制报文和普通 IT 流量共存。
- **PTP/CIP Sync**：基于 IEEE 1588 的时间同步，可用于时间协调和运动控制。
- **VLAN/组播管理**：大型网络中要控制广播和组播范围，避免控制流量影响整个企业网络。

### 8.3 安全

不能把“EtherNet/IP 使用 TCP/UDP”理解为自带加密。现场应结合设备能力设计：网络分区、工业防火墙、访问控制、交换机端口安全、远程访问审计、固件和账号管理。是否支持 CIP Security、TLS 或具体认证机制，要以设备和版本文档为准，不要默认所有设备支持。

## 9. Wireshark 抓包学习方法

### 9.1 抓包前

1. 记录 PLC、Adapter、交换机管理口的 IP、子网和拓扑。
2. 确认设备是否使用线型、星型或 DLR。
3. 在交换机上配置 SPAN/Port Mirroring，或者把抓包主机放到合适的链路位置。
4. 先建立正常通信，再制造一个可控故障，例如拔掉网线或修改一个不影响安全的参数。

### 9.2 建议观察顺序

1. ARP 和 IP 是否正常。
2. TCP 44818 是否能建立。
3. 是否出现 `RegisterSession`。
4. 是否出现 `ListIdentity` 或其他识别通信。
5. 是否出现 `Forward_Open`。
6. 是否开始 UDP 2222 的周期性 I/O。
7. I/O 周期是否接近 RPI，是否出现超时或组播异常。
8. 设备断线后是否发送 `Forward_Close`，是否能够重连。

### 9.3 常用 Wireshark 过滤器

```text
tcp.port == 44818
udp.port == 2222
cip
enip
ip.addr == 192.168.1.10
tcp.stream eq 0
```

字段分析顺序：

```text
Ethernet -> IP -> TCP/UDP -> EtherNet/IP Encapsulation -> CPF -> CIP -> Service/Path/Data
```

不要只盯着十六进制；先用 Wireshark 展开协议树，再对照规范确认字段的长度、端序、状态和连接 ID。

## 10. 实践路线

### 阶段 A：普通网络基础，1-2 天

- 用两台电脑或虚拟机完成静态 IP、`ping`、ARP 和 TCP/UDP 基础测试。
- 用 Wireshark 抓 ARP、TCP 三次握手、UDP 数据报。
- 画出 Ethernet、IP、TCP/UDP 的封装关系。

### 阶段 B：EtherNet/IP 识别和显式消息，2-4 天

- 安装 Wireshark。
- 使用支持 EtherNet/IP 的真实设备、仿真器或开发板。
- 观察 `ListIdentity`、`RegisterSession`、`SendRRData`。
- 找到一个 Identity Object 读取请求，标出 Class、Instance、Attribute 和 Service。
- 记录请求、响应、状态码和错误响应。

### 阶段 C：I/O 连接，3-7 天

- 准备一个 Adapter 设备或仿真 I/O 端点。
- 配置 EDS、输入 Assembly、输出 Assembly 和连接参数。
- 抓取 `Forward_Open`。
- 观察 UDP 2222 的 Connection ID、序列计数、I/O Payload 和实际周期。
- 逐项改变 RPI、连接大小或网络负载，记录周期、抖动、丢包和超时。

### 阶段 D：嵌入式 Linux / C 开发，1-2 周

- 先写普通 TCP/UDP Socket 程序，不要一开始就实现完整协议栈。
- 实现 Encapsulation Header 的编码和解析。
- 实现 `RegisterSession` 和 `ListIdentity`。
- 实现 CPF 的 Item 解析。
- 实现一个只支持 `Get_Attribute_Single` 的 CIP Message Router。
- 再研究 `Forward_Open`、Assembly 和周期性 I/O。
- 加入超时、断线重连、日志和抓包回归测试。

推荐源码入口：OpENer。阅读顺序为 README -> 初始化 -> Ethernet/IP 收发 -> Encapsulation -> CIP 服务分发 -> Assembly/I/O -> 连接状态机。

## 11. 一个可执行的第一个小实验

目标：只完成设备识别和一个显式读取，不涉及控制电机或改变危险参数。

1. 将电脑与测试设备接入同一隔离交换机。
2. 给电脑和设备设置同一网段的静态 IP。
3. 用 Wireshark 开始抓包。
4. 通过设备工具执行一次识别或读取设备信息。
5. 找到 TCP 44818 的 `RegisterSession`。
6. 找到 `SendRRData` 中的 CIP 请求。
7. 标记请求里的 Service、Class、Instance、Attribute。
8. 对照响应状态确认读取是否成功。
9. 把一条完整报文按“网络层次、Encapsulation、CPF、CIP”画成图。
10. 保存 pcapng 和你的字段解释，形成第一个实验记录。

安全要求：实验网络与生产网络隔离；初次实验只做识别、读取和诊断；不要直接向真实设备发送复位、写参数、启动、停止或运动控制服务。

## 12. 常见故障排查表

| 现象 | 优先检查 |
|---|---|
| Ping 不通 | 电缆、链路灯、IP、子网、VLAN、ARP、重复 IP |
| TCP 44818 建不起来 | 防火墙、端口访问、设备服务状态、路由 |
| 能注册会话但 CIP 报错 | Service、Class/Instance/Attribute、路径、设备权限 |
| Forward_Open 失败 | EDS、电子钥匙、Assembly 实例/大小、连接类型、RPI、目标路径 |
| 有 UDP 但 I/O 不更新 | Connection ID、组播、交换机 IGMP、方向和 Assembly 映射 |
| 周期性超时 | RPI、超时倍数、组播丢包、交换机队列、CPU 负载、线缆质量 |
| 断线后不能恢复 | Forward_Close/重连状态机、Session Handle、资源释放、日志 |

## 13. 推荐资料、视频和工具

### 官方和规范方向

- ODVA 官网：<https://www.odva.org/>。用于了解 CIP、EtherNet/IP、互操作和规范体系。
- 你提供的《CIP 网络库第 2 卷：CIP 的 EtherNet/IP 适配》：重点阅读简介、封装协议、显式/ I/O 消息映射、连接管理和 TCP/IP 要求。
- 你提供的《Ethernet Reference Manual》：重点阅读 CIP、TCP/UDP、CIP Connections、Implicit Messages、Explicit Messages、DLR、QoS 和 PTP/CIP Sync。
- Rockwell Automation Literature Library：<https://literature.rockwellautomation.com/>。搜索 `ENET-UM006`、`ENET-AT007`、`EtherNet/IP Network Devices User Manual`。

### 开源和抓包

- OpENer：<https://github.com/EIPStackGroup/OpENer>。适合研究 EtherNet/IP Adapter、CIP 服务和 I/O 连接。
- Wireshark：<https://www.wireshark.org/>。用于 Ethernet/IP/CIP/TCP/UDP 报文分析。
- tcpdump：适合在嵌入式 Linux 上采集 pcap 文件，再用 Wireshark 离线分析。

### 中文入门阅读

- 你笔记中整理的 CSDN 上下两篇 EtherNet/IP 基础文章：
  - <https://blog.csdn.net/weixin_47677928/article/details/108398509>
  - <https://blog.csdn.net/weixin_47677928/article/details/108399288>
- CSDN 专栏：<https://wenku.csdn.net/column/rhp4krhtkd>

### 视频搜索建议

视频内容更新较快，建议在 Bilibili 或 YouTube 使用以下关键词，并优先选择有抓包、PLC/Adapter 实验和完整报文分析的课程：

- `EtherNet/IP CIP 入门`
- `EtherNet/IP explicit implicit messaging`
- `EtherNet/IP Wireshark Forward Open`
- `EtherNet/IP Assembly Object RPI`
- `OpENer EtherNet/IP tutorial`
- `Rockwell EtherNet/IP CIP training`

判断视频质量的标准：讲清 CIP 对象模型、Encapsulation/CPF、显式与隐式消息的区别，并展示真实 Wireshark 报文；只讲 PLC 软件点选步骤而不解释报文的内容，适合作为操作演示，不适合作为协议主教材。

## 14. 最终记忆框架

```text
Ethernet/IP 网络问题
  1. 线缆、交换机、MAC、IP、ARP
  2. TCP 44818 会话和 Encapsulation
  3. CPF Item
  4. CIP Service + Path
  5. Identity / Assembly / Connection Manager
  6. Forward_Open 建立 I/O 连接
  7. UDP 2222 周期性数据、Connection ID、RPI、超时
  8. Wireshark 验证实际行为
```

真正掌握的标准是：你能从一份 pcapng 中回答“谁和谁通信、使用什么端口、是否注册会话、访问哪个 CIP 对象、是否建立 I/O 连接、I/O 数据的周期和连接状态是否正常”。

## 15. 本笔记使用的资料

本笔记综合了以下本地资料：

- `EtherNet与IP.md`：硬件、OSI 映射、CIP、应用场景和协议对比。
- `未命名 1.md`：ODVA、封装头、CPF、显式/隐式消息和协议层细节。
- `未命名 2.md`：设备等级和网页来源。
- `未命名 3.md`：CIP 对象模型、Scanner/Adapter、RPI、EDS、Wireshark、OpENer 和学习顺序。
- `未命名.md`：通信类型、CIP 对象/服务、封装命令和报文分析图示来源。
- `EtherNetIP中文版(翻译).pdf`：CIP Networks Library Volume 2，EtherNet/IP 适配、封装、消息映射、连接类别和 RPI/超时相关内容。
- `5359b7f7-174d-45ab-afd4-0d6b65f672e2.pdf`：Ethernet Reference Manual，CIP、网络基础、DLR、QoS、PTP/CIP Sync、显式/隐式消息和连接说明。

原始资料中的示意图和网页摘录没有全部复制到本文件；本文件保留了它们的关键概念，并对端口、OSI 映射、隐式消息封装和安全能力等容易误解的地方做了限定说明。



---

## 原始资料：EtherNet与IPdsaffgxhd.md


**EtherNet/IP**（以太网工业协议）是一种基于标准以太网的工业自动化通信协议，由 **ODVA（开放设备网供应商协会）** 管理。它融合了 **CIP（通用工业协议）** 和以太网技术，支持实时控制、设备配置和数据采集。以下从硬件接线、底层协议到应用层展开详解：


#####  一、**硬件接线与物理层**

物理介质：

- 线缆：采用标准以太网线（Cat5e/Cat6/Cat6a），工业环境中常用 屏蔽双绞线（STP） 抗干扰。
- 接头：工业级 M12 防水接头（IP67防护）或RJ45，适应振动、粉尘等严苛环境。
- 传输距离：铜缆最长100米（可通过光纤扩展至数公里）。

拓扑结构：
- 星型拓扑：通过工业以太网交换机连接设备，支持冗余（如环网冗余协议 DLR）。
- 设备级联：支持菊花链拓扑（需交换机支持）。

关键硬件：
- 工业交换机：支持 IEEE 802.1Q VLAN、QoS（优先级标记）、IEEE 1588 PTP（精确时间同步）。
- 终端设备：PLC（如罗克韦尔ControlLogix）、I/O模块、变频器、HMI等


##### **二、底层协议栈（OSI模型分层）**

EtherNet/IP 基于标准以太网协议栈，扩展了实时通信能力：

| OSI 层 | 协议 / 技术                       | 功能说明                                     |
|-------|-------------------------------|------------------------------------------|
| 物理层   | IEEE 802.3（以太网）               | 定义电气信号、线缆、接口（如 100BASE-TX、1000BASE-T）    |
| 数据链路层 | IEEE 802.3 MAC + VLAN 标签      | MAC 地址寻址，支持 VLAN 隔离流量，QoS（优先级标记，如 CoS 值） |
| 网络层   | IPv4/IPv6                     | IP 地址分配（静态 / DHCP），支持子网划分                |
| 传输层   | TCP（端口 44818）+ UDP（端口 2223）   | 显性消息（配置 / 诊断）用 TCP，隐性消息（实时 I/O）用 UDP     |
| 会话层   | CIP 封装会话管理                    | 建立 / 维护设备间逻辑连接                           |
| 表示层   | CIP 数据编码（如 BOOL, INT, STRING） | 标准化数据格式，确保设备间兼容                          |
| 应用层   | CIP（通用工业协议）                   | 定义设备对象模型、服务接口（如 Read/Write Tag）          |


##### 三、核心协议：CIP（通用工业协议）
CIP 是 EtherNet/IP、DeviceNet、ControlNet 的共同应用层协议，提供统一的设备交互框架 。

- CIP 对象模型：
 - 身份对象（Identity Object）：包含设备厂商、型号、序列号等。
 - 连接对象（Connection Object）：管理实时数据通道参数。
 - 参数对象（Parameter Object）：存储设备配置参数。
 - I/O数据对象：映射输入输出数据。
- 通信模式：
  - 显性消息（Explicit Messaging）：基于TCP，用于非实时操作（如读写配置、诊断）。
  - 隐性消息（Implicit Messaging）：基于UDP，用于高速周期I/O数据交换（如传感器数据）。
- 服务类型：
  - Unconnected：临时通信（如设备发现）。
  - Connected：预建立连接，保障带宽和优先级。


**四、应用层功能与典型场景**
实时I/O控制：
- 使用 CIP Sync 扩展实现微秒级同步（基于IEEE 1588 PTP），适用于运动控制。
- CIP Motion：直接控制伺服驱动器，实现多轴协同。

设备配置与诊断：
- 通过 EDS（电子数据表） 文件描述设备参数，支持离线配置。
- 使用 FINS/UDP 或 CIP指令 远程读取设备状态、日志。

安全机制：
- CIP Security：支持设备身份认证、数据加密（如TLS 1.3）。
- 网络隔离：通过VLAN划分控制流量，防火墙过滤非授权访问。



##### **五、与其他工业协议对比**

| **协议**          | **优势**            | **适用场景**          |
| --------------- | ----------------- | ----------------- |
| **EtherNet/IP** | 基于标准以太网，支持CIP灵活扩展 | 复杂控制、跨厂商集成（如汽车制造） |
| **Profinet**    | 高实时性（IRT级）        | 高速运动控制（如包装机械）     |
| **Modbus TCP**  | 简单易实现，资源占用低       | 中小型SCADA系统        |


---

## 原始资料：未命名 1.md


一、EtherNet/IP基础概念
1.ODVA组织简介
ODVA成立于1995年，是一个全球性协会，其成员包括世界领先的自动化公司。ODVA的使命是促进工业自动化 中开放的，可互操作的信息和通信技术。ODVA将其独立于媒体的网络协议，通用工业协议或“CIPTM”以及CIP的网络改进版EtherNet/IPTM，DeviceNetTM , CompoNetTM和ControlNetTM视为其核心技术。其负责的主要内容有：

设备的一致性测试认证与互操作；
授权EtherNet/IP技术；
发放Vendor ID代表厂商设备信息；
其下的SIG小组负责各类规范、方针等内容的制定。
2.EtherNet/IP基础架构
2.1 EtherNet/IP OSI参考模型

图：EtherNet/IP的ISO/OSI参考模型
EtherNet/IP的ISO/OSI参考模型如上图所示，其物理层、链路层、网络层 、传输层均与标准以太网相同，特点在于其在传输层之上增添了一个封装层，对来自应用层的数据进行报文格式的封装；而应用层采用的是CIP协议，满足其在工业自动化领域的一些功能和服务。

特点：

服务于各类工业自动化应用；
基于标准的IEEE802.3与TCP/IP Suite；
应用层采用通用工业协议CIP；
IT集成，包括Web、OPC等服务；
物理层实现基于标准的以太网控制器；
规定了特定的传输端口，UDP/IP(PORT 44818)，TCP/IP(PORT 44818)。
2.2 EtherNet/IP 各层功能特点
2.2.1 物理层：

1）传输介质可选铜缆双绞线或光纤；

2）几乎可以容纳无限数量的点对点节点；

3）可以通过嵌入式交换机技术和设备级环网技术来实现线型、星型和环网拓扑；

4）可根据需求选择不同的物理接口：基本的以太网连接，支持COTS现成商用标准，适合IP 20应用；工业等级应用，需性能增强的RJ-45连接器，密封RJ-45连接器以及更紧凑的M12连接器，适合IP 67应用；

2.2.2 数据链路层与网络层：


图：EtherNet/IP的以太网帧格式
EtherNet/IP的链路层帧格式如上图所示，与标准以太网没有区别，遵循IEEE 802.3规范设计，并支持全双工传输。


图：EtherNet/IP的网络层与传输层
 

同时网络层与传输层也使用TCP/IP与UDP/IP进行传输，两种方式分别传输两种不同的消息：TCP/IP是面向连接的点对点传输，并且传输的是封装CIP explicit messages（显示消息），主要用于配置信息与诊断数据的传输；UDP/IP是单对多的广播传输，主要用于实时性要求较高的数据传输，并由Timeout机制检测是否传输出错。

2.2.3 Encapsulation封装协议

2.2.3.1 主要功能


图：EtherNet/IP封装层协议
实时性要求不同的数据会采用不同的封装形式与传输形式：实时性要求不高的数据会采用CPF（Common Packet Format通用包格式）+Encapsulation的封装，走TCP/IP传输通道；而实时性要求较高的数据仅采用CPF封装，并走UDP/IP传输通道。封装层主要功能如下：

解除IP网络的CIP消息封装；
将CIP消息包装为以太网消息；
会话管理；
与下层TCP/IP协议层以及TCP/IP堆栈进行交互。
2.2.3.2 封装结构


图：EtherNet/IP以太网包结构

图：封装部分的结构
EtherNet/IP的以太网包与其中的封装部分结构如上图所示，其中封装部分各细分部分功能如下：

Command：代表该包的功能和作用；

Length：指示后方数据域的长度；

Session Handle：会话句柄，表示会话建立或响应的请求；

Status：指示改包的命令有无正确执行，可比对状态码表进行确认；

Sender Context：包含描述发送者信息的内容；

Options：可选设置；

Command-specific Data：根据命令的不同有所区别

2.2.3.3 命令类型


表：部分命令类型
 2.2.3.4 命令特定数据域（CPF）结构


图：数据域结构
数据域部分的结构如上图所示，其结构为1个1个的item的连接，其每种item的含义如下：

Item Count：指示该包数据域中item的总个数；
Address Item：存放目的地地址信息：
Type ID：地址类型；

Length：地址长度；

Data：地址具体数值；

Data Item：存放数据信息：
Type ID：数据类型；

Length：数据长度；

Data：具体数据，封装了CIP报文包的内容，包含CIP命令、请求路径等内容；

2.2.4 传输层的消息类型

2.2.4.1 显式消息

应用：非周期性从端数据的读写

特点：

①报文中包含地址项和服务请求（命令）；

②通常用于两个站点之间的通用通信路径；

③面向逻辑连接的连接方式，点对点传输，请求对应响应；

 


图：显式消息封装部分的报文结构
上图中，Command的值为0x6F，指示该报文为SendRRData，其上半部分的结构为封装报文头结构，下半部分为命令特定数据域结构，与前文提到的报文结构基本无异。

2.2.4.2 隐式消息

应用：实时的IO应用数据交互

特点：

①报文中没有地址项，也没有服务请求（命令）；

②通过Connection ID来确定任务目标；

③生产者与（一或多个）消费者之间的特殊通信路径；

④点对点或多播；


图：隐式消息封装部分的报文结构
上图的报文结构中，不含有封装协议报文头，且命令特定数据域中的地址项与数据均固定，通过Connection ID来标识访问的数据。

2.2.5 EtherNet/IP 应用层-CIP


图：CIP协议基本架构
基本原理：以是否存在实时性要求为依据，划分数据的优先级，区别对待不同优先级的数据。具体的协议内容，将单独成篇进行介绍。

2.3 EtherNet/IP 特点总结
①在标准以太网基础上建立，而不仅仅是与之兼容；

②多种数据传输速率：10、100、1000Mbit/s；

③可以使用标准基础架构来构建系统；

④可以使用IP路由器将网络构建为子网；

⑤使用IP寻址进行所有通信，完全支持跨子网的通信，因此网络中几乎没有节点数量限制；

⑥非实时通信和实时通信可以在同一子网中共存；

⑦支持协调驱动和运动控制；

⑧支持设备级环网（DLR），该环网通过介质冗余提供单容错能力；

⑨与其他上层协议（如HTTP,FTP,VOIP等）共存；




---

## 原始资料：未命名 3.md



结合你之前准备的 STM32、嵌入式 Linux、CAN/CANopen、EtherCAT、电机控制和工业网关项目，我建议把 EtherNet/IP 分成六大部分：

1. 协议基础与分层架构
    
2. CIP 核心数据模型
    
3. EtherNet/IP 报文与封装
    
4. 显式消息与隐式 I/O 消息
    
5. 连接管理、设备角色与实时通信
    
6. Linux 开发、协议栈实现、调试和面试
    

先建立一个整体认识：

工业控制应用

PLC、HMI、伺服驱动器、远程 I/O、机器人

## CIP

对象模型、服务、参数读写、I/O 数据、连接管理

TCP

显式消息的常见承载方式

UDP

周期性 I/O 的常见承载方式

IP + Ethernet

IP 寻址、MAC 地址、以太网帧、物理介质

需要先纠正一个常见理解：EtherNet/IP 不是 EtherCAT 的另一种叫法，也不是简单地用 TCP Socket 发送几个自定义字节。它在标准以太网之上使用 CIP 定义工业设备的对象、服务和通信机制。

下面按知识点展开，你可以直接用作学习笔记。

# 第一部分：协议基础与整体架构

## 1. EtherNet/IP 是什么？

EtherNet/IP 全称 Ethernet Industrial Protocol，中文一般叫工业以太网协议。

它由 ODVA 维护，以 CIP（Common Industrial Protocol，通用工业协议）作为核心工业通信机制。

它主要解决这些问题：

- 工业设备之间如何交换控制数据。
    
- 如何读取和修改设备参数。
    
- 如何获取设备身份、状态和故障信息。
    
- 如何建立、维护和监控周期性 I/O 通信。
    
- 如何让不同厂商的兼容设备接入同一套工业控制系统。
    

典型设备包括：

- PLC 控制器。
    
- 远程 I/O 模块。
    
- 变频器与伺服驱动器。
    
- 工业机器人。
    
- 传感器、执行器及工业网关。
    

## 2. EtherNet/IP 的协议分层

学习时要掌握每一层的作用。

|层次|主要内容|解决的问题|
|---|---|---|
|应用层|CIP、设备对象和服务|设备数据代表什么、可以执行什么操作|
|传输层|TCP、UDP|消息如何传输|
|网络层|IP|数据包如何寻址与路由|
|数据链路层|Ethernet、MAC|数据如何在局域网中传输|
|物理层|Ethernet PHY、网线或光纤|信号如何传输|

这是便于理解的简化映射，不代表 CIP 与传统 OSI 七层模型严格一一对应。

### 需要掌握的基础知识

- Ethernet 帧的基本结构。
    
- MAC 地址与 IP 地址的区别。
    
- ARP 的作用。
    
- TCP 三次握手、连接和断开。
    
- UDP 数据报的特点。
    
- IP、子网掩码、网关与路由。
    
- TCP/UDP 端口号。
    

常见的 EtherNet/IP 端口包括：

|端口|用途|
|---|---|
|TCP 44818|EtherNet/IP 封装协议及显式消息常用通信端口|
|UDP 44818|EtherNet/IP 相关 UDP 封装通信|
|UDP 2222|常见的隐式 I/O 数据通信端口|

注意：这些是常见标准端口，不代表所有数据都使用同一种端口，也不能只凭端口号就断定某个数据包一定是某类 CIP 消息。

# 第二部分：CIP 核心知识——最重要的部分

如果你只记住 EtherNet/IP 的一个核心知识点，那就是：CIP 定义了工业设备的数据模型、服务以及通信行为。

## 3. CIP 是什么？

CIP 是 Common Industrial Protocol，通用工业协议。

EtherNet/IP 使用 CIP，DeviceNet、ControlNet 等工业通信技术也使用 CIP 体系中的相关机制。

CIP 并不只是定义数据怎么传输，还定义了设备如何对外提供功能。

例如，伺服驱动器可以把自己的身份、状态、配置参数和控制能力抽象成对象，让外部控制器通过规定的服务访问。

## 4. CIP 对象模型

必须掌握五个概念：

### Class（类）

定义一类对象的属性、服务和行为。例如 Identity Object（身份对象）这一类设备对象。

### Instance（实例）

表示某个类中的具体对象实例。同一类可以存在一个或多个实例，具体由对象规范决定。

### Attribute（属性）

对象内部的具体数据，例如厂商标识、设备类型、状态或某项参数。

### Service（服务）

对对象执行的操作，例如读取属性、写入属性、复位或执行特定功能。

### Path（路径）

用于指定要访问哪个对象、哪个实例，以及必要时访问哪个属性或其他目标。

### 4.1 用一个具体例子理解

假设 PLC 想要读取一台工业设备的身份信息。

逻辑过程如下：

```
PLC
 │
 │ 发送 CIP 请求
 │ 指定 Identity Object
 │ 指定目标实例
 │ 指定要读取的属性
 ▼
EtherNet/IP 传输
 │
 ▼
目标设备
 │
 │ 查找对象和属性
 │ 生成响应
 ▼
返回属性值
```

你需要理解：读取的不是某个随意约定的内存地址，而是通过 CIP 规定的对象、路径和服务访问数据。

实际报文中的路径编码、服务码、实例号和属性号，需要按照相应的 CIP 对象规范与报文格式进行解析。

### 4.2 常见 CIP 对象

|对象|主要作用|
|---|---|
|Identity Object|设备身份、厂商、设备类型及版本等信息|
|Message Router Object|在 CIP 对象体系中路由请求|
|Assembly Object|将多个数据项组织成适合 I/O 交换的数据集合|
|Connection Manager Object|连接相关服务与管理|
|TCP/IP Interface Object|设备的 TCP/IP 接口配置与相关信息|
|Ethernet Link Object|以太网链路相关信息|

其中，Assembly Object 和 Connection Manager Object 是工业 I/O 通信中非常重要的对象。

Assembly Object 通常用于组织输入、输出或配置数据；Connection Manager 相关机制则用于处理连接请求及连接管理功能。

但不要认为所有厂商设备都实现所有对象，具体对象、实例及其能力需要查看设备的 EDS 文件、用户手册或相关规范。

## 5. CIP 服务与常见服务码

CIP 服务表示“我要对对象做什么”。

常见服务包括：

|服务|常见服务码|作用|
|---|---|---|
|Get_Attribute_Single|`0x0E`|读取单个属性|
|Set_Attribute_Single|`0x10`|写入单个属性|
|Get_Attributes_All|`0x01`|读取对象规定的全部属性|
|Reset|`0x05`|执行对象定义的复位操作|
|Forward_Open|`0x54`|请求建立特定类型的 CIP 连接|
|Forward_Close|`0x4E`|请求关闭相应 CIP 连接|

这些服务的可用性、参数格式和具体语义取决于目标对象和对应规范。例如，不是所有对象都支持 `Reset`，也不是所有设备都支持所有连接类型。

还有一个容易混淆的点：`Forward_Open` 和 `Forward_Close` 是 CIP 连接管理服务，并不是 TCP 的 `connect()` 和 `close()` 的直接替代品。

TCP 连接与 CIP 应用连接是不同层面的概念。

## 6. CIP 请求与响应

学习报文时，要区分请求与响应。

一个典型的 CIP 显式消息会涉及：

- Service Code：服务码。
    
- Request Path：请求路径。
    
- Request Data：请求参数或数据。
    
- General Status：通用状态。
    
- Additional Status：附加状态信息。
    
- Response Data：响应数据。
    

请求的字段布局与响应的字段布局并不完全相同。

举例来说，PLC 读取一个属性时，请求指定服务和目标路径；设备返回执行状态以及对应的属性数据。

如果操作失败，就需要检查状态码和附加状态，而不是只看 TCP 是否成功发送。

# 第三部分：EtherNet/IP 报文结构

## 7. 什么是 Encapsulation（封装）？

EtherNet/IP 在 CIP 之上定义了封装协议。

封装层会携带诸如命令类型、数据长度、会话标识、状态和上下文信息等字段，让接收端知道如何处理后续数据。

一个常见的 EtherNet/IP 封装头长度为 24 字节。

|字段|长度|作用|
|---|---|---|
|Command|2 字节|指示封装命令类型|
|Length|2 字节|表示后续封装数据的长度|
|Session Handle|4 字节|会话标识|
|Status|4 字节|封装层状态|
|Sender Context|8 字节|请求与响应关联等用途的上下文|
|Options|4 字节|封装选项|
|合计|24 字节|固定封装头|

以上是封装头的通用结构。不同命令的后续数据结构不同，有的命令不携带额外数据，有的会携带消息路由相关内容。

### 7.1 常见封装命令

需要认识以下命令名称：

- `ListServices`：查询可提供的服务。
    
- `ListIdentity`：查询设备身份信息。
    
- `RegisterSession`：注册封装会话。
    
- `UnregisterSession`：注销会话。
    
- `SendRRData`：发送相关的非连接数据。
    
- `SendUnitData`：发送相关的连接数据。
    

这些命令是学习 EtherNet/IP 报文分析的基础。

特别要注意：

Encapsulation Command 和 CIP Service 不是一回事。

前者属于 EtherNet/IP 封装层；后者属于 CIP。比如 `SendRRData` 是封装命令，而 `Get_Attribute_Single` 是 CIP 服务。

## 8. 完整报文的封装关系

一个常见的显式消息通信路径可以抽象成：

```
Ethernet Frame
└── IP Packet
    └── TCP Segment
        └── EtherNet/IP Encapsulation
            └── CPF（Common Packet Format）
                └── CIP Message
                    ├── Service
                    ├── Request Path
                    └── Request Data
```

这是一个用于学习的典型结构，不是每种 EtherNet/IP 报文都必须具有完全相同的嵌套形式。

其中 CPF 是 Common Packet Format（通用数据包格式），用于组织相关的通信数据项。不同命令和消息类型所携带的数据项会有所不同。

学习报文时要按层分析：

1. 先确认 Ethernet 帧的源 MAC 和目标 MAC。
    
2. 再确认 IP 地址与协议。
    
3. 再查看 TCP/UDP 端口和传输内容。
    
4. 解析 EtherNet/IP 封装头。
    
5. 继续解析 CPF。
    
6. 最后解析 CIP 服务和对象路径。
    

这套方法与你以后分析 CAN、CANopen、EtherCAT 报文的思路是类似的：从外层逐步进入内层。

# 第四部分：显式消息与隐式消息

这是 EtherNet/IP 最重要的面试知识点之一。

## 9. Explicit Messaging（显式消息）

显式消息是有明确服务含义的消息，通常用于设备配置、参数访问、诊断和非周期性数据交换。

例如：

```
PLC
 │
 │ 读取设备身份
 │ CIP Get_Attribute_Single
 ▼
EtherNet/IP
 │
 │ TCP 上的显式消息（常见方式）
 ▼
工业设备
 │
 │ 返回属性值和状态
 ▼
PLC
```

需要掌握：

- 请求和响应的基本流程。
    
- CIP Service Code。
    
- 对象路径和属性访问。
    
- 封装会话与消息路由。
    
- 错误状态与超时处理。
    

显式消息并不全部都是 TCP 连接消息；具体传输方式与通信类型有关。但在典型 EtherNet/IP 工业配置中，显式消息常使用 TCP。

## 10. Implicit Messaging（隐式消息）

隐式消息通常用于周期性的 I/O 数据交换。

例如，PLC 每隔一段时间向驱动器发送控制字和速度给定，同时接收驱动器返回的状态字与实际速度。

```
PLC / Scanner
      │
      │ Output Assembly
      │ 控制字、速度给定
      ▼
  UDP / I/O 连接
      │
      ▼
伺服驱动器 / Adapter
      │
      │ Input Assembly
      │ 状态字、实际速度
      ▼
PLC / Scanner
```

这里的 Input Assembly、Output Assembly 是从控制器与设备的数据交换方向来描述的；具体字节布局由设备的 I/O 定义决定。

隐式消息之所以重要，是因为控制器不需要为每次数据交换都发送一个完整的 CIP 属性读取请求。双方已经配置好 I/O 连接，便可按约定周期交换数据。

### 10.1 两种消息的区别

|维度|Explicit|Implicit / I/O|
|---|---|---|
|核心目的|访问明确的服务与参数|周期性交换 I/O 数据|
|数据含义|由服务、对象、路径等描述|由连接与 I/O 数据映射约定|
|典型承载|TCP|UDP|
|常见封装命令|`SendRRData`|`SendUnitData`|
|常见用途|配置、诊断、参数读写|控制字、状态字、过程数据|
|时间要求|取决于操作及应用需求|通常有明确的周期和超时要求|

注意：上表是常见配置，不是对所有通信形式的绝对限制。尤其不能认为使用 UDP 就必然是 EtherNet/IP I/O，或使用 TCP 就必然不是 CIP 通信。

# 第五部分：连接管理、设备角色与实时性

## 11. Scanner 和 Adapter

EtherNet/IP 中经常会看到 Scanner、Adapter、Originator、Target 等术语。

常见对应关系如下：

|角色|典型设备|主要职责|
|---|---|---|
|Scanner|PLC、控制器|发起 I/O 连接，管理目标设备的数据交换|
|Adapter|远程 I/O、部分驱动器|接受连接请求，提供或消费相应 I/O 数据|
|Originator|连接发起方|发起特定 CIP 连接|
|Target|连接目标方|接受或处理连接请求|

Scanner 和 Adapter 是工业 I/O 通信中常用的角色称呼；Originator 和 Target 则是更一般的连接角色。不要把它们理解成在所有情况下都严格一一对应。

### 11.1 Scanner 的典型工作流程

```
启动
 │
 ▼
检查目标设备及网络配置
 │
 ▼
建立所需通信连接
 │
 ▼
配置 I/O 数据与通信周期
 │
 ▼
周期性交换输入/输出数据
 │
 ▼
检测超时、故障和连接状态
```

如果你以后开发 EtherNet/IP Adapter，那么重点会转为：如何处理连接请求、维护连接状态、组织 I/O 数据，以及在通信中断时安全地处理输出。

## 12. RPI 是什么？

RPI 全称 Requested Packet Interval，即请求数据包间隔。

它是配置 EtherNet/IP 周期性 I/O 通信时非常重要的参数，用于指定请求的数据包间隔。

例如，某个 I/O 连接的 RPI 设置为 10 ms，意味着按照该连接配置请求以约 10 ms 的间隔交换数据。

但要注意：

- RPI 不等于整个控制系统的端到端响应时间。
    
- RPI 不保证每个数据包都准时到达。
    
- 实际性能会受到网络负载、设备处理能力、交换机、控制器任务周期和连接类型等因素影响。
    
- 不同方向的数据生产和消费也要结合具体连接配置理解。
    

## 13. EtherNet/IP 是实时协议吗？

这是面试中常见的追问。

准确回答是：EtherNet/IP 支持用于工业实时 I/O 交换的通信机制，但普通 EtherNet/IP 并不自动等于硬实时通信。

它通过周期性 I/O、连接监控以及相应的网络机制实现工业控制所需的数据交换。

不过，若应用要求严格确定性的多轴运动同步，需要进一步考虑具体的 CIP Motion、同步机制、设备能力以及整个控制系统的时序设计，不能只凭 EtherNet/IP 这个名称判断实时性能。

## 14. EDS 文件是什么？

EDS 是 Electronic Data Sheet，电子数据表。

它用于描述设备的身份和能力、参数、通信相关信息等，供兼容的配置工具或控制系统使用。

在实际项目中，工程师经常需要导入对应设备的 EDS 文件，再进行设备配置。

你需要掌握：

- EDS 与设备型号的关系。
    
- EDS 的用途与局限。
    
- EDS、设备手册和实际固件能力之间的关系。
    

EDS 不是 EtherNet/IP 协议栈，也不是设备本身的固件。 它是设备描述信息的一种标准化文件形式，具体内容与设备及相应规范有关。

# 第六部分：Linux、嵌入式开发与实际调试

## 15. 如果要在嵌入式 Linux 上使用 EtherNet/IP

建议你掌握以下技术栈：

|技术|需要掌握的内容|
|---|---|
|C 语言|指针、结构体、字节数组、内存管理|
|Linux 网络编程|Socket、TCP/UDP、收发缓冲区、错误处理|
|多线程与并发|线程同步、I/O 线程、共享数据保护|
|网络配置|IP、路由、网卡、ARP、防火墙|
|抓包工具|Wireshark、tcpdump|
|协议栈|EtherNet/IP 封装、CIP、连接状态机、I/O 数据处理|
|工程调试|日志、超时、断线恢复、异常状态分析|

如果只是使用厂商已经实现的 EtherNet/IP 功能，通常不需要自己实现整套协议栈。

如果是开发一个 EtherNet/IP Adapter，则要深入研究协议栈如何解析封装命令、处理 CIP 请求、维护连接和交换 I/O 数据。

## 16. 实际调试需要会什么？

使用 Wireshark 时，建议按下面顺序分析：

1. 设备能否互相访问，IP 地址和路由是否正确。
    
2. TCP 连接能否建立。
    
3. 是否出现 `RegisterSession`。
    
4. 是否出现 `ListIdentity` 或其他发现、识别相关通信。
    
5. 是否发送 `Forward_Open` 等连接请求。
    
6. 是否开始周期性 I/O 数据交换。
    
7. 是否出现超时、连接关闭或错误状态。
    

不是所有设备都必须按完全相同的顺序执行所有步骤。不同厂商、连接类型和工程配置可能导致实际通信流程不同。

常用工具：

- Wireshark：分析 Ethernet、IP、TCP/UDP 和 EtherNet/IP 报文。
    
- tcpdump：在 Linux 终端抓取网络数据包。
    
- `ip addr`、`ip route`：查看网卡地址和路由。
    
- `ss`：检查 TCP/UDP 套接字状态。
    
- 厂商配置软件：检查设备参数、I/O 映射和连接状态。
    

## 17. 开源协议栈 OpENer

如果你希望从“会使用协议”进阶到“会开发协议”，可以研究 OpENer。

推荐顺序：

1. 阅读 README，理解支持的设备角色与功能范围。
    
2. 找到程序入口与初始化流程。
    
3. 找到 Ethernet/IP 封装层的收发处理代码。
    
4. 理解 CIP 请求的解析与服务分发。
    
5. 追踪 Assembly 和 I/O 数据处理逻辑。
    
6. 分析连接建立、超时和关闭的状态处理。
    
7. 用实际抓包验证代码行为。
    

项目地址：[OpENer GitHub 源码](https://github.com/EIPStackGroup/OpENer)。

建议先看懂数据如何进入协议栈、如何被解析、最终如何影响 I/O，而不是一上来就逐行阅读所有源文件。

# 第七部分：EtherNet/IP 面试必考知识点

下面这些问题建议你整理成笔记，逐个练习回答。

## 面试题自测

0/12 已查看

1. EtherNet/IP 是什么？基础

2. EtherNet/IP 和普通 TCP 通信有什么区别？基础

3. CIP 是什么？核心

4. Explicit Messaging 和 Implicit Messaging 有什么区别？核心

5. EtherNet/IP 为什么会使用 UDP？核心

6. CIP 和 EtherNet/IP 封装层有什么区别？报文

7. Scanner 和 Adapter 分别是什么？架构

8. RPI 是什么？实时性

9. Assembly Object 是什么？核心

10. EDS 文件有什么作用？工程

11. EtherNet/IP 报文如何分析？调试

12. 如何排查 EtherNet/IP I/O 通信失败？调试展开全部答案

# 第八部分：你的 EtherNet/IP 笔记目录

我建议直接把下面这个目录放进你之前规划的嵌入式开发知识库中。

## 工业通信 / EtherNet-IP

建议目录编号：07 · 可持续扩展

01_协议基础

01 EtherNet/IP 简介与应用场景

02 OSI 分层与 TCP/IP 架构

03 Ethernet、IP、TCP、UDP

02_CIP 核心机制

01 CIP 对象模型

02 Class、Instance、Attribute、Service、Path

03 Identity、Assembly、Connection Manager

04 CIP 状态码与错误处理

03_报文与通信

01 Encapsulation Header

02 CPF 数据包格式

03 Explicit Messaging

04 Implicit I/O Messaging

05 Scanner、Adapter、Originator、Target

06 RPI、连接管理与超时

04_Linux 与协议栈开发

01 Socket 与网络配置

02 Wireshark / tcpdump 抓包

03 OpENer 源码结构

04 CIP 服务处理与 I/O 数据映射

05 断线重连、异常恢复与日志

05_项目与面试

01 PLC 与 Adapter 通信实验

02 报文分析案例

03 常见故障排查

04 面试题与项目总结

复制完整笔记目录

## 最后，学习顺序怎么安排？

不要一开始就把所有知识点平均用力。我建议优先顺序如下：

1. 第一优先级： 协议分层、CIP 对象模型、TCP/UDP、显式与隐式消息。
    
2. 第二优先级： Encapsulation、CPF、Scanner/Adapter、RPI 和连接管理。
    
3. 第三优先级： Wireshark 抓包、Linux Socket、OpENer 协议栈。
    
4. 第四优先级： 深入 CIP 对象规范、复杂连接类型、运动控制扩展与协议符合性。
    

你目前最应该做的是先把第一、第二优先级的知识讲清楚，并且能够脱离笔记画出一张协议架构图。接下来再进入报文字段和实际抓包，这样学习效率会更高。


---

## 原始资料：未命名.md


### EtherNet/IP 通信类型

EtherNet/IP 定义了两种类型的通讯方式，显性（explicit）与隐形（implicit）。

|CIP 消息类型|CIP 通讯关系|传输协议|通讯类型|适用情形|举例|
|---|---|---|---|---|---|
|显性|已连接或未连接|TCP/IP|请求/响应 交换|低时间相关性数据信息|读/写，配置字段|
|隐形|已连接|UDP/IP|I/O 数据交换|实时 I/O 数据|实时控制数据|

**显性通信：** 通过TCP 传输数据，适合用于传递高准确性，低及时性的数据，例如配置设备信息，上传或下载程序。表格中CIP通讯关系中的 _已连接与未连接_ 是CIP 协议的一个服务，完成该服务后，连接的双方各持一串号码，用该号码来完成后续的信息交流。  
**隐形通信：** 通过UDP传输数据，适合用来传输高实时性的数据，例如马达控制，传感器数据。该通信方式需要提前建立CIP连接（详见后续）。

CIP 连接需要通过 Connection Manager(CM) 对象的ForwardOpen 服务来完成。客户端作为请求的发起方，请求中包含传输类，时间信息，电子密钥以及连接ID。当接收到ForwardClose请求或响应超时，清空连接信息。  
对于隐形通信，其数据可以广播或发送给某特定地址。数据的发送必须包含对象的IP(单一IP或广播地址)以及 CIP 连接 ID。


## 二：CIP

ODVA 提供了七卷关于CIP网络库的指南，需要成为会员才能获取。  
卷一：描述了CIP的通用对象库，设备信息库。  
卷二：描述CIP与EtherNet/IP如何适配。  
卷三：描述CIP与DeviceNet如何适配。  
卷四：描述CIP与ControlNet如何适配。  
卷五：描述CIP安全协议， CIP Safety Protocol。  
卷六：描述CIP与CompoNet如何适配。  
卷七：描述如何将 Modbus 设备接入 CIP 网络。

其总体格式如下：

|CIP|包含命令，具体执行对象信息|
|---|---|
|EtherNet/IP|包含通信对象，连接信息|
|TCP or UDP|通信端口信息|
|IP|双方IP地址信息|
|Ethernet|双方物理地址信息|

### 1. CIP 对象模型

每个CIP节点(node)都是一组对象(object)的集合。每个对象都代表了设备的某个特定组件，所有未被描述的对象，都无法通过CIP访问（没有定义当然没法访问）。CIP对象由 类(class)，实例(instance)，属性（attribute）构成。每个类可以有多个实例，每个实例可以有多个属性。（举例：一个类叫做 男人。这个 男人 类可以有实例 王二狗，李铁柱。王二狗 可以包含属性年龄，身高，体重。在读取或写入的数据的时候，协议中就要申明要操作的是哪个类的哪个实例的哪个属性。）  
类（class）是一组代表相同系统组件的对象。实例（instance）是该类中的某个特定对象。每个实例可以有自己特有的属性。

CIP 网络中的每个节点都有节点地址，在EtherNet/IP 网络中，该地址即为设备的IP地址。  
CIP中每个类，实例，属性都有其对应的ID (Class ID, Instance ID, Attribute ID)。  
CIP中使用服务代码来明确操作指令。

类ID分为两个部分，公共对象（范围：0x0000–0x0063, 0x00F0–0x02FF），厂家自定义对象（范围：0x0064–0x00C7, 0X0300-0X04FF）。其它范围为预留部分。  
实例ID也分为两个部分，公共实例（范围：0x0001–0x0063,0x00C8-0x02FF），厂家自定义实例（范围：0x0064-0xxC7,0x0300-0x04FF）。其它范围为预留部分。  
属性ID,公共属性（范围：0x0000–0x0063，0x0100–0x02FF，0x0500–0x08FF），厂家自定义属性（范围：0x0064–0x00C7，0x0300–0x04FF，0x000–0x0CFF）。

### 2. CIP 服务

CIP 服务代码用来明确请求的操作指令，上述提到的 Forward Open就是一个服务，其它包括常规的读/写服务，创建实例服务。CIP服务代码同样分为公共服务以及厂家自定义服务。

### 3. CIP 对象库

CIP协议簇包含了许多常用对象，大致可分为三类：一般用途，特定应用，特定网络。  
对于一般用途如下表：

|名称|代码|名称|代码|
|---|---|---|---|
|Assembly|0x04|Message Router|0x02|
|Acknowledge Handler|0x2B|Originator Connection List|0x45|
|Connection|0x05)|Parameter|0x0F|
|Connection Configuration|0xF3|Parameter Group|0x10|
|Connection Manager|0x06|Port|0xF4|
|File|0x37|Register|0x07|
|Identity|0x01|Selection|0x2E|

对于特定应用：

|名称|代码|名称|代码|
|---|---|---|---|
|AC/DC Drive|0x2A|Analog Group|0x22|
|Analog Input Group|0x20|Analog Input Point|0x0A|
|Analog Output Group|0x21|Analog Output Point|0x0B|
|Base Energy|0x4E|Block Sequencer|0x26|
|Command Block|0x27|Control Supervisor|0x29|
|Discrete Group|0x1F|Discrete Input Group|0x1D|
|Discrete Output Group|0x1E|Discrete Input Point|0x08|
|Discrete Output Point|0x09|Electrical Energy|0x4F|
|Event Log|0x41|Group|0x12|
|Motion Device Axis|0x42|Motor Data|0x28|
|Non-Electrical Energy|0x50|Overload|0x2C|
|Position Controller|0x25|Position Controller Supervisor|0x24|
|Position Sensor|0x23|Power Curtailment Object|0x5C|
|Power Management Object|0x53|Presence Sensing|0x0E|
|S-Analog Actuator|0x32|S-Analog Sensor|0x31|
|S-Device Supervisor|0x30|S-Gas Calibration|0x34|
|S-Partial Pressure|0x38|S-Sensor Calibration|0x40|
|S-Single Stage Controller|0x33|Safety Analog Input Group|0x4A|
|Safety Analog Input Point|0x49|Safety Dual Channel Feedback Object|0x59|
|Safety Feedback Object|0x5A|Safety Discrete Input Group|0x3E|
|Safety Discrete Input Point|0x3D|Safety Discrete Output Group|0x3C|
|Safety Discrete Output Point|0x3B|Safety Dual Channel Analog Input|0x4B|
|Safety Dual Channel Output|0x3F|Safety Limit Functions Object|0x5B|
|Safety Stop Functions Object|0x5A|Safety Supervisor|0x39|
|Safety Validator|0x3A|Softstart|0x2D|
|Target Connection List|0x4D|Time Sync|0x43|
|Trip Point|0x35|||

对于特定网络：

|名称|代码|名称|代码|
|---|---|---|---|
|Base Switch|0x51|CompoNet Link|0xF7|
|CompoNet Repeater|0xF8|ControlNet|0xF0|
|ControlNet Keeper|0xF1|ControlNet Scheduling|0xF2|
|Device Level Ring|DLR|0x47|DeviceNet|
|Ethernet Link|0xF6|Modbus|0x44|
|Modbus Serial Link|0x46|Parallel Redundancy Protocol|0x56|
|Power Management|0x53|PRP Nodes Table|0x57|
|SERCOS III Link|0x4C|SNMP|0x52|
|QoS|0x48|RSTP Bridge|0x54|
|RSTP Port|0x55|TCP/IP Interface|0xF5|

#### 3.1 身份对象 （Class ID: 0X01）

该对象包含了设备的基础身份信息，其主要属性包括： 产商ID， 设备类型，产品码，修订版本，当前状态，序列码，产品名。可选或条件属性包括：状态，配置一致性，活动间隔，当前语言，支持语言列表，国际产品名，信号量，指定名称，指定描述，地理位置，Modbus身份信息，保护模式。

#### 3.2 参数对象 （Class ID: 0x0F）

该对象包含两类：完整对象以及缩写版本。缩写版本主要用于仅有有限内存的DeviceNet以及CompoNet设备。  
该对象的设计目的是提供一种不需要工具即可允许访问其它诸多对象的方法。  
该对象的属性包含当前设备存在的实例总数，类描述。

#### 3.3 集成对象 （Class ID: 0X04）

该对象提供了将来自不同类的不同实列的属性映射进集成对象的单一属性的选项。这样可以优化I/O消息交换的效率。（减少请求以及信息传递）


## 三、CIP 与 EtherNet/IP

### 1. EtherNet/IP 格式

下图为EtherNet/IP的一般结构，EtherNet/IP 为 传输层中的内容。EtherNet/IP 可分为两个部分：封装头部，以及封装数据。CIP指令包含于封装数据中。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/75c6a66e6e754907d8f46379c4d1a8f8.png)

下图为EtherNet/IP的头部格式，其中包括指令，长度信息，会话句柄，状态，等  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/8c2b9ea14fa1429abcc44c24775d1e6e.png)

### 2. EtherNet/IP 主要指令

设备发现 （ListIdentity Command ），该指令通过UDP 广播发送给所有网络中的设备，接收到消息并且支持EtherNet/IP的设备会返回自身的身份信息。

注册会话 （RegisterSession/UnRegisterSession Commands），该指令用于注册或注销会话。会话注册之后，设备才能够进行数据交换，两台设备之间同时存在一组会话。发起请求后，服务器会返回一个Session Handle，后续交流需要使用该Session Handle 的值方可交流。

SendRRData/SendUnitData Commands，SendRRData 用于发送未建立CIP连接的显性数据，SendUnitData用于发送连接了的显性数据。发送RRdata时需要使用Sender Context，发送UnitData时则不需要。

### 3. EtherNet/IP 封装数据一般格式

一般都是两个Item，一个地址项，一个数据项，随后是具体CIP命令。每个项的前2bytes用于申明该项的类型，后2bytes用于申明该项值的长度。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/e8428c1d15240332fb9a3120ed204cd2.png)

### 4. 封装数据的使用

#### 4.1 显性信息传输 （explicit Messaging）

显性传输分为 已连接与未连接传输。其区别在于，建立了连接之后，设备会保留管理连接所需要的资源，可以提高设备响应效率。隐藏未连接的显性传输通常仅在应用程序要求不规则且不频繁的情况下使用。

下图为WireShark 实例  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/35c4b84d005691870b5337512064731f.png)  
在下图中，CIP 协议由服务请求，请求对象类型，对象实例构成，最后是请求的数据，这里Wireshark自动识别出了数据格式符合CIP connection manager的格式。在其它环境下，数据值可能是厂家自行设定的，Wireshark不一定能识别出来，就会显示一串十六进制值。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/27ef6ef2e239bd7edcbf7a836399732f.png)

#### 4.2 隐性信息传输 （implicit / I/O Messaging）

I/O 信息基于UDP/IP 发送，强调高实时性，该传输方式不需要封装头部，其格式如下所示。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/6b1d41b6df952dac7732a8b446f43f87.png)  
下图时WireShark 实例，请对照上图。  
![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/3d522e8b646d46277243f4923d6ff5e4.png)  
其中CIP Sequence Count，在一般情况不影响设备的交流，该项是用于CIP Safety 使用的。

---

## 原始资料：未命名 2.md（从本次会话已读取内容恢复）

1. EtherNet/IP 设备类型

![[Pasted image 20261009195224.png]]

为了方便设备厂商进行EtherNet/IP 设备的开发，或是开发商选用合适的EtherNet/IP设备，ODVA为EtherNet/IP设备制定了一套明确的设备类型标准，每一个类型都界定了一个功能的范围：

Level 1：显示消息服务器

·仅用于显式消息;

·示例：程序上传、数据收集器、状态监视……

Level 2：I/O消息服务器

·类 似于Level 1，但具有循环I/O消息；

·响应显式和隐式消息；

·示例：现场设备、旋转编码器、驱动器等；

Level 3：显式消息客户端/服务器

·类似于Level 1，但是有客户端用于显式消息；

·能发送、接收显式消息；

·示例：编码器、SCADA设备等；

Level 4：I/O消息客户端/服务器

·类似于Level 1、Level 2、Level 3，加上客户端用于隐式IO参考；

·可以发送和接收显式和隐式消息；

·示例：PLC、softPLC、I/O Scanner 路由器等；

[EtherNET/IP协议基础知识（上）_ehternetip-CSDN博客](https://blog.csdn.net/weixin_47677928/article/details/108398509)

[EtherNET/IP协议基础知识（下）_ethernet／ip协议-CSDN博客](https://blog.csdn.net/weixin_47677928/article/details/108399288?ops_request_misc=elastic_search_misc&request_id=a3f5074227f3c185c124c432cc0aa8d0&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-3-108399288-null-null.142^v102^pc_search_result_base2&utm_term=EtherNET%2FIP%E5%8D%8F%E8%AE%AE%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86%EF%BC%88%E4%B8%8A%EF%BC%89&spm=1018.2226.3001.4187)

[EtherNet-IP中文版详解：一网打尽协议基础与应用秘籍 - CSDN文库](https://wenku.csdn.net/column/rhp4krhtkd#1.%20EtherNet%2FIP%E5%8D%8F%E8%AE%AE%E6%A6%82%E8%BF%B0)

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

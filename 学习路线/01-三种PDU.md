# 三种 PDU：I-PDU、N-PDU、L-PDU

PDU 是 Protocol Data Unit，某一层协议眼里的一包数据。AUTOSAR 用前缀标明这包数据站在哪一层。同一段字节走到下一层，名字就换一次。

## 学完应能做到

1. 说出 I、N、L 各自对应哪一层，以及 CAN 栈里谁在处理它。
2. 分开两条发送路径：放得进一帧的信号，和必须分段的长报文。
3. 分清三件事：COM 拆装信号，PduR 按编号转发，CanTp 才做分段。

## 先记住的关系

一层的 PDU，到了下一层就是那一层的 SDU（Service Data Unit，交给下层去传、语义不变的用户数据）。下层若要自己的控制信息，再在前面加上 PCI（Protocol Control Information）。

写成式子就是：**(X)-PDU = (X-1)-SDU**。

因此不是每条报文都三种 PDU 齐全。放得进一个链路帧时，没有 N-PDU。只有走传输协议时，才会把一条 I-PDU 切成多个 N-PDU，再各自装进 L-PDU。

## 三个名字

用语取自 AUTOSAR Classic Platform 的 Communication Stack Types，以及 COM、PDU Router、CAN Transport Layer 的规范。CAN Transport Layer 把前缀说得更贴 CAN 模块，和总的分层用语是对齐的。

| 名字 | 层 | 在 AUTOSAR 里谁在看它 | 里面有什么 |
| --- | --- | --- | --- |
| I-PDU | 交互层。规范把交互层等同于 Communication Services | PduR 正上方的模块，例如 COM、DCM | COM 的 I-PDU 由一个或多个信号排成。DCM 交给 PduR 的诊断报文，对 PduR 来说也是 I-PDU |
| N-PDU | 网络层。规范把网络层等同于 Transport Protocol | CanTp 等传输层模块，介于 PduR 与 CanIf 之间 | 标识、长度，以及 PCI 加上整段或一段 N-SDU。CAN 上分成 SF、FF、CF、FC |
| L-PDU | 数据链路层。规范把数据链路层等同于通信硬件抽象和微控制器抽象 | 落到总线上的那一帧。CAN 上由 CanIf 与 CAN 驱动一起完成 | 标识、DLC、数据。数据就是 L-SDU |

CAN Transport Layer 里这几条对照，最容易把名字记混：

- **CAN N-SDU**：从 PDU Router 交给 CanTp 的那一整段数据。
- **CAN N-PDU**：CanTp 发出的一帧传输协议数据，PCI 加上整个 N-SDU 或其中一段。
- **CAN L-SDU**：同一段字节从 CanIf 看出去的名字。规范写明它和 CAN N-PDU 相似，只是观察点换成了 CanIf。

所以 N-PDU 和 L-SDU 常常是同一段字节的两个名字，不是总线上的两份拷贝。

四种 CAN N-PDU：

| 缩写 | 名称 | 作用 |
| --- | --- | --- |
| SF | Single Frame，单帧 | 数据短到一帧能装完 |
| FF | First Frame，首帧 | 告诉对方总长度，并带上第一段数据 |
| CF | Consecutive Frame，连续帧 | 按序号送后续的段 |
| FC | Flow Control，流控帧 | 接收方告诉发送方还能收多少、隔多久 |

## 字节在模块之间怎么走

下面以 CAN 为例。PduR 用编译期配好的 I-PDU ID 决定交给谁，不在运行时根据载荷改道。路由动作本身不修改 I-PDU，只是转发。传输层网关可以在整包收完之前就开始往下送，那是转发的时机，不是去改载荷。

### 短报文：一个 L-PDU 装得下

车速这类周期信号通常走这条路。

1. 软件组件通过 RTE 把信号交给 COM。
2. COM 按配置把信号排进一条 I-PDU。需要一起保持一致的若干信号可以组成信号组。
3. COM 调用 PduR，把这条 I-PDU 交下去。
4. PduR 查表，交给 CanIf。
5. CanIf 按 PDU ID 找到 CAN ID 和长度，驱动把标识、DLC 和数据组成 L-PDU 发到总线。

这条路上没有 CanTp，也就没有 N-PDU。I-PDU 的字节直接成为 L-SDU。

### 长报文：一个 L-PDU 装不下

诊断读一个很长的 DID，或 COM 里的 Large I-PDU，走这条路。Large I-PDU 指放不进底层协议单个 L-PDU 的 I-PDU。COM 仍按 I-PDU 管理它，收发改走 TP 接口。

1. DCM 或 COM 把整条 I-PDU 交给 PduR。
2. PduR 把这整段作为 N-SDU 交给 CanTp。
3. CanTp 加上 PCI，切成 SF，或 FF 加若干 CF，并处理对端的 FC。每一段是一个 N-PDU。
4. 每个 N-PDU 交给 CanIf。对 CanIf 来说，这段字节是一个 L-SDU。
5. 驱动为每一段各发一个 L-PDU。

接收把方向反过来。CanIf 按标识把收到的帧交给 PduR（普通 I-PDU）或 CanTp（传输协议帧），由配置决定。CanTp 去掉 PCI、按序号拼回整段，再经 PduR 交给 DCM 或 COM。COM 最后把 I-PDU 拆成信号，通知 RTE。

发送完成后，确认从驱动向上返回。COM 因此知道自己请求的发送已经完成，这和「对方应用已经读到信号」不是同一件事。

## 缓冲区和总线上的帧

模块之间传递的 `PduInfoType` 是载荷指针加长度。多数 CAN 帧的标识不在这块载荷里，而在 CanIf 的配置表里，用 PDU ID 查出来。J1939 这类帧才会把标识放进元数据，跟载荷一起走。

所以画图时可以把 L-PDU 画成「ID + DLC + 数据」，同时要记住：软件里向上递交的往往只有数据那一段。

经典 CAN 的数据段最长 8 字节，CAN FD 最长 64 字节，FlexRay 更长。一条 I-PDU 算不算「长」，看它能不能放进这条总线的一个 L-PDU，不是看一个固定的 8。

## I-PDU 组

I-PDU 组是 COM 里同一方向（全是发送，或全是接收）的一组 I-PDU，用来成组启动和停止。现行 COM 规范的规则是：

1. 一条 I-PDU 可以属于任意 I-PDU 组。
2. 它是启动的，当它所属的至少一个组已启动；若它不属于任何组，则按规范隐式启动。
3. 组的数量上限在预编译时配置。
4. 一个组里不能混放送方向和接收方向的 I-PDU。

初始化之后，所有 I-PDU 组默认处于停止。组的嵌套在 COM 里只是概念，配置时要展开。

## 不要沿用的旧说法

| 旧讲义 | 现行规范里的说法 |
| --- | --- |
| 非 TP 的 I-PDU 不得超过 8 字节 | 这是早期经典 CAN 的限制。放不进单个 L-PDU 的称为 Large I-PDU，走 TP。CAN FD 的单帧数据可以到 64 字节 |
| 每条 I-PDU 必须且只能属于一个 I-PDU 组 | 可以属于任意多个组，也可以不属于任何组 |
| PduR 会把信号拆开，或负责 ISO-TP 分段 | PduR 转发 I-PDU。拆信号是 COM，分段是 CanTp |
| 有 I-PDU 就一定有 N-PDU | 只有走传输协议的路径才产生 N-PDU |

## 自测

1. 一帧里的车速信号，发送路径上会出现 N-PDU 吗？
2. 诊断仪要读 200 字节，谁把 I-PDU 切成多帧？
3. PduR 转发时会改写 I-PDU 的载荷吗？
4. 同一段字节，CanTp 和 CanIf 各叫它什么？
5. 一条发送 I-PDU 可以放进接收组吗？

## 答案

1. 不会，只要这帧装得下。路径是 COM → PduR → CanIf → L-PDU。
2. CanTp。DCM 交出整条 I-PDU，PduR 把它交给 CanTp，CanTp 做成 FF/CF，并处理 FC。
3. 不改。它按静态 I-PDU ID 转发。
4. 已经组好的那一帧，CanTp 叫 N-PDU，CanIf 叫 L-SDU。从 PduR 交下来、还没加 PCI 的整段，在 CanTp 里叫 N-SDU。
5. 不可以。一个 I-PDU 组只能收纳同一方向的 I-PDU。

## 这一节还没展开

信号在 I-PDU 里的起始位、发送模式、截止时间监视、更新位、I-PDU 复用（IpduM）、CanTp 的填充字节和超时（N_As、N_Bs、N_Cr）。这些留在后面的路线里，不在这一篇展开。

## 依据

公开规范，阅读时以所用发行版的 PDF 为准。上面的定义对照的是这些文档里的术语表和 PDU 路由章节，不是逐句摘录。

- AUTOSAR Classic Platform，Specification of Communication Stack Types：I-PDU、N-PDU、L-PDU 与各层的对应
- Specification of Communication（COM）：I-PDU 承载信号；Large I-PDU 与 Large Signal；I-PDU 组
- Specification of PDU Router：按静态 ID 转发 I-PDU，转发不修改 I-PDU
- Specification of CAN Transport Layer：N-PDU、N-SDU、L-SDU，以及 SF / FF / CF / FC

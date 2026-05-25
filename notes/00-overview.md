# 00. 总览

## 目的

本知识库用于系统整理 AMBA AXI 相关知识，覆盖协议基础、进阶规则、SoC 集成、RTL 设计模式、验证方法、性能分析和常见 bug。

AXI 不是一个单纯的“总线信号表”。在实际 SoC 中，它通常是 CPU、DMA、GPU、NPU、ISP、DDR controller、SRAM controller、外设桥接模块和 NoC/interconnect 之间的数据交换协议。理解 AXI 需要同时看三层：协议规则、微架构实现、系统集成。

## 推荐学习路径

1. AMBA family: 先理解 APB、AHB、AXI、ACE、CHI 的定位。
2. AXI 基础: transaction、transfer、master、subordinate、五个 channel。
3. 握手机制: `VALID`/`READY` 是 AXI 所有 channel 的基础。
4. burst: 理解 `AxLEN`、`AxSIZE`、`AxBURST`、`WSTRB`、`LAST`。
5. ordering: 理解 `AxID`、outstanding transaction 和返回顺序。
6. SoC 实践: interconnect、crossbar、NoC、bridge、CDC、width conversion、DMA、DDR。
7. 验证和调试: assertion、VIP、scoreboard、coverage、常见死锁和协议违规。

## AXI 在 SoC 中的位置

典型 SoC 中，AXI 会出现在这些路径上：

```text
CPU cluster  -> interconnect/NoC -> DDR controller
CPU cluster  -> AXI-to-APB bridge -> peripheral registers
DMA          -> interconnect/NoC -> DDR controller
NPU/GPU/ISP  -> interconnect/NoC -> SRAM/DDR
debug block  -> AXI access port -> memory/peripheral
```

AXI4 常用于 high-throughput memory-mapped 数据访问。AXI4-Lite 常用于低带宽寄存器访问。AXI4-Stream 常用于无地址的数据流，例如视频、网络包、DMA stream path。

## 三个核心抽象

### Transaction

Transaction 是一次完整访问请求。例如一次 AXI write transaction 包含写地址、写数据和写响应；一次 read transaction 包含读地址和读数据响应。

### Transfer

Transfer 是某个 channel 上一次 `VALID && READY` 成立的数据交接。一个 burst transaction 会包含多个 data transfer，也就是多个 beat。

### Channel

AXI 将地址、数据和响应拆分成多个独立 channel。AXI4 memory-mapped interface 有五个 channel：`AW`、`W`、`B`、`AR`、`R`。

## 为什么 AXI 适合高性能 SoC

AXI 的设计目标之一是支持高吞吐和高并发：

- 读写地址通道分离，读写数据通道分离。
- 地址和数据可解耦，interconnect 可以流水化处理。
- burst transaction 降低地址开销，提升 DDR/SRAM 访问效率。
- ID 支持多个 outstanding transactions。
- sideband signals 支持 cache、protection、QoS、region、user-defined 信息。

PULP AXI 项目的资料也体现了实际工程中对 topology independence、modularity、heterogeneous networks、data width conversion、ID width conversion 和 transaction concurrency 的重视。

## 一个最小读写例子

AXI4-Lite 写一个外设寄存器：

```text
1. Master 在 AW channel 发出写地址，例如 0x4000_0010。
2. Master 在 W channel 发出写数据和 WSTRB。
3. Subordinate 接收地址和数据后执行寄存器写。
4. Subordinate 在 B channel 返回写响应，例如 OKAY。
```

AXI4 从 DDR 读取 4 个 beat：

```text
1. Master 在 AR channel 发出读地址和 burst 属性。
2. Subordinate 或 DDR controller 接收请求。
3. Subordinate 在 R channel 返回 4 个 beat。
4. 最后一个 beat 需要 `RLAST=1`。
```

## 本知识库的引用原则

- 协议规则优先引用 Arm 官方规格和官方入口。
- 工程实践引用 AMD/Xilinx product guide、PULP AXI paper、开源 RTL 项目文档。
- 博客和开源代码只作为辅助理解，不作为协议合法性的唯一依据。

## 后续重点

- 补齐 Arm `IHI0022` AXI/ACE 官方规格 PDF。
- 补齐 AXI4-Stream 官方规格资料。
- 在每个主题中增加具体 timing diagram、RTL 片段和 debug checklist。

## 参考资料

- `arm-amba-architecture-page`
- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `pulp-platform-axi-microarchitecture-paper`
- `xilinx-pg059-axi-interconnect`
- `xilinx-pg021-axi-dma`

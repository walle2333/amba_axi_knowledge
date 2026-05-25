# 01. AMBA Family

## 目的

AMBA 是 Arm 定义的一组片上互连协议族。AXI 是 AMBA family 中面向高性能 memory-mapped 访问的协议。理解 AXI 前，需要先明确 APB、AHB、AXI、ACE、CHI 的定位和差异。

## AMBA 协议族概览

| 协议 | 典型定位 | 常见用途 |
| --- | --- | --- |
| APB | 低复杂度、低功耗外设总线 | UART、SPI、I2C、timer、GPIO、控制寄存器 |
| AHB | 较早期高性能总线 | MCU、legacy subsystem、简单片上 SRAM 访问 |
| AXI | 高性能 memory-mapped interface | CPU/DMA/accelerator 到 DDR/SRAM/interconnect |
| AXI4-Lite | AXI4 简化子集 | 外设寄存器访问、控制面 |
| AXI4-Stream | 无地址流式接口 | 视频、网络、DSP、DMA stream |
| ACE/ACE-Lite | AXI coherency extension | cache coherent master、I/O coherency |
| CHI | 现代 coherent interconnect interface | 多核 CPU cluster、coherent NoC、复杂 SoC |

## APB

APB 适合简单外设寄存器访问，协议复杂度低，不追求高吞吐。APB 常作为 AXI-to-APB bridge 后面的 peripheral bus。

典型路径：

```text
CPU -> AXI interconnect -> AXI-to-APB bridge -> UART/SPI/GPIO registers
```

SoC 中不建议让低速外设直接挂在高性能 AXI fabric 上，否则会浪费互连资源，也可能增加 timing 和 power 压力。

## AHB

AHB 是较早期的 high-performance bus，仍可能出现在 legacy subsystem 或 MCU 类设计中。现代复杂应用处理器 SoC 更常使用 AXI/ACE/CHI。

## AXI

AXI 是高性能 memory-mapped interface。它通过五个独立 channel 分离地址、数据和响应，支持 burst、outstanding transactions、ID、sideband attributes，适合高带宽和高并发场景。

典型 master：

```text
CPU cluster
DMA
GPU
NPU
ISP
display controller
debug access port
```

典型 subordinate：

```text
DDR controller
SRAM controller
register block
AXI-to-APB bridge
interconnect target port
```

## AXI4-Lite

AXI4-Lite 是 AXI4 的简化 memory-mapped 子集，通常用于控制寄存器访问。它不支持 burst，单次事务只传输一个 data beat，适合控制面，不适合大块数据搬运。

例子：

```text
CPU 写 DMA 控制寄存器 -> AXI4-Lite
DMA 搬运 DDR 数据       -> AXI4 memory-mapped
DMA 输出视频流          -> AXI4-Stream
```

## AXI4-Stream

AXI4-Stream 没有地址通道，核心是 `TVALID`/`TREADY` 和 `TDATA`。它适合连续数据流，不适合随机地址访问。

典型场景：

```text
camera input -> ISP -> video encoder
Ethernet MAC -> packet processing pipeline
DMA memory-mapped read -> AXI4-Stream output
```

## ACE 和 ACE-Lite

ACE 是 AXI coherency extension，用于让多个 cache master 保持一致性。ACE-Lite 常用于 I/O coherent master，例如某些 DMA 或 accelerator，可以参与有限的 cache coherency 交互。

是否需要 ACE/ACE-Lite，取决于系统是否需要硬件 cache coherency。如果 DMA 是 non-coherent，软件通常需要显式执行 cache clean/invalidate。

## CHI

CHI 是更现代的 coherent interface，常见于多核、高性能、复杂 coherent interconnect 或 NoC 系统。它和 AXI 的关系不是简单替代全部场景，而是在 coherent fabric 中承担更复杂的一致性和事务管理。

## SoC 设计取舍

实际 SoC 不会所有地方都用 AXI4 full interface。常见分层是：

```text
高带宽数据面: AXI4 / CHI / NoC
控制寄存器面: AXI4-Lite
低速外设面: APB
流数据面: AXI4-Stream
coherent CPU/accelerator: ACE/ACE-Lite/CHI
```

设计时要避免两个极端：

- 把所有模块都接 AXI4 full，导致面积、功耗、验证复杂度过高。
- 把高带宽模块接到过窄、过慢或无 outstanding 能力的通路，导致性能瓶颈。

## 参考资料

- `arm-amba-architecture-page`
- `arm-amba-axi-ace-protocol-spec`
- `arm-amba-chi-protocol-spec`
- `xilinx-pg059-axi-interconnect`
- `pulp-platform-axi-readme`

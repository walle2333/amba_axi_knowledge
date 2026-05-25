# 12. Interconnect, Crossbar and NoC

## 目的

AXI interconnect 是 SoC 中连接多个 master 和多个 subordinate 的核心基础设施。它不仅仅是 mux/demux，还需要处理地址 decode、仲裁、response routing、ID 扩展、ordering、QoS、backpressure、错误响应和死锁风险。

## 基本形态

常见互连形态：

| 形态 | 特点 | 典型用途 |
| --- | --- | --- |
| shared interconnect | 多个 master 共享通路，面积小，并发低 | 小型子系统、低速外设 |
| crossbar | 多 master 到多 slave，可支持更高并发 | 中型 SoC、FPGA system |
| NoC | packetized/networked fabric，可扩展性强 | 大型 SoC、多 cluster、高带宽系统 |

alexforencich 文档中区分了 shared `axi_interconnect` 和 nonblocking `axi_crossbar`。PULP AXI 文档强调 topology independence、modularity 和 heterogeneous networks，说明真实 SoC 通常需要按性能、面积、时序和功耗做分层互连。

## 典型 SoC 结构

```text
CPU cluster ----\
DMA -------------> AXI interconnect / NoC -> DDR controller
GPU/NPU --------/

CPU cluster -----> AXI4-Lite fabric -> AXI-to-APB bridge -> peripherals
```

数据面通常使用 AXI4 full 或 CHI/NoC。控制面通常使用 AXI4-Lite 和 APB。

## Address Decode

Interconnect 根据 `AxADDR` 选择目标 subordinate。

例子：

```text
0x0000_0000 - 0x0FFF_FFFF  Boot ROM / SRAM
0x4000_0000 - 0x4FFF_FFFF  Peripheral region
0x8000_0000 - 0xFFFF_FFFF  DDR
```

Decode 设计要点：

- 地址区间不能意外重叠。
- 每个合法地址必须有唯一目标，除非明确支持 alias。
- 未命中地址应路由到 default error slave 或由 interconnect 返回 `DECERR`。
- Burst 不能跨越目标 region，也不能跨越 AXI 4KB boundary。

## Arbitration

当多个 master 同时访问同一个 slave，需要仲裁。

常见策略：

| 策略 | 特点 |
| --- | --- |
| fixed priority | 简单，但低优先级可能 starvation |
| round robin | 公平性较好 |
| weighted round robin | 可按带宽需求分配权重 |
| QoS-aware | 根据 `AxQOS` 或系统策略调度 |

仲裁必须与 ordering 约束配合。不能为了性能破坏同 ID ordering，也不能让某个 master 永久得不到服务。

## Read Path 和 Write Path

AXI read 和 write 是相对独立的路径。

Read path：

```text
AR decode/arbitration -> route to slave -> R response route back master
```

Write path：

```text
AW decode/arbitration -> W data route -> B response route back master
```

Write path 更复杂，因为 AW 和 W 是独立 channel，但必须指向同一个目标并保持事务关联。

## Response Routing

Interconnect 必须知道每个 outstanding transaction 的 response 应返回哪个 master。

常见 tracking 信息：

```text
upstream master port
upstream ID
downstream slave port
downstream ID
burst length / remaining beats
ordering state
```

如果 response routing 错误，master 会收到不属于自己的 `RID/BID`，这是严重协议错误。

## ID Extension

多个 master 连接到同一个 downstream slave 时，不同 master 可能使用相同 ID。Interconnect 常通过给 ID 拼接 master port 编号来避免冲突。

```text
M0 ARID=3 -> downstream ID={0, 3}
M1 ARID=3 -> downstream ID={1, 3}
```

返回时再剥离扩展位并路由回正确 master。

## Ordering Protection

Interconnect 需要保护 AXI ordering 规则。

典型要求：

- 同 ID transaction 不应被错误重排。
- ID remap 后不能破坏上游 ordering 语义。
- 同一 master 到不同 slave 的事务是否需要保序，取决于协议、属性和系统设计。
- Decode error 也要按正确 ID 和通道返回。

alexforencich 的 AXI crossbar 文档明确提到 ID-based transaction ordering protection logic，这是 crossbar 设计的关键能力。

## Backpressure 传播

Interconnect 必须正确传播 backpressure。

例子：DDR controller 暂停接收 W data：

```text
DDR WREADY=0
-> interconnect downstream buffer full
-> upstream WREADY=0
-> DMA 暂停发送 WDATA
```

Backpressure 不能导致 payload 丢失，也不能形成循环依赖死锁。

## Deadlock 风险

常见死锁来源：

- AW path 等 W path，W path 又等 AW path。
- Response buffer 满，导致 request path 无法释放资源。
- 多个 channel 之间存在循环 backpressure。
- NoC virtual channel 或 credit 配置错误。
- Clock/reset 不一致导致某个端口永远不 ready。

设计时应明确每个 channel 的 buffer 深度、资源申请顺序和释放条件。

## QoS

`AxQOS` 可作为服务质量或优先级提示。实际是否使用取决于 interconnect/NoC 实现。

典型场景：

- display/ISP 需要稳定带宽，避免 underrun。
- CPU latency-sensitive transaction 需要较低延迟。
- DMA bulk copy 可以较低优先级。

QoS 不是魔法，必须与仲裁策略、DDR scheduler、NoC flow control 一起设计。

## Verification Checklist

- 地址 decode 是否唯一、无遗漏、无意外重叠。
- Decode miss 是否返回 `DECERR`。
- Burst 是否不会跨 slave region 或 4KB boundary。
- 多 master 同 ID 是否通过 ID extension/remap 区分。
- Response 是否路由回正确 master。
- 同 ID ordering 是否保持。
- Random backpressure 下是否无死锁。
- Arbitration 是否满足公平性和 QoS 需求。
- Reset 后 outstanding tracking table 是否清空或安全 drain。

## 常见 bug

- 未映射地址无响应，总线挂死。
- 多 master ID 冲突，response 路由错误。
- Crossbar 允许 burst 跨 region。
- AW 和 W 路由到不同 slave。
- Response buffer 太小，在压力下死锁。
- QoS 信号接了但仲裁完全忽略，系统性能不符合预期。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `xilinx-pg059-axi-interconnect`
- `pulp-platform-axi-readme`
- `pulp-platform-axi-microarchitecture-paper`
- `alexforencich-verilog-axi-readme`
- `zipcpu-wb2axip-readme`

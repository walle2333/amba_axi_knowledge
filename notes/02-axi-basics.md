# 02. AXI 基础

## 目的

本篇整理 AXI memory-mapped interface 的基本概念：master、subordinate、transaction、transfer、channel、burst、response 和 sideband attributes。

## 基本角色

### Manager / Master

发起 transaction 的一方。很多资料仍使用 master 这个传统术语；Arm 新文档中也会使用 manager。为便于阅读，本知识库主要使用 master，并在必要时注明 manager。

典型 master：CPU、DMA、GPU、NPU、debug module。

### Subordinate / Slave

响应 transaction 的一方。很多资料仍使用 slave；Arm 新文档中也会使用 subordinate。为便于阅读，本知识库主要使用 subordinate，并在必要时注明 slave。

典型 subordinate：DDR controller、SRAM、register block、AXI-to-APB bridge。

## Transaction 与 Transfer

Transaction 是一次完整请求。Transfer 是 channel 上一次握手成功的数据交接。

例子：一个 4-beat write burst transaction：

```text
AW channel: 1 次 address transfer
W channel : 4 次 data transfer
B channel : 1 次 response transfer
```

例子：一个 4-beat read burst transaction：

```text
AR channel: 1 次 address transfer
R channel : 4 次 data transfer，最后一次 RLAST=1
```

## AXI4 五个通道

AXI4 memory-mapped interface 有五个独立 channel：

| Channel | 方向 | 作用 |
| --- | --- | --- |
| AW | master -> subordinate | Write address |
| W | master -> subordinate | Write data |
| B | subordinate -> master | Write response |
| AR | master -> subordinate | Read address |
| R | subordinate -> master | Read data and read response |

每个 channel 使用自己的 `VALID`/`READY` 握手。通道之间相互独立，但一个完整 transaction 在语义上需要多个 channel 协同完成。

## 写事务基本流程

```text
Master                         Subordinate
  | -- AWADDR/AWLEN/... ------> |
  | -- WDATA/WSTRB/WLAST -----> |
  | <------------- BRESP/BID -- |
```

关键点：

- `AW` 和 `W` 是独立 channel，地址和数据不要求同周期到达。
- subordinate 必须能正确关联写地址和写数据。
- `B` response 表示写事务在 subordinate 侧的完成结果。

## 读事务基本流程

```text
Master                         Subordinate
  | -- ARADDR/ARLEN/... ------> |
  | <--- RDATA/RRESP/RLAST ---- |
```

关键点：

- 一次读地址请求可能返回多个 R beat。
- 每个 R beat 都带 `RRESP`。
- 最后一个 R beat 必须标记 `RLAST=1`。

## Burst 基础

AXI4 支持 burst transaction。一次地址传输后，可以连续传输多个 data beat。

常用字段：

| 字段 | 含义 |
| --- | --- |
| `AxLEN` | burst length minus 1，实际 beat 数为 `AxLEN + 1` |
| `AxSIZE` | 每个 beat 的字节数，编码为 `log2(bytes_per_beat)` |
| `AxBURST` | burst 类型，如 FIXED、INCR、WRAP |
| `WLAST` | write burst 最后一个 data beat |
| `RLAST` | read burst 最后一个 data beat |

例子：64-bit 数据总线，4-beat INCR burst：

```text
AxSIZE = 3    // 2^3 = 8 bytes per beat
AxLEN  = 3    // 3 + 1 = 4 beats
total  = 32 bytes
```

## Response 基础

AXI response 表示访问结果。常见响应包括：

| Response | 含义 |
| --- | --- |
| OKAY | 正常访问成功 |
| EXOKAY | exclusive access 成功 |
| SLVERR | subordinate 侧错误 |
| DECERR | decode error，常见于地址没有映射到有效目标 |

写响应在 `B` channel 上返回，读响应在 `R` channel 上随每个 data beat 返回。

## Sideband Attributes

AXI 地址通道上包含多个 sideband attributes，常见包括：

| 字段 | 作用 |
| --- | --- |
| `AxID` | transaction ID，用于 outstanding 和 ordering |
| `AxPROT` | protection/security/privilege 属性 |
| `AxCACHE` | cacheability、bufferability 等属性 |
| `AxQOS` | QoS/priority hint |
| `AxREGION` | region 标识 |
| `AxUSER` | 用户自定义 sideband |

这些信号在简单外设中可能被忽略，但在 CPU、DMA、NoC、DDR、TrustZone、cache coherency 场景中会影响系统行为。

## 最小 AXI4-Lite 寄存器访问示例

CPU 写控制寄存器：

```text
地址: 0x4000_0010
数据: 0x0000_0001
WSTRB: 0b1111
BRESP: OKAY
```

语义：CPU 对外设控制寄存器执行一次 32-bit 写，使能某个功能。AXI4-Lite 不使用 burst，所以 `AWLEN` 这类 AXI4 full burst 字段不存在或固定化。

## 常见误区

- 误以为 `AW` 和 `W` 必须同周期握手。实际它们是独立 channel。
- 误以为一次 transaction 等于一次 transfer。burst transaction 包含多个 transfer。
- 误以为 `READY` 必须一直为 1。实际接收方可以通过 `READY=0` 施加 backpressure。
- 误把 AXI4-Lite 当成 AXI4 full 的等价替代。AXI4-Lite 不适合大块数据搬运。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `xilinx-pg021-axi-dma`

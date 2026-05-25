# 10. AXI4-Stream

## 目的

AXI4-Stream 是 AXI 协议族中面向流数据的接口。它没有地址通道，不表达 memory-mapped 访问，而是用 `TVALID`/`TREADY` 在 source 和 sink 之间传输连续数据、packet 或 frame。

## 定位

AXI4-Stream 适合数据流，不适合随机地址访问。

典型场景：

```text
camera sensor -> ISP pipeline
Ethernet MAC -> packet processing
ADC sample stream -> DSP
DMA memory-mapped read -> AXI4-Stream output
AXI4-Stream input -> DMA memory-mapped write
NPU layer pipeline internal stream
```

## 与 AXI4 Memory-Mapped 的差异

| 项目 | AXI4 Memory-Mapped | AXI4-Stream |
| --- | --- | --- |
| 地址 | 有 AW/AR 地址通道 | 无地址 |
| 事务 | read/write transaction | stream transfer/frame |
| 方向 | master 读写 subordinate | source 发送到 sink |
| 响应 | B/R response | 通常无协议级 response |
| 典型用途 | DDR/SRAM/register access | packet/frame/sample stream |

## 核心信号

| 信号 | 方向 | 含义 |
| --- | --- | --- |
| `TVALID` | source -> sink | payload 有效 |
| `TREADY` | sink -> source | sink 可接收 |
| `TDATA` | source -> sink | 数据 |
| `TKEEP` | source -> sink | byte lane 有效标记 |
| `TSTRB` | source -> sink | byte qualifier，使用较少 |
| `TLAST` | source -> sink | packet/frame 最后一个 beat |
| `TID` | source -> sink | stream ID |
| `TDEST` | source -> sink | routing destination |
| `TUSER` | source -> sink | 用户自定义 sideband |

`TVALID`/`TREADY` 的基本规则与 AXI memory-mapped channel 相同：transfer 发生在 `TVALID && TREADY`。

## Packet 和 TLAST

`TLAST` 常用于标记 packet 或 frame 结束。

以 Ethernet packet 为例：

```text
beat 0: TVALID=1, TDATA=packet bytes[0..7],   TLAST=0
beat 1: TVALID=1, TDATA=packet bytes[8..15],  TLAST=0
beat N: TVALID=1, TDATA=last bytes,           TLAST=1
```

不同设计对 `TLAST` 的语义可能不同：

- 一个 Ethernet frame 的结尾。
- 一行 video line 的结尾。
- 一帧 image 的结尾。
- 一个 DMA descriptor 对应 buffer 的结尾。

接口文档必须明确 `TLAST` 的含义。

## TKEEP

`TKEEP` 表示 `TDATA` 中哪些 byte 有效，常用于最后一个 beat 不是总线宽度整数倍的场景。

64-bit stream 发送 14 bytes：

```text
beat 0: TDATA[63:0] 有 8 bytes, TKEEP=8'b1111_1111, TLAST=0
beat 1: TDATA[47:0] 有 6 bytes, TKEEP=8'b0011_1111, TLAST=1
```

如果忽略 `TKEEP`，接收方可能把无效 padding byte 当成有效数据。

## Backpressure

Sink 可以通过 `TREADY=0` 反压 source。

```text
TVALID=1, TREADY=0 时，TDATA/TKEEP/TLAST/TUSER 必须保持稳定。
```

Backpressure 常见来源：

- 下游 FIFO 满。
- DMA 写 DDR 被 W channel 反压。
- packet parser 正在处理前一个 header。
- clock domain crossing buffer 满。

## TUSER

`TUSER` 是用户自定义 sideband。常见用途：

- 错误标记，例如 bad frame。
- start-of-frame 或 metadata。
- timestamp。
- byte-level error mask。
- video format 信息。

`TUSER` 没有统一语义，必须由接口规范定义。

## TID 和 TDEST

`TID` 可表示流 ID，`TDEST` 可用于 routing。AXI4-Stream switch 或 NoC-like stream fabric 可能根据 `TDEST` 路由。

例子：

```text
TDEST=0 -> video encoder
TDEST=1 -> display pipeline
TDEST=2 -> memory writer DMA
```

## DMA 中的 AXI4-Stream

DMA 经常连接 memory-mapped AXI 和 AXI4-Stream：

```text
MM2S: DDR -> AXI4 read -> DMA -> AXI4-Stream
S2MM: AXI4-Stream -> DMA -> AXI4 write -> DDR
```

需要关注：

- Stream packet boundary 如何映射到 memory buffer。
- `TLAST` 是否用于结束 descriptor。
- `TKEEP` 如何决定最后一个 beat 写入多少 bytes。
- DDR backpressure 如何反压到 stream source。

## Verification Checklist

- `TVALID=1 && TREADY=0` 时 payload 是否稳定。
- `TLAST` 是否准确标记 packet/frame 结尾。
- `TKEEP` 是否与有效 byte 数一致。
- Backpressure 随机插入时是否丢 beat。
- Packet 连续发送时 boundary 是否错位。
- `TUSER` 错误标记是否与数据 beat 同步。
- Switch 根据 `TDEST` 路由时是否保持 packet 完整性。

## 常见 bug

- 只看 `TVALID`，忘记必须 `TVALID && TREADY` 才算 transfer。
- `TLAST` 早一拍或晚一拍。
- 最后一个 beat 的 `TKEEP` 错误。
- Backpressure 下 `TDATA` 变化。
- 跨 CDC 后 `TLAST/TUSER` 与数据错位。
- 不同模块对 `TLAST` 或 `TUSER` 语义理解不一致。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-cocotbext-axi-readme`
- `alexforencich-verilog-axi-readme`
- `zipcpu-wb2axip-readme`
- `fpganinja-taxi-readme`
- `xilinx-pg021-axi-dma`

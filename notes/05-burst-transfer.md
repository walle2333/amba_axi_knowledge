# 05. Burst Transfer

## 目的

Burst transfer 是 AXI4 提升吞吐效率的关键机制。一次地址握手后，数据通道可以连续传输多个 beat，从而减少地址通道开销，并更好匹配 DDR、SRAM、DMA 和 cache line 访问。

## 核心术语

| 术语 | 含义 |
| --- | --- |
| beat | burst 中一次数据传输 |
| burst transaction | 一次地址传输对应多个 data beat 的事务 |
| burst length | beat 数，AXI 中由 `AxLEN + 1` 表示 |
| beat size | 每个 beat 的字节数，AXI 中由 `AxSIZE` 编码 |
| burst type | 地址递增方式，由 `AxBURST` 表示 |

`Ax` 表示 AW 或 AR，也就是写地址或读地址通道。

## 关键字段

| 字段 | 所在通道 | 含义 |
| --- | --- | --- |
| `AxADDR` | AW/AR | burst 起始地址 |
| `AxLEN` | AW/AR | burst length minus 1 |
| `AxSIZE` | AW/AR | 每 beat 字节数编码，bytes = `2^AxSIZE` |
| `AxBURST` | AW/AR | FIXED、INCR、WRAP |
| `WSTRB` | W | 写数据 byte lane 有效标记 |
| `WLAST` | W | 写 burst 最后一个 beat |
| `RLAST` | R | 读 burst 最后一个 beat |

## AxLEN

`AxLEN` 表示 beat 数减 1。

| `AxLEN` | beat 数 |
| --- | --- |
| 0 | 1 |
| 1 | 2 |
| 3 | 4 |
| 15 | 16 |
| 255 | 256 |

例子：

```text
ARLEN = 3
=> read burst 有 4 个 R beat
=> 第 4 个 R beat 必须 RLAST=1
```

## AxSIZE

`AxSIZE` 编码每个 beat 的字节数：

```text
bytes_per_beat = 2 ^ AxSIZE
```

常见例子：

| `AxSIZE` | 每 beat 字节数 | 常见数据宽度 |
| --- | --- | --- |
| 0 | 1 byte | 8-bit |
| 1 | 2 bytes | 16-bit |
| 2 | 4 bytes | 32-bit |
| 3 | 8 bytes | 64-bit |
| 4 | 16 bytes | 128-bit |

在 64-bit AXI 数据总线上，最大自然 beat size 通常是 8 bytes，也就是 `AxSIZE=3`。如果 `AxSIZE` 小于总线宽度，就是 narrow transfer。

## AxBURST

AXI 常见 burst type：

| 类型 | 地址行为 | 典型用途 |
| --- | --- | --- |
| FIXED | 每个 beat 地址相同 | FIFO、设备端口 |
| INCR | 地址按 beat size 递增 | DDR、SRAM、DMA、cache line |
| WRAP | 地址递增并在边界回绕 | cache line wrap access |

### FIXED Burst

每个 beat 使用同一地址。适合访问 FIFO data register 这类端口。

```text
start address = 0x1000
beat size     = 4 bytes
length        = 4 beats
addresses     = 0x1000, 0x1000, 0x1000, 0x1000
```

### INCR Burst

地址逐 beat 递增。最常见。

```text
start address = 0x1000
beat size     = 4 bytes
length        = 4 beats
addresses     = 0x1000, 0x1004, 0x1008, 0x100C
```

### WRAP Burst

地址递增到 wrap boundary 后回绕。常用于 cache line 访问。

```text
start address = 0x1018
beat size     = 8 bytes
length        = 4 beats
wrap size     = 8 * 4 = 32 bytes
wrap base     = 0x1000
addresses     = 0x1018, 0x1000, 0x1008, 0x1010
```

## 总传输大小

```text
total_bytes = (AxLEN + 1) * (2 ^ AxSIZE)
```

例子：128-bit 数据总线，16-beat INCR burst：

```text
AxSIZE = 4    // 16 bytes per beat
AxLEN  = 15   // 16 beats
total  = 256 bytes
```

## WSTRB

`WSTRB` 表示 `WDATA` 中哪些 byte lane 有效。每一位对应一个 byte。

32-bit 写数据例子：

```text
WDATA = 0xAABB_CCDD
WSTRB = 4'b1111  => 4 个 byte 都写入
WSTRB = 4'b0011  => 只写低 2 个 byte
WSTRB = 4'b1100  => 只写高 2 个 byte
```

`WSTRB` 在 unaligned write、narrow write、partial register write 中非常重要。subordinate 必须根据 `WSTRB` 控制 byte enable，不能简单地总是写完整数据宽度。

## Narrow Transfer

Narrow transfer 指每 beat 字节数小于数据总线宽度。

例子：64-bit AXI 总线执行 32-bit transfer：

```text
DATA_WIDTH = 64 bits = 8 bytes
AWSIZE     = 2       = 4 bytes per beat
```

这时每个 beat 只使用部分 byte lane。实际有效 byte lane 由地址低位和 `WSTRB` 决定。

## Unaligned Transfer

Unaligned transfer 指起始地址没有按 transfer size 对齐。

例子：32-bit transfer 从地址 `0x1001` 开始：

```text
AWSIZE = 2  // 4 bytes per beat
AWADDR = 0x1001
```

写操作中，master 需要用 `WSTRB` 指明有效 byte。subordinate、width converter、DMA 等模块必须正确处理地址低位和 byte lane 映射。

工程上，很多 DMA 或 interconnect 支持 unaligned transfer，但也可能提供参数禁用，以节省面积和简化逻辑。alexforencich 的 AXI DMA 文档中就提到 unaligned transfer 可通过参数禁用以节省资源。

## 4KB Boundary Rule

AXI burst 不能跨越 4KB 地址边界。这条规则对 interconnect 地址译码很重要，因为很多系统以 4KB page 或 region 为基本边界。

检查方法：

```text
start_addr[31:12] == end_addr[31:12]
```

其中：

```text
end_addr = start_addr + total_bytes - 1
```

合法例子：

```text
start = 0x0000_0F00
total = 128 bytes
end   = 0x0000_0F7F
=> 没有跨越 0x1000 边界
```

非法例子：

```text
start = 0x0000_0FF0
total = 32 bytes
end   = 0x0000_100F
=> 跨越 4KB 边界
```

## WLAST 和 RLAST

写 burst 中，master 必须在最后一个 W beat 拉高 `WLAST`。

读 burst 中，subordinate 必须在最后一个 R beat 拉高 `RLAST`。

常见检查：

```text
expected_beats = AxLEN + 1
last must occur exactly on beat expected_beats - 1
```

过早 `LAST` 会截断 burst。过晚 `LAST` 会让接收方认为还有数据，导致 transaction 卡住或后续数据错位。

## SoC 实践

### DMA

DMA 通常使用 INCR burst 搬运连续内存数据。burst length 越长，地址开销越低，但过长 burst 会影响仲裁公平性和 latency。

### DDR Controller

DDR controller 更喜欢连续、对齐、较长的 burst，因为这更容易形成高效率访问。但 AXI burst 仍需满足系统边界、QoS 和 outstanding 限制。

### Interconnect

Interconnect 需要检查 burst 是否跨目标地址区间。跨 4KB 或跨 slave region 的 burst 通常应在 master 侧避免，或由转换模块拆分。

### Width Converter

Width converter 会把一个宽 beat 拆成多个窄 beat，或把多个窄 beat 合并成一个宽 beat。它必须保持 `WSTRB`、`LAST`、response 和 ordering 的语义正确。

## 常见 bug

- `AxLEN` 理解成 beat 数，而不是 beat 数减 1。
- `WLAST` 或 `RLAST` 与 beat 计数不匹配。
- burst 跨越 4KB 边界。
- `WSTRB` 没有根据 unaligned address 正确生成。
- width converter 处理 narrow transfer 时 byte lane 错误。
- interconnect 没有阻止跨 slave region 的 burst。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `xilinx-pg021-axi-dma`

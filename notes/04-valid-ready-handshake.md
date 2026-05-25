# 04. VALID/READY Handshake

## 目的

`VALID`/`READY` 是 AXI 最核心的流控机制。AXI 的每个 channel 都用同一种握手规则完成一次 transfer。本篇整理规则、时序示例、RTL 设计原则、SoC 实践和常见 bug。

## 基本规则

一次 transfer 发生在时钟上升沿，并且该周期满足：

```text
VALID == 1 && READY == 1
```

其中：

- `VALID` 由发送方驱动，表示 payload 有效。
- `READY` 由接收方驱动，表示可以接收 payload。
- `VALID && READY` 成立时，双方完成一次 payload 交接。

## 最重要的协议约束

发送方不能等待 `READY` 才拉高 `VALID`。接收方可以等待 `VALID` 才拉高 `READY`。

原因是如果双方都等待对方，系统可能死锁：

```text
sender: 等 READY=1 才 VALID=1
receiver: 等 VALID=1 才 READY=1
结果: VALID=0 且 READY=0，永远无法握手
```

## Payload 稳定性

当发送方已经拉高 `VALID`，但接收方 `READY=0` 时，payload 必须保持稳定，直到发生握手。

对于 AW channel：

```text
AWVALID=1, AWREADY=0 时，AWADDR/AWLEN/AWSIZE/AWBURST/AWID 等不能变化。
```

对于 W channel：

```text
WVALID=1, WREADY=0 时，WDATA/WSTRB/WLAST 不能变化。
```

## 基础时序示例

### 接收方立即 ready

```text
cycle:    0  1  2
VALID:   0  1  0
READY:   1  1  1
transfer    ^
```

`cycle 1` 上升沿发生一次 transfer。

### 接收方插入 wait state

```text
cycle:    0  1  2  3  4
VALID:   0  1  1  1  0
READY:   0  0  0  1  1
payload: -  A  A  A  -
transfer          ^
```

发送方在等待期间保持 `VALID=1` 和 payload `A` 不变。直到 `cycle 3` `READY=1`，transfer 才完成。

### 连续传输

```text
cycle:    0  1  2  3  4
VALID:   0  1  1  1  0
READY:   1  1  1  1  1
payload: -  A  B  C  -
transfer    ^  ^  ^
```

当 `VALID` 和 `READY` 连续为 1 时，每个周期都可以完成一个 transfer。

## Backpressure

`READY=0` 表示接收方暂时不能接收数据，这就是 backpressure。AXI 系统中的 backpressure 可以跨模块传播。

例子：DDR controller 繁忙导致 DMA 写通路反压：

```text
DDR controller WREADY=0
-> interconnect 对 DMA WREADY=0
-> DMA 暂停发送 WDATA
-> DMA 内部 FIFO 逐渐填满
-> 上游 stream source 被 TREADY=0 反压
```

设计时需要确认 backpressure 链路不会形成环路死锁。

## RTL 设计原则

发送方原则：

```verilog
// conceptual pattern
if (!valid || ready) begin
    valid <= have_payload;
    data  <= next_payload;
end
```

含义：如果当前没有有效 payload，或者当前 payload 已经被接收，发送方才可以更新 `valid` 和 `data`。

接收方原则：

```verilog
assign ready = can_accept_payload;
```

接收方可以根据 FIFO 空间、内部状态、仲裁结果决定 `READY`。但如果 `READY` 经过很长组合路径，会影响 timing，常需要 register slice 或 skid buffer。

## Skid Buffer

Skid buffer 用于打断 `READY` 组合路径，同时避免丢失一个已经被发送方推出的 payload。

典型用途：

- 高速 AXI interconnect 中切断长 ready path。
- AXI register slice。
- CDC 或 width converter 前后的缓冲。
- timing closure 困难的长链路。

## 五个 channel 中的应用

| Channel | VALID | READY | Payload 稳定对象 |
| --- | --- | --- | --- |
| AW | `AWVALID` | `AWREADY` | `AWADDR`、`AWLEN`、`AWSIZE`、`AWBURST`、`AWID` 等 |
| W | `WVALID` | `WREADY` | `WDATA`、`WSTRB`、`WLAST` |
| B | `BVALID` | `BREADY` | `BID`、`BRESP` |
| AR | `ARVALID` | `ARREADY` | `ARADDR`、`ARLEN`、`ARSIZE`、`ARBURST`、`ARID` 等 |
| R | `RVALID` | `RREADY` | `RID`、`RDATA`、`RRESP`、`RLAST` |

## Verification Checklist

- `VALID` 是否可能组合依赖对应 channel 的 `READY`。
- `VALID=1 && READY=0` 时 payload 是否保持稳定。
- reset 后 `VALID` 是否处于合法状态。
- 连续 transfer 是否无丢失。
- backpressure 随机插入时是否仍能完成 transaction。
- burst 中 `LAST` 是否与 beat 计数一致。

## 常见 bug

- `VALID` 等待 `READY`，导致死锁风险。
- `VALID=1 && READY=0` 时 payload 被更新，导致接收方拿到错误地址或数据。
- `READY` 组合路径过长，导致 timing violation。
- register slice 插入后没有保持 `LAST`、`ID`、`RESP` 与数据同步。
- 只在 `VALID` 为 1 时采样 payload，忘记要求 `VALID && READY`。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `zipcpu-wb2axip-readme`

# 03. AXI Channels

## 目的

AXI4 memory-mapped interface 由五个独立 channel 组成。本篇整理每个 channel 的职责、典型信号、握手方向和 SoC 实践注意点。

## Channel 总览

| Channel | 名称 | 方向 | Payload 类型 |
| --- | --- | --- | --- |
| AW | Write address | master -> subordinate | 写地址和写属性 |
| W | Write data | master -> subordinate | 写数据、byte strobe、last |
| B | Write response | subordinate -> master | 写响应和 ID |
| AR | Read address | master -> subordinate | 读地址和读属性 |
| R | Read data | subordinate -> master | 读数据、读响应、last、ID |

每个 channel 都有独立 `VALID` 和 `READY`。`VALID` 由发送方驱动，`READY` 由接收方驱动。

## AW: Write Address Channel

AW channel 描述一次写事务的地址和属性。

常见信号：

| 信号 | 含义 |
| --- | --- |
| `AWID` | 写事务 ID |
| `AWADDR` | 写起始地址 |
| `AWLEN` | burst beat 数减 1 |
| `AWSIZE` | 每 beat 字节数编码 |
| `AWBURST` | burst 类型 |
| `AWLOCK` | lock/exclusive 相关属性 |
| `AWCACHE` | cache/buffer 属性 |
| `AWPROT` | protection/security 属性 |
| `AWQOS` | QoS hint |
| `AWVALID` | master 表示 AW payload 有效 |
| `AWREADY` | subordinate 表示可接收 AW payload |

AW handshake 只说明写地址和属性被接收，不说明写数据已经接收，也不说明写操作已经完成。

## W: Write Data Channel

W channel 传输写数据。

常见信号：

| 信号 | 含义 |
| --- | --- |
| `WDATA` | 写数据 |
| `WSTRB` | byte lane 有效标记 |
| `WLAST` | burst 最后一个 beat |
| `WVALID` | master 表示 W payload 有效 |
| `WREADY` | subordinate 表示可接收 W payload |

AXI4 中 W channel 不再支持 write data interleaving，因此 W channel 通常不带 `WID`。AXI3 曾支持写数据交织，AXI4 移除了该能力，简化 subordinate 和 interconnect 实现。

## B: Write Response Channel

B channel 返回写事务响应。

常见信号：

| 信号 | 含义 |
| --- | --- |
| `BID` | 对应写事务 ID |
| `BRESP` | 写响应 |
| `BVALID` | subordinate 表示 B payload 有效 |
| `BREADY` | master 表示可接收 B payload |

一个 write transaction 通常对应一个 B response。对于 burst write，不是每个 beat 一个 B response，而是整个 write transaction 一个 B response。

## AR: Read Address Channel

AR channel 描述一次读事务的地址和属性。

常见信号与 AW 类似：

| 信号 | 含义 |
| --- | --- |
| `ARID` | 读事务 ID |
| `ARADDR` | 读起始地址 |
| `ARLEN` | burst beat 数减 1 |
| `ARSIZE` | 每 beat 字节数编码 |
| `ARBURST` | burst 类型 |
| `ARCACHE` | cache/buffer 属性 |
| `ARPROT` | protection/security 属性 |
| `ARQOS` | QoS hint |
| `ARVALID` | master 表示 AR payload 有效 |
| `ARREADY` | subordinate 表示可接收 AR payload |

AR handshake 只说明读请求被接收，读数据会在之后的 R channel 返回。

## R: Read Data Channel

R channel 返回读数据和读响应。

常见信号：

| 信号 | 含义 |
| --- | --- |
| `RID` | 对应读事务 ID |
| `RDATA` | 读数据 |
| `RRESP` | 读响应 |
| `RLAST` | burst 最后一个 beat |
| `RVALID` | subordinate 表示 R payload 有效 |
| `RREADY` | master 表示可接收 R payload |

读 burst 中，每个 beat 都有 `RRESP`，最后一个 beat 必须 `RLAST=1`。

## Channel 解耦示例

写事务中，AW 和 W 不要求同时发生：

```text
cycle 1: AWVALID && AWREADY，写地址被接收
cycle 2: WVALID=0，写数据尚未到达
cycle 3: WVALID && WREADY，写数据 beat 0 被接收
cycle 4: WVALID && WREADY && WLAST，最后一个写数据 beat 被接收
cycle 6: BVALID && BREADY，写响应被接收
```

这要求 subordinate 或 interconnect 设计能够暂存地址、数据或二者的匹配关系。

## SoC 实践注意点

- Interconnect 通常分别处理 read path 和 write path。
- Write path 需要处理 AW/W 汇合，因为写地址和写数据是分离 channel。
- Read path 需要保持 `RID` 和 `RLAST` 正确，尤其在多个 outstanding read 存在时。
- Crossbar 需要 address decode、admission control、decode error handling 和 ordering protection。
- Register slice、FIFO、CDC、width converter 会分别作用于各 channel，但不能破坏 transaction 语义。

## 常见 bug

- AW 已接收但 W 永远不来，导致内部资源泄漏或死锁。
- W beat 数和 `AWLEN` 不匹配，`WLAST` 过早或过晚。
- R beat 数和 `ARLEN` 不匹配，`RLAST` 错误。
- `BID` 或 `RID` 返回错误，导致 master 无法匹配事务。
- interconnect 丢失 decode error response。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `xilinx-pg059-axi-interconnect`

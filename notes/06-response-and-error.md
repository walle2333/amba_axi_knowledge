# 06. Response and Error

## 目的

AXI response 用于表示一次访问的完成状态。写事务通过 B channel 返回 response，读事务通过 R channel 在每个 data beat 上返回 response。理解 response 对 debug、interconnect decode、firewall、安全访问、异常处理都很关键。

## Response 类型

常见 response 编码语义：

| Response | 典型编码 | 含义 |
| --- | --- | --- |
| OKAY | `2'b00` | 正常访问成功，或普通访问被正常接受 |
| EXOKAY | `2'b01` | exclusive access 成功 |
| SLVERR | `2'b10` | subordinate 侧错误 |
| DECERR | `2'b11` | decode error，通常没有有效目标响应 |

具体编码以 Arm 官方规格为准。这里列出的是 AXI 常用语义。

## 写响应 B Channel

写事务响应通过 B channel 返回：

| 信号 | 含义 |
| --- | --- |
| `BID` | 对应写事务 ID |
| `BRESP` | 写响应 |
| `BVALID` | response 有效 |
| `BREADY` | master 可接收 response |

一个 write transaction 对应一个 B response。即使是多 beat write burst，也不是每个 W beat 返回一个 B response。

例子：4-beat write burst：

```text
AW handshake: 1 次
W handshake : 4 次，最后一次 WLAST=1
B handshake : 1 次，BRESP=OKAY 或错误响应
```

## 读响应 R Channel

读事务响应随 R channel 每个 beat 返回：

| 信号 | 含义 |
| --- | --- |
| `RID` | 对应读事务 ID |
| `RDATA` | 读数据 |
| `RRESP` | 当前 beat 的读响应 |
| `RLAST` | 最后一个 beat |
| `RVALID` | R payload 有效 |
| `RREADY` | master 可接收 R payload |

读 burst 中，每个 beat 都有 `RRESP`。如果某个 beat 出错，系统需要定义该 beat 的 `RDATA` 是否有意义；一般软件或上层逻辑不应依赖错误响应下的数据值。

## OKAY

`OKAY` 表示普通访问成功。对大多数正常读写，subordinate 返回 `OKAY`。

例子：CPU 读取一个合法外设寄存器：

```text
ARADDR = 0x4000_0010
RRESP  = OKAY
RDATA  = register value
```

## EXOKAY

`EXOKAY` 与 exclusive access 相关。exclusive access 通常用于实现锁、原子更新或同步原语。普通访问不会返回 `EXOKAY`。

需要注意：exclusive access 的完整语义依赖官方规格和系统是否实现 exclusive monitor。简单 AXI subordinate 可能不支持 exclusive access。

## SLVERR

`SLVERR` 表示目标 subordinate 已被选中，但访问在 subordinate 内部失败或不被允许。

可能场景：

- 访问只读寄存器并执行写操作。
- 访问 reserved register。
- subordinate 内部检测到 ECC/parity 错误。
- firewall/security block 拒绝访问，但选择以 slave error 返回。
- 设备当前状态不允许该访问。

例子：写只读寄存器：

```text
AWADDR = STATUS_REG
WDATA  = 0x1
BRESP  = SLVERR
```

## DECERR

`DECERR` 表示地址 decode 失败，通常没有任何合法 subordinate 对该地址负责。DECERR 常由 interconnect 或 default error slave 生成。

可能场景：

- 地址不在 memory map 中。
- interconnect address decode 没有命中任何 slave。
- 安全隔离后该 master 无可见目标。

例子：访问未映射地址：

```text
ARADDR = 0xDEAD_0000
RRESP  = DECERR
```

工程上，很多 interconnect 会实现 default error slave，保证非法地址访问不会让总线永久挂起，而是返回 `DECERR`。

## Error Response 与 Burst

写 burst 的错误通常通过唯一的 B response 返回。读 burst 每个 beat 都有 `RRESP`，因此不同 beat 理论上可以返回不同 response。

实际设计中需要注意：

- 即使发生错误，也要完整结束协议层面的 transaction。
- 读 burst 需要返回预期数量的 R beat，并正确给出 `RLAST`，除非官方规格定义了特定异常行为。
- 写 burst 不能因为内部错误就丢失 B response。

## Interconnect 中的错误处理

Interconnect 至少需要处理两类错误：

| 错误 | 典型响应 |
| --- | --- |
| 地址 decode 不命中 | DECERR |
| 目标 subordinate 返回错误 | 透传 SLVERR/DECERR |

alexforencich 的 AXI crossbar 文档提到 per-port address decode、admission control 和 decode error handling，这是实际 crossbar 必须处理的重要功能。

## 软件可见性

AXI response 通常会进一步映射到 CPU exception、bus fault、SError、device error interrupt 或 DMA status。

例子：DMA 读源地址时收到 `DECERR`：

```text
DMA status.error = 1
DMA status.err_type = decode_error
DMA 停止当前 descriptor
软件中断处理并检查 descriptor 地址
```

## Verification Checklist

- 每个 accepted write transaction 最终是否都有 B response。
- B response 的 `BID` 是否匹配对应 AW ID。
- 每个 accepted read transaction 是否返回 `ARLEN + 1` 个 R beat。
- 最后一个 R beat 是否 `RLAST=1`。
- Decode miss 是否返回 DECERR，而不是挂死。
- Subordinate 内部错误是否能返回 SLVERR。
- Backpressure 下 response 是否保持稳定。

## 常见 bug

- 非法地址没有 default error slave，导致访问永远不返回。
- 写数据已接收但 B response 丢失。
- 读错误提前终止，`RLAST` 没有返回。
- `BID`/`RID` 错误，master 无法匹配 response。
- 把 `SLVERR` 和 `DECERR` 混用，导致软件无法定位错误来源。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `zipcpu-wb2axip-readme`
- `xilinx-pg059-axi-interconnect`

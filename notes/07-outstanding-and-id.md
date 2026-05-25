# 07. Outstanding and ID

## 目的

Outstanding transaction 和 transaction ID 是 AXI 支持高并发和乱序返回能力的基础。它们直接影响 interconnect 设计、DDR 性能、DMA 吞吐、master/subordinate buffer 深度和 verification complexity。

## Outstanding Transaction

Outstanding transaction 指已经被接收但尚未完成响应的事务。

写事务：

```text
AW handshake 已发生，但对应 B response 尚未完成
=> write transaction outstanding
```

读事务：

```text
AR handshake 已发生，但对应最后一个 R beat 尚未完成
=> read transaction outstanding
```

Outstanding 能力越强，系统越能隐藏内存延迟，但需要更多 buffer、计数器、ID tracking 和 verification。

## 为什么需要 Outstanding

如果 master 每次只能发起一个事务，并等待响应后再发起下一个，那么 DDR 或远端 slave 的 latency 会严重限制吞吐。

例子：DDR read latency 为 80 cycles，每次 read burst 16 beats。

```text
无 outstanding: 发起 read -> 等 80 cycles -> 收 16 beats -> 再发下一个
有 outstanding: 连续发多个 read -> DDR pipeline 填满 -> R channel 持续返回数据
```

Outstanding transaction 是高带宽 DMA、CPU cache miss handling、GPU/NPU memory access 的关键。

## AXI ID 信号

AXI 用 ID 区分并发事务。

| 信号 | 通道 | 含义 |
| --- | --- | --- |
| `AWID` | AW | write transaction ID |
| `BID` | B | write response ID |
| `ARID` | AR | read transaction ID |
| `RID` | R | read response ID |

AXI4 的 W channel 通常没有 `WID`，因为 AXI4 不支持 write data interleaving。

## ID 的作用

ID 至少承担三个作用：

- 帮助 master 匹配 response 和原始 request。
- 允许同一个 master 同时发出多个事务。
- 给 interconnect 和 subordinate 提供 ordering 判断依据。

例子：master 同时发两个读事务：

```text
ARID=3, ARADDR=0x8000_0000, ARLEN=15
ARID=4, ARADDR=0x9000_0000, ARLEN=15
```

如果 subordinate/interconnect 支持不同 ID 乱序返回，`RID=4` 的数据可以先于 `RID=3` 返回。master 需要用 `RID` 区分数据属于哪一个事务。

## 同 ID 与不同 ID

同 ID 通常表示这些 transaction 之间存在更强的 ordering 约束。不同 ID 通常允许更自由的调度和返回。

直观理解：

```text
同 ID: 我希望这些事务被看作同一个顺序流
不同 ID: 我允许系统更灵活地并发处理它们
```

具体 ordering 规则以 Arm 官方规格为准，本知识库在 `08-ordering-model.md` 单独整理。

## Outstanding Depth

Outstanding depth 是某个接口、master、slave 或 interconnect 端口允许同时存在的未完成事务数量。

常见配置：

```text
CPU master read outstanding: 8/16/32+
DMA read outstanding: 4/8/16
AXI4-Lite peripheral: 通常 1 或很小
simple SRAM slave: 可能 1 或少量
DDR controller: 通常较大
```

Outstanding depth 受这些因素限制：

- ID width。
- master reorder buffer 大小。
- interconnect tracking table 大小。
- subordinate queue depth。
- response buffer 大小。
- QoS 和 deadlock 避免策略。

## ID Width

ID width 决定 ID 编码空间，但不等于 outstanding depth。

例子：

```text
ID_WIDTH = 4
=> 可编码 16 个 ID 值
```

但实际 outstanding depth 可能只有 4，因为硬件 tracking table 只有 4 项。反过来，一个 ID 下也可能有多个 outstanding transaction，但会受到 ordering 规则限制。

## Interconnect 中的 ID 扩展

多个 master 连接到一个 shared subordinate 时，interconnect 必须保证 response 能路由回正确 master。常见做法是在下游 ID 中拼接 master port number。

例子：两个 master 都使用 `ARID=3`：

```text
M0: ARID=3 -> interconnect -> downstream ID={M0, 3}
M1: ARID=3 -> interconnect -> downstream ID={M1, 3}
```

返回时：

```text
RID={M0, 3} -> route to M0, strip prefix -> RID=3
RID={M1, 3} -> route to M1, strip prefix -> RID=3
```

这就是 ID extension 或 ID prepend 的典型思路。

## ID Remapping 和 ID Serialization

有些 subordinate 支持的 ID width 比 master/interconnect 小。这时需要 ID remapping 或 serialization。

PULP AXI 文档中列出了 `axi_id_remap`、`axi_id_serialize`、`axi_iw_converter` 等模块，说明真实 SoC 中 ID width conversion 是常见工程问题。

典型问题：

```text
上游 ID width = 8
下游 ID width = 4
```

可选策略：

- Remap: 用一个表把上游 outstanding ID 映射到较小的下游 ID 空间。
- Serialize: 当无法安全映射时，减少并发，按顺序发往下游。
- Reject/configure: 系统集成时限制上游 ID 使用范围。

## Read Outstanding

Read outstanding 的完成标志是最后一个 R beat 握手完成，也就是 `RVALID && RREADY && RLAST`。

读 tracking table 通常需要记录：

```text
ARID
目标 slave 或返回路由
剩余 beat 数
ordering 信息
QoS 或仲裁信息
```

## Write Outstanding

Write outstanding 的完成标志是 B response 握手完成。

写事务更复杂，因为 AW 和 W channel 解耦。interconnect 或 subordinate 可能需要分别跟踪：

```text
AW 是否已接收
W beat 是否完整接收
B response 是否已返回
```

AXI4 不支持 write data interleaving，降低了 W channel 按 ID 交织的复杂度，但 AW/W 汇合仍是实际设计重点。

## Performance Tradeoff

提高 outstanding depth 的收益：

- 隐藏 DDR/NoC latency。
- 提高总线利用率。
- 允许 interconnect 更好地调度不同 slave。
- 提高 DMA/GPU/NPU 大块搬运吞吐。

成本：

- 增加 buffer 和 tracking table 面积。
- 增加 verification 状态空间。
- 增加 deadlock 和 starvation 分析难度。
- 增加 response reorder 处理复杂度。

## Verification Checklist

- response ID 是否总能匹配一个 outstanding request。
- outstanding counter 是否在 request accepted 时加，在 response 完成时减。
- overflow 时是否正确 backpressure，而不是丢事务。
- reset 或 flush 时 outstanding transaction 如何处理。
- ID remap 表是否不会重复分配同一个 active downstream ID。
- read burst 的 `RLAST` 是否正确释放 tracking entry。
- write response 的 `BID` 是否正确释放 tracking entry。

## 常见 bug

- ID width 配置不一致，导致 response 路由错误。
- interconnect 忘记给下游 ID 拼接 master port，多个 master ID 冲突。
- outstanding counter 在错误时机释放，导致后续 response 匹配错。
- `RLAST` 丢失，读 outstanding entry 永远不释放。
- 下游 ID 空间不足但没有 backpressure 或 serialization。
- AXI4 设计错误地假设 W channel 有 `WID`。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `alexforencich-verilog-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `xilinx-pg059-axi-interconnect`

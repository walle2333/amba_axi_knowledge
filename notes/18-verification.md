# 18. Verification

## 目的

AXI 验证不能只看“能读写一次”。真实 SoC 中需要覆盖协议合法性、burst corner cases、ID/outstanding/ordering、backpressure、error response、CDC/reset、security、performance 和系统级数据一致性。本篇整理 AXI 验证方法和检查清单。

## 验证层次

| 层次 | 目标 |
| --- | --- |
| protocol assertion | 检查 AXI 基本规则是否违反 |
| interface VIP/BFM | 生成合法/非法事务，模拟 master/slave |
| monitor | 被动采集 transaction |
| scoreboard | 比较期望数据和实际数据 |
| coverage | 衡量场景覆盖度 |
| system test | 验证 DMA、CPU、DDR、interconnect 组合行为 |
| formal verification | 对关键协议属性做穷尽或半穷尽证明 |

## Protocol Assertion

Protocol assertion 用于捕捉局部协议违规。

典型断言：

```text
VALID=1 && READY=0 时 payload 必须稳定
WLAST 必须出现在 AWLEN 对应的最后一个 beat
RLAST 必须出现在 ARLEN 对应的最后一个 beat
BID 必须匹配某个 outstanding AWID
RID 必须匹配某个 outstanding ARID
burst 不能跨越 4KB boundary
```

Assertion 应尽量贴近接口放置，便于快速定位是 upstream、downstream 还是 converter/interconnect 引入的问题。

## Master/Slave BFM

BFM 或 VIP 用于主动生成 AXI transaction。

Master BFM 应支持：

- single read/write。
- burst read/write。
- narrow/unaligned transfer。
- 多 outstanding。
- 不同 ID。
- random sideband。
- random backpressure on response channels。

Slave BFM 应支持：

- random wait states。
- random response latency。
- error response injection。
- out-of-order response for different ID。
- limited outstanding capacity。

cocotbext-axi 提供 AXI/AXI-Lite/APB master/slave/RAM、AXI stream source/sink/monitor 等模型，可作为 Python/cocotb testbench 的参考。

## Monitor

Monitor 被动监听 AXI channel，重构 transaction。

Monitor 需要记录：

```text
AW/AR request
W beat sequence
B response
R beat sequence
ID
address
burst length/size/type
response
timestamp/latency
```

对于 interconnect，monitor 应放在 upstream 和 downstream 两侧，用于确认转换、路由和 ordering 是否正确。

## Scoreboard

Scoreboard 用于比较期望行为和实际行为。

常见模型：

- Memory model: 写入更新期望内存，读取比较数据。
- Register model: 根据寄存器定义检查读写副作用。
- Routing scoreboard: 检查 response 回到正确 master。
- Ordering scoreboard: 检查同 ID 和系统要求的 ordering。
- Error scoreboard: 检查非法地址和权限错误响应。

PULP AXI 文档中提到 `axi_scoreboard`、`axi_sim_mem`、`axi_chan_logger` 等验证模块，说明高质量 AXI IP 通常会配套 transaction-level 检查工具。

## Coverage

AXI coverage 至少应覆盖：

```text
burst length: 1, 2, 4, 16, 256
burst type: FIXED, INCR, WRAP
transfer size: full-width, narrow
address alignment: aligned, unaligned
boundary: near 4KB boundary, crossing attempt
response: OKAY, SLVERR, DECERR, EXOKAY if supported
ID: same ID, different ID, ID reuse
outstanding: 0, 1, max-1, max
backpressure: each channel independently stalled
reset: idle reset, active transaction reset
```

## Random Backpressure

AXI bug 很多只在 backpressure 下暴露。验证时应随机拉低每个 channel 的 `READY` 或 `VALID`。

重点场景：

- AW accepted, W delayed。
- W arrives before AW。
- BVALID held while BREADY=0。
- RVALID held while RREADY=0。
- Stream `TVALID` held while `TREADY=0`。
- CDC FIFO near full/empty。

## Error Injection

需要主动注入错误：

- Decode miss。
- Slave error。
- Unsupported burst。
- Security violation。
- SMMU/firewall fault。
- Stream early/late `TLAST`。
- Reset during outstanding transaction。

验证目标不是“没有错误”，而是“错误发生时协议仍能完成，并且软件/上层能诊断”。

## Formal Verification

Formal 适合验证局部协议属性和小型桥接模块，例如 AXI4-Lite slave、skid buffer、FIFO、crossbar arbiter、firewall。

适合 formal 的属性：

- Handshake payload stability。
- 每个 request 最终有 response，带 bounded liveness 假设。
- Outstanding counter 不 underflow/overflow。
- Response ID 必须属于 active request。
- FIFO 不丢不重。
- Firewall 拒绝后仍返回协议合法 response。

ZipCPU 资料中大量强调 formal verification 和 bus firewall，对 AXI-Lite、crossbar、firewall 等模块很有参考价值。

## System-Level Tests

系统级验证应包含：

- CPU 通过 AXI4-Lite 配置 DMA。
- DMA 从 DDR 读、写 DDR。
- DMA stream path 的 `TLAST/TKEEP`。
- 多 master 同时访问 DDR。
- 非法地址访问和 error interrupt。
- Non-coherent DMA cache maintenance。
- Low-power/reset/clock gating 场景。

## Debug 资料输出

建议 testbench 保存：

- AXI transaction log。
- outstanding table snapshot。
- performance counters。
- error response log。
- waveform bookmarks。
- decoded memory map。

这些输出比单纯 VCD 波形更利于定位复杂 SoC 问题。

## 常见验证盲区

- 只测 `READY` 永远为 1。
- 只测 single-beat，不测 burst。
- 只测 aligned full-width，不测 narrow/unaligned。
- 只测一个 ID，不测 outstanding 和 reordering。
- 只测 OKAY，不测错误响应。
- 不测 reset during transaction。
- 不测 bridge/converter 的属性信号保留。
- 不测 long-run random traffic 和 starvation。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-cocotbext-axi-readme`
- `pulp-platform-axi-readme`
- `zipcpu-wb2axip-readme`
- `fpganinja-taxi-readme`

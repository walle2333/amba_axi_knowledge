# 17. QoS and Performance

## 目的

AXI 性能不仅由数据位宽决定，还受到 burst length、outstanding depth、interconnect 仲裁、DDR 行命中率、backpressure、clock frequency、width converter、CDC、QoS 和软件访问模式影响。本篇整理 SoC 中常见 AXI 性能指标、瓶颈和调优方法。

## 基本指标

| 指标 | 含义 |
| --- | --- |
| bandwidth | 单位时间传输数据量 |
| latency | 从 request 到 response 的延迟 |
| throughput | 稳态可完成事务或 beat 的速率 |
| utilization | 总线有效传输周期占比 |
| outstanding depth | 未完成事务并发数 |
| backpressure ratio | `VALID=1 && READY=0` 的比例 |

理论峰值带宽：

```text
peak_bandwidth = data_width_bytes * frequency
```

例子：

```text
DATA_WIDTH = 128 bits = 16 bytes
clock      = 500 MHz
peak       = 16 * 500M = 8 GB/s
```

实际带宽通常低于峰值，因为存在地址开销、仲裁、latency、refresh、page miss、backpressure 和软件访问碎片。

## Burst Length

较长 burst 可减少地址开销，提高数据通道利用率。

短 burst 问题：

```text
每 1 beat 都发一次地址
地址通道和仲裁开销高
DDR row/bank 调度效率低
```

长 burst 好处：

```text
地址开销低
数据通道连续传输
更适合 DDR 和 DMA
```

但 burst 过长也有代价：

- 增加其他 master 的等待时间。
- 影响 latency-sensitive traffic。
- 可能降低仲裁公平性。
- 需要更大 buffer。

## Outstanding Depth

Outstanding depth 用于隐藏 latency。

例子：DDR read latency 很高时，单 outstanding 可能无法填满 R channel。多个 outstanding read 可以让 DDR controller pipeline 工作。

性能估算直觉：

```text
需要的 outstanding bytes ≈ latency_cycles * data_width_bytes
```

如果每个 burst 传输 64 bytes，而需要覆盖 512 bytes 的 latency window，则至少需要约 8 个 outstanding burst。

## Data Width

增加数据宽度能提高峰值带宽，但不是免费：

- 增加布线面积和功耗。
- 增加 mux/crossbar 复杂度。
- 增加 timing closure 难度。
- 宽窄转换可能引入额外 latency。

高带宽 master 应尽量接近 DDR/interconnect 的自然宽度，低速外设不应使用过宽 AXI。

## AxQOS

`AxQOS` 可作为 QoS hint。是否生效取决于 interconnect、NoC 和 DDR controller 实现。

典型策略：

```text
display read: high QoS，避免 underflow
CPU interactive access: medium/high QoS，降低 latency
bulk DMA copy: low/medium QoS，追求吞吐但可让路
background memory scrub: low QoS
```

QoS 需要系统级验证。单纯拉高 `AxQOS` 不保证性能，仲裁器和 DDR scheduler 必须使用该信息。

## Backpressure 分析

AXI 性能调试时，应统计每个 channel 的握手情况：

```text
valid_cycles = VALID 为 1 的周期数
stall_cycles = VALID=1 && READY=0 的周期数
transfer_cycles = VALID=1 && READY=1 的周期数
```

如果 W channel stall 很高，可能是下游写 buffer 满或 DDR 写入慢。如果 AR channel stall 很高，可能是 interconnect admission control 或 read outstanding table 满。

## DDR 相关瓶颈

DDR 性能受以下因素影响：

- row hit / row miss。
- bank conflict。
- read/write turnaround。
- refresh。
- controller queue depth。
- outstanding 和 reorder 能力。
- 访问地址连续性。

AXI 侧优化通常包括：

- 使用较长 INCR burst。
- 地址对齐。
- 提高 outstanding。
- 合并小访问。
- 避免频繁读写切换。

## Interconnect 瓶颈

Interconnect 可能成为瓶颈：

- shared interconnect 不支持并发。
- crossbar 某个 slave port 被多个 master 争用。
- response buffer 太小。
- ID remap table 满。
- QoS 配置不合理。
- width converter 降低有效吞吐。
- CDC FIFO 深度不足。

## 性能计数器

建议在关键 AXI 端口加性能计数器：

```text
AW handshake count
W beat count
B response count
AR handshake count
R beat count
stall cycles per channel
max outstanding
latency histogram
response error count
```

ZipCPU 资料中提到 AXI performance measurement peripheral，这类监控模块对定位总线瓶颈非常有价值。

## 调优流程

1. 明确目标：带宽、延迟、公平性还是功耗。
2. 测量各 channel utilization 和 stall。
3. 检查 burst length 和地址连续性。
4. 检查 outstanding 是否足够。
5. 检查 interconnect 仲裁和 QoS。
6. 检查 DDR controller queue 和 row hit。
7. 检查 width converter、CDC、clock frequency。
8. 修改配置后用真实 workload 复测。

## 常见 bug

- 只看理论位宽，忽略 `READY` stall。
- Burst 太短，DDR 效率低。
- Outstanding depth 太小，无法隐藏 latency。
- QoS 信号设置了但 interconnect 没用。
- Width converter 让高带宽 path 被窄口限制。
- DMA descriptor 太碎，导致大量小 burst。
- Performance counter 只统计 data beat，不统计等待时间。

## Verification Checklist

- Random traffic 下是否满足最低带宽。
- High-priority traffic 是否能在压力下满足 latency。
- Low-priority traffic 是否不会 starvation。
- Outstanding table full 时是否正确 backpressure。
- QoS 配置变化是否影响仲裁结果。
- 读写混合场景是否符合 DDR 性能预期。
- Performance counter 是否与波形一致。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `pulp-platform-axi-microarchitecture-paper`
- `alexforencich-cocotbext-axi-readme`
- `zipcpu-wb2axip-readme`
- `fpganinja-taxi-readme`

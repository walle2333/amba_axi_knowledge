# 20. SoC Practice

## 目的

本篇把前面章节串成 SoC 实践视角：如何在一个芯片系统中规划 AXI fabric、master/slave、地址映射、DMA、DDR、外设、coherency、security、QoS、CDC 和验证策略。

## 典型 AXI SoC 拓扑

```text
CPU cluster ----\
DMA -------------> main AXI interconnect / NoC -> DDR controller
GPU/NPU/ISP -----/

CPU cluster -----> AXI4-Lite config fabric -> AXI-to-APB bridge -> peripherals

DMA MM2S/S2MM <-> AXI4 memory-mapped + AXI4-Stream pipelines
```

高带宽数据面和低速控制面应分层设计，避免所有模块都堆在同一条 AXI fabric 上。

## Master 清单

SoC 规划时应列出所有 AXI master：

| Master | 访问类型 | 关注点 |
| --- | --- | --- |
| CPU | instruction/data/register | latency、coherency、security |
| DMA | bulk memory copy/stream | bandwidth、burst、descriptor、error |
| GPU/NPU | high bandwidth memory | outstanding、QoS、DDR efficiency |
| ISP/display | real-time stream/memory | bandwidth guarantee、underflow |
| debug | memory/register access | security、lifecycle |
| peripheral bus bridge | register access | simplicity、error response |

## Subordinate 清单

典型 subordinate：

```text
DDR controller
SRAM/scratchpad
Boot ROM
AXI4-Lite register blocks
AXI-to-APB bridge
SMMU/IOMMU
firewall/TZC
debug/config blocks
```

每个 subordinate 应明确：

- 地址范围。
- 支持的数据宽度。
- 支持的 burst 类型和长度。
- 支持的 outstanding depth。
- 是否支持 exclusive/atomic/coherent access。
- 错误响应策略。

## Memory Map

Memory map 是 AXI SoC 的基础合同。

设计原则：

- 地址区域对齐，避免复杂 decode。
- 高速内存和低速外设分区明确。
- Secure/non-secure 区域明确。
- 预留区域访问必须返回错误或有明确行为。
- Burst 不应跨 region。

示例：

```text
0x0000_0000 - 0x000F_FFFF  Boot ROM
0x1000_0000 - 0x100F_FFFF  SRAM
0x4000_0000 - 0x4FFF_FFFF  APB peripherals
0x8000_0000 - 0xFFFF_FFFF  DDR
```

## CPU 访问外设寄存器

路径：

```text
CPU -> AXI4-Lite fabric -> AXI-to-APB bridge -> peripheral
```

关注点：

- AXI4-Lite slave 正确处理 AW/W 解耦。
- `WSTRB` 正确支持 byte/halfword write。
- APB wait state 正确映射到 AXI response latency。
- 未映射寄存器返回错误。
- Register side effect 文档清楚。

## DMA 写 DDR

路径：

```text
DMA descriptor fetch -> AXI read
DMA data movement    -> AXI read/write or AXI4-Stream
DMA status update    -> AXI write or interrupt
```

关注点：

- Descriptor 地址和 buffer 地址合法。
- Burst length 和 alignment 合理。
- `WSTRB/TKEEP` 正确处理最后一个 beat。
- Error response 能停止 descriptor 并上报软件。
- Non-coherent DMA 需要 cache maintenance。
- Outstanding depth 足够隐藏 DDR latency。

## 多 Master 访问 DDR

典型 master：CPU、DMA、NPU、GPU、display。

关注点：

- DDR 带宽预算。
- QoS 和仲裁。
- Latency-sensitive 与 bandwidth-heavy traffic 分离。
- Outstanding 和 ID width 配置。
- Performance counter 放置位置。
- Starvation 和 deadline miss。

## Interconnect 分层

建议按流量类型分层：

```text
main high-performance AXI/NoC fabric
peripheral AXI4-Lite fabric
debug/config fabric
low-power always-on fabric
```

好处：

- 降低高性能 fabric 负载。
- 简化外设验证。
- 降低 timing 和功耗压力。
- 安全域和电源域更清楚。

## Security 和 Coherency

SoC 级设计必须同时考虑：

- Master 是否 secure/non-secure。
- Master 是否 coherent。
- DMA 是否经过 SMMU/IOMMU。
- Buffer memory attribute 是否正确。
- Firewall 是否覆盖所有 master。
- Debug 访问是否受 lifecycle 限制。

一个高频错误是只验证地址 decode，不验证 master identity、security attribute 和 cache coherency。

## Clock/Power Domain

AXI fabric 常跨多个 clock/power domain。

设计原则：

- 每个跨域点使用经过验证的 clock converter。
- Power down 前 drain outstanding。
- Isolation 和 reset 顺序明确。
- Always-on debug/path 与 main fabric 的边界清楚。
- Low-power entry/exit 有系统级测试。

## SoC Integration Checklist

- Master/slave 清单完整。
- Memory map 有唯一 decode。
- 每个端口 ID width、data width、outstanding depth 已确认。
- Burst capability 与 converter 策略已确认。
- CDC 和 reset domain 已标注。
- Security/firewall/SMMU 策略已确认。
- Coherency 和 cache maintenance 策略已确认。
- QoS 和性能预算已确认。
- Error response 和 interrupt/reporting 已确认。
- Verification plan 覆盖 normal/error/stress/reset/security/performance。

## Bring-up Debug Checklist

1. 先用 AXI4-Lite 读写已知 ID/version register。
2. 测未映射地址是否返回错误而非挂死。
3. 测 SRAM 简单读写。
4. 测 DDR aligned burst。
5. 测 DMA 小 buffer，再测大 buffer。
6. 加入 random backpressure 和多 master 并发。
7. 打开 performance counter 定位瓶颈。
8. 测 reset、clock gating、power mode。
9. 测 secure/non-secure 访问隔离。
10. 测 non-coherent DMA cache maintenance。

## 文档交付建议

SoC AXI 集成文档至少应包含：

- Top-level AXI fabric diagram。
- Master/slave port table。
- Address map。
- ID width and remapping policy。
- Outstanding depth table。
- Clock/reset domain diagram。
- Security/coherency attribute table。
- QoS/performance budget。
- Error response policy。
- Verification coverage matrix。

## 参考资料

- `arm-amba-architecture-page`
- `arm-amba-axi-ace-protocol-spec`
- `arm-amba-chi-protocol-spec`
- `xilinx-pg059-axi-interconnect`
- `xilinx-pg021-axi-dma`
- `pulp-platform-axi-readme`
- `pulp-platform-axi-microarchitecture-paper`
- `alexforencich-cocotbext-axi-readme`
- `fpganinja-taxi-readme`

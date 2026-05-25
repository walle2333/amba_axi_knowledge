# 13. Bridges and Converters

## 目的

SoC 中很少所有模块都使用完全相同的 AXI 配置。Bridges and converters 用于连接不同协议、不同数据宽度、不同 ID 宽度、不同 burst 能力和不同系统边界。本篇整理常见转换类型、设计难点和验证重点。

## 常见类型

| 类型 | 作用 |
| --- | --- |
| AXI-to-AXI-Lite | full AXI 到控制寄存器接口 |
| AXI4-Lite-to-APB | 连接低速外设总线 |
| data width converter | 数据宽度转换，如 128-bit 到 32-bit |
| ID width converter | ID 宽度转换、ID remap、serialization |
| clock converter | 跨时钟域传输 AXI transaction |
| protocol converter | AXI3/AXI4/AXI4-Lite/APB/自定义 memory interface 转换 |
| burst splitter | 把 burst 拆成更小 transaction 或 single-beat |

PULP AXI 和 alexforencich verilog-axi 都包含 data width converter、ID converter、AXI-to-AXI-Lite、AXI-Lite-to-APB、CDC 等模块，说明转换器是 AXI SoC 实践中的常规组件。

## AXI4 to AXI4-Lite

AXI4 full 到 AXI4-Lite 的转换常用于将高性能 interconnect 接到简单寄存器块。

主要问题：

- AXI4 支持 burst，AXI4-Lite 不支持 burst。
- AXI4 可能有多个 outstanding，AXI4-Lite 目标通常只支持很小并发。
- AXI4 有 ID，AXI4-Lite 通常无 ID 或简化。

转换策略：

- 拒绝或拆分 burst。
- 将 AXI4 burst 分解成多个 AXI4-Lite single-beat access。
- 限制 outstanding，必要时 serialization。
- 正确聚合或返回 response。

## AXI4-Lite to APB

AXI4-Lite-to-APB bridge 是最常见桥之一。

```text
AXI4-Lite write:
AW + W -> APB setup/access -> B

AXI4-Lite read:
AR -> APB setup/access -> R
```

设计要点：

- APB 非 channelized，桥内需要将 AXI 的独立通道重新排序成 APB 事务。
- APB wait state 需要转化为 AXI backpressure 或延迟 response。
- `PSLVERR` 需要映射到 AXI `SLVERR`。
- APB 通常不支持 pipelined outstanding，桥需要限制 AXI 侧并发。

## Data Width Converter

Data width converter 连接不同 `DATA_WIDTH` 的 AXI 接口。

### Downsizer

宽 master 到窄 subordinate：

```text
128-bit AXI master -> 32-bit AXI subordinate
```

一个宽 beat 可能拆成多个窄 beat。需要处理：

- 地址递增。
- `WSTRB` 拆分。
- `WLAST/RLAST` 重新生成。
- response 聚合。
- unaligned/narrow transfer。

### Upsizer

窄 master 到宽 subordinate：

```text
32-bit AXI master -> 128-bit AXI subordinate
```

多个窄 beat 可合并为一个宽 beat。需要处理：

- 合并 buffer。
- byte lane 对齐。
- partial write 的 `WSTRB`。
- burst 边界和 4KB boundary。
- response 拆分或映射回上游。

## ID Width Converter

ID width converter 连接不同 `ID_WIDTH` 的接口。

上游 ID 宽，下游 ID 窄时，需要：

- ID remap table。
- active downstream ID 分配。
- 无可用 ID 时 backpressure。
- 保证 response 能恢复到原始 upstream ID。
- 必要时 serialization。

PULP AXI 中的 `axi_id_remap`、`axi_id_serialize`、`axi_iw_converter` 是这类问题的典型工程模块。

## Burst Splitter

Burst splitter 将长 burst 拆成短 burst 或 single-beat transaction。

用途：

- 下游不支持长 burst。
- 下游不支持 WRAP burst。
- 避免跨边界。
- 连接 AXI4 full 到 AXI4-Lite/APB。

注意点：

- 拆分后仍要保持上游可见的 ordering。
- 写 response 可能需要聚合。
- 读 response 需要连续返回并正确 `RLAST`。
- 错误响应如何聚合必须有明确策略。

## AXI to Memory Interface

很多 SRAM controller 或 scratchpad 使用更简单的 memory request/grant/valid 接口。AXI-to-memory converter 需要把 AXI 的 burst、ID、WSTRB、response 转换为内部存储器访问。

常见简化：

- 只支持 INCR burst。
- 限制 outstanding。
- 不支持 exclusive/atomic。
- 对不支持的属性返回错误或忽略。

## AXI Memory-Mapped to AXI4-Stream

DMA 是典型的 memory-mapped 与 stream 转换器：

```text
MM2S: AXI read -> stream output
S2MM: stream input -> AXI write
```

关键问题：

- stream `TLAST` 与 descriptor/buffer 长度关系。
- `TKEEP` 与 `WSTRB` 的映射。
- DDR backpressure 与 stream backpressure 的传播。
- descriptor error、AXI error、stream early/late TLAST 的处理。

## Verification Checklist

- 转换前后 beat 数是否一致。
- `WSTRB/TKEEP` byte lane 是否正确映射。
- `LAST` 是否正确重建。
- Burst 拆分是否不跨 4KB 或目标 region。
- Error response 是否正确聚合或拆分。
- ID remap 是否不会冲突。
- Backpressure 随机插入时是否无丢失、无重复。
- 下游不支持的特性是否被拒绝、过滤或安全降级。

## 常见 bug

- Width converter 在 unaligned transfer 下 byte lane 错误。
- Burst splitter 生成错误 `RLAST/WLAST`。
- AXI-to-AXI-Lite converter 允许多个 outstanding 打到单事务寄存器块。
- APB bridge 在 wait state 下丢失 AXI response。
- ID converter 复用仍 active 的 downstream ID。
- DMA stream `TLAST` 与内存写入长度不一致。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `alexforencich-verilog-axi-readme`
- `xilinx-pg021-axi-dma`
- `zipcpu-wb2axip-readme`
- `fpganinja-taxi-readme`

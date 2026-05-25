# 09. AXI4-Lite

## 目的

AXI4-Lite 是 AXI4 的轻量级 memory-mapped 子集，主要用于低带宽、低复杂度的控制寄存器访问。它保留 AXI 的五通道和 `VALID`/`READY` 握手机制，但去掉 burst、多 ID 并发等高性能特性。

## 定位

AXI4-Lite 适合控制面，不适合数据面。

典型用途：

```text
CPU 配置 DMA 控制寄存器
CPU 读取中断状态寄存器
debug master 访问外设寄存器
软件配置 accelerator 参数
```

不适合用途：

```text
大块 DDR 搬运
视频帧传输
高吞吐 DMA 数据通路
cache line refill
```

## 与 AXI4 Full 的关键差异

| 项目 | AXI4 Full | AXI4-Lite |
| --- | --- | --- |
| Burst | 支持 | 不支持 |
| 每个 transaction beat 数 | 1 到 256 | 固定 1 |
| ID | 支持复杂 outstanding/ordering | 通常无 ID 或固定/简化 |
| 数据通路用途 | 高吞吐 memory access | 控制寄存器访问 |
| 实现复杂度 | 高 | 低 |
| 常见目标 | DDR、SRAM、DMA buffer | control/status register |

## 通道

AXI4-Lite 仍保留五个 channel：

| Channel | 作用 |
| --- | --- |
| AW | 写地址 |
| W | 写数据 |
| B | 写响应 |
| AR | 读地址 |
| R | 读数据和读响应 |

与 AXI4 full 相比，AXI4-Lite 没有 burst 相关复杂性，因此通常没有 `AxLEN`、`AxSIZE`、`AxBURST` 这类完整 burst 控制，或者在某些实现中固定化。

## 写事务流程

```text
1. Master 在 AW channel 发送写地址。
2. Master 在 W channel 发送写数据和 WSTRB。
3. Subordinate 完成寄存器写。
4. Subordinate 在 B channel 返回 BRESP。
```

注意：AW 和 W 仍然是独立 channel，不要求同周期握手。AXI4-Lite slave 不能假设地址和数据总是同时到达。

## 读事务流程

```text
1. Master 在 AR channel 发送读地址。
2. Subordinate 查询寄存器或内部状态。
3. Subordinate 在 R channel 返回 RDATA 和 RRESP。
```

AXI4-Lite 读事务只有一个 data beat，因此不需要处理多 beat 的 `RLAST` 计数问题。

## WSTRB 与寄存器写

AXI4-Lite 写寄存器时必须正确处理 `WSTRB`。

32-bit register 示例：

```text
old value = 0x1122_3344
WDATA     = 0xAABB_CCDD
WSTRB     = 4'b0011
new value = 0x1122_CCDD
```

如果 slave 忽略 `WSTRB`，软件执行 byte write 或 halfword write 时会破坏其他 byte。

## 寄存器类型

常见寄存器行为：

| 类型 | 行为 |
| --- | --- |
| RW | 软件可读写 |
| RO | 只读，写入忽略或返回错误 |
| WO | 只写，读返回 0 或未定义值 |
| W1C | write-one-to-clear，写 1 清除对应 bit |
| W1S | write-one-to-set，写 1 置位对应 bit |
| RC | read-clear，读取后清除 |

这些行为不是 AXI4-Lite 协议本身定义的，而是外设寄存器设计约定。设计文档必须明确说明。

## 地址 Decode

AXI4-Lite slave 通常根据地址低位选择寄存器：

```text
base address = 0x4000_0000
offset 0x00  = CTRL
offset 0x04  = STATUS
offset 0x08  = IRQ_ENABLE
offset 0x0C  = IRQ_STATUS
```

未映射 offset 的常见处理：

- 返回 `SLVERR`。
- 返回 `DECERR`，如果错误由上游 interconnect/default slave 生成。
- 返回 0 并 `OKAY`，但这会降低 debug 可见性，不推荐作为默认策略。

## AXI4-Lite to APB Bridge

AXI4-Lite 经常通过 bridge 转到 APB：

```text
CPU -> AXI4-Lite -> AXI-to-APB bridge -> APB peripherals
```

Bridge 需要完成：

- AXI AW/W/B 到 APB write phase 的转换。
- AXI AR/R 到 APB read phase 的转换。
- APB `PSLVERR` 到 AXI `SLVERR` 的映射。
- AXI backpressure 与 APB wait state 的适配。

## RTL 设计注意点

最小 AXI4-Lite slave 也需要处理这些问题：

- AW 和 W 可能不同周期到达。
- 写响应必须在地址和数据都被接收后返回。
- 读数据在 `RVALID=1 && RREADY=0` 时必须保持稳定。
- 写响应在 `BVALID=1 && BREADY=0` 时必须保持稳定。
- reset 后不能残留有效 response。
- `WSTRB` 必须参与寄存器 byte enable。

## Verification Checklist

- AW 先到、W 后到是否正确。
- W 先到、AW 后到是否正确。
- B channel backpressure 下 `BRESP` 是否稳定。
- R channel backpressure 下 `RDATA/RRESP` 是否稳定。
- `WSTRB` partial write 是否正确。
- 未映射地址是否返回预期错误。
- 连续读写是否不会丢 response。

## 常见 bug

- 假设 AW 和 W 同周期握手。
- 忽略 `WSTRB`，导致 byte write 错误。
- 写响应过早返回，此时 W 数据尚未接收。
- `RDATA` 在 `RVALID=1 && RREADY=0` 时变化。
- 未映射地址无响应，导致 CPU bus fault 卡死或超时。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `pulp-platform-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `fpganinja-taxi-readme`

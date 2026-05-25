# DMA

用于保存 DMA 与 AXI memory-mapped/AXI4-Stream 的系统示例。

## Planned Examples

- AXI master 写 DDR。
- AXI master 读 DDR。
- Memory-mapped to stream。
- Stream to memory-mapped。
- Descriptor fetch and data movement。

## Memory-to-Stream DMA

```text
1. CPU writes descriptor through AXI4-Lite.
2. DMA reads source buffer from DDR through AXI4 read.
3. DMA emits payload through AXI4-Stream.
4. Last stream beat asserts TLAST.
5. DMA updates status or raises interrupt.
```

检查点：non-coherent 系统中，CPU 启动 DMA 前需要 clean descriptor 和 source buffer。

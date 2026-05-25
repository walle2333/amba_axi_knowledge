# AXI4-Stream

用于保存 AXI4-Stream 数据流示例。

## Planned Examples

- Stream source/sink。
- TLAST packet boundary。
- TKEEP byte qualifier。
- Backpressure handling。

## 14-Byte Packet on 64-Bit Stream

```text
beat 0: TKEEP=8'b1111_1111, TLAST=0
beat 1: TKEEP=8'b0011_1111, TLAST=1
```

`TKEEP` 标记最后一个 beat 只有 6 bytes 有效，`TLAST` 标记 packet 结束。

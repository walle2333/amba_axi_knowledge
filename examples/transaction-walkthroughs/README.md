# Transaction Walkthroughs

用于逐周期拆解 AXI transaction。

## Planned Examples

- Single write transaction。
- Single read transaction。
- 4-beat INCR burst write。
- Unaligned write with WSTRB。
- 4KB boundary violation example。

## 4-Beat INCR Write Walkthrough

```text
AWADDR=0x1000, AWSIZE=2, AWLEN=3, AWBURST=INCR
W beat 0 -> address 0x1000, WLAST=0
W beat 1 -> address 0x1004, WLAST=0
W beat 2 -> address 0x1008, WLAST=0
W beat 3 -> address 0x100C, WLAST=1
B response -> one BRESP for the whole burst
```

检查点：`WLAST` 必须只在第 4 个 beat 拉高，B response 只有一个。

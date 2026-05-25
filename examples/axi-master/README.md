# AXI Master

用于保存 AXI master 发起读写事务的示例。

## Planned Examples

- Single write master。
- Single read master。
- Burst write master。
- Outstanding read requests。

## Burst Read Master Scenario

```text
ARADDR=0x8000_0000
ARSIZE=3       // 8 bytes per beat
ARLEN=15       // 16 beats
ARBURST=INCR
```

Master 需要准备接收 16 个 R beats，并在 `RLAST=1` 的最后一个 beat 后释放 outstanding entry。

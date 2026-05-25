# AXI to APB Bridge

用于保存 AXI-to-APB bridge 的事务转换和设计注意点。

## Planned Examples

- AXI4-Lite write to APB write。
- AXI4-Lite read to APB read。
- APB wait state。
- Error response mapping。

## AXI4-Lite Write to APB

```text
AXI AW/W accepted
-> APB setup:  PSEL=1, PENABLE=0, PWRITE=1
-> APB access: PSEL=1, PENABLE=1
-> if PREADY=1, return AXI B response
```

`PSLVERR=1` 通常映射为 AXI `SLVERR`。

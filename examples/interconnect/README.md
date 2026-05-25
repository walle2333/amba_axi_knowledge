# Interconnect

用于保存 AXI interconnect、crossbar 和 NoC 场景示例。

## Planned Examples

- Address decode。
- Multi-master arbitration。
- ID remapping。
- Backpressure propagation。
- Deadlock risk analysis。

## Address Decode Example

```text
0x0000_0000 - 0x000F_FFFF -> Boot ROM
0x1000_0000 - 0x100F_FFFF -> SRAM
0x4000_0000 - 0x4FFF_FFFF -> APB bridge
0x8000_0000 - 0xFFFF_FFFF -> DDR
default                  -> DECERR
```

Interconnect 必须保证 burst 不跨 slave region，decode miss 必须返回错误而不是挂死。

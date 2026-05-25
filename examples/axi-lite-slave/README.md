# AXI4-Lite Slave

用于保存 AXI4-Lite slave 寄存器访问示例。

## Planned Examples

- Minimal register bank。
- Read/write handshake FSM。
- WSTRB byte write。
- Error response for unmapped address。

## Register Write Example

```text
Address map:
0x00 CTRL
0x04 STATUS
0x08 IRQ_ENABLE

Write CTRL:
AWADDR=base+0x00
WDATA=0x0000_0001
WSTRB=4'b1111
BRESP=OKAY
```

RTL 注意点：AW 和 W 可以不同周期到达，必须等地址和数据都接收后再返回 B response。

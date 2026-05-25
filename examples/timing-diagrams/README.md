# Timing Diagrams

用于保存 AXI 握手、burst、backpressure、outstanding transaction 等时序示例。

## Planned Examples

- VALID/READY basic handshake。
- READY low wait states。
- INCR burst read/write。
- Write address and write data channel decoupling。
- Read data interleaving and ordering examples。

## VALID/READY Wait State Example

```text
cycle:    0  1  2  3  4
VALID:   0  1  1  1  0
READY:   0  0  0  1  1
payload: -  A  A  A  -
transfer          ^
```

规则：`VALID=1 && READY=0` 时 payload `A` 必须保持稳定，直到 `cycle 3` 完成 transfer。

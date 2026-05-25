# Burst Examples

## INCR Burst

```text
start address = 0x1000
AxSIZE = 2     // 4 bytes per beat
AxLEN  = 3     // 4 beats
addresses = 0x1000, 0x1004, 0x1008, 0x100C
```

## FIXED Burst

```text
start address = 0x2000
AxSIZE = 2
AxLEN  = 3
addresses = 0x2000, 0x2000, 0x2000, 0x2000
```

## WRAP Burst

```text
start address = 0x1018
AxSIZE = 3     // 8 bytes per beat
AxLEN  = 3     // 4 beats
wrap size = 32 bytes
wrap base = 0x1000
addresses = 0x1018, 0x1000, 0x1008, 0x1010
```

## 4KB Boundary

合法：

```text
start = 0x0000_0F00
total = 128 bytes
end   = 0x0000_0F7F
```

非法：

```text
start = 0x0000_0FF0
total = 32 bytes
end   = 0x0000_100F
```

## WSTRB

```text
32-bit WDATA = 0xAABB_CCDD
WSTRB=1111 -> write AABB_CCDD
WSTRB=0011 -> write low halfword CCDD
WSTRB=1100 -> write high halfword AABB
```

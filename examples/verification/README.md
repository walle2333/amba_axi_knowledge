# Verification

用于保存 AXI 验证示例和检查点。

## Planned Examples

- VALID/READY assertion。
- Payload stability assertion。
- Burst length and RLAST check。
- WLAST check。
- Ordering scoreboard。

## Payload Stability Assertion Idea

```systemverilog
property payload_stable_when_waiting;
    @(posedge ACLK) disable iff (!ARESETn)
    AWVALID && !AWREADY |=> AWVALID && $stable(AWADDR);
endproperty
```

实际工程中还需要把 `AWLEN/AWSIZE/AWBURST/AWID/AWPROT/AWCACHE` 等 payload 一并纳入稳定性检查。

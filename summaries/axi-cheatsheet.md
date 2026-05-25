# AXI Cheatsheet

## 五通道

| Channel | 方向 | 作用 | 完成条件 |
| --- | --- | --- | --- |
| AW | master -> subordinate | write address | `AWVALID && AWREADY` |
| W | master -> subordinate | write data | `WVALID && WREADY` |
| B | subordinate -> master | write response | `BVALID && BREADY` |
| AR | master -> subordinate | read address | `ARVALID && ARREADY` |
| R | subordinate -> master | read data/response | `RVALID && RREADY` |

## 必记规则

- Transfer 发生在 `VALID && READY`。
- 发送方不能等待 `READY` 才拉高 `VALID`。
- `VALID=1 && READY=0` 时 payload 必须稳定。
- `AxLEN` 是 beat 数减 1。
- Burst 不能跨 4KB boundary。
- AXI4 W channel 不支持 write data interleaving，通常没有 `WID`。
- 写 burst 只有一个 B response。
- 读 burst 每个 R beat 都有 `RRESP`，最后一个 beat `RLAST=1`。

## 常用公式

```text
beats = AxLEN + 1
bytes_per_beat = 2 ^ AxSIZE
end_addr = start_addr + total_bytes - 1
```

## Response

| Response | 含义 |
| --- | --- |
| OKAY | 正常完成 |
| EXOKAY | exclusive access 成功 |
| SLVERR | subordinate 侧错误 |
| DECERR | 地址 decode 错误 |

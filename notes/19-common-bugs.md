# 19. Common Bugs

## 目的

本篇整理 AXI 设计、集成和验证中高频 bug，并给出 debug 线索。很多 AXI 问题不是单点错误，而是 channel 解耦、backpressure、ID、ordering、CDC、reset、bridge 转换共同作用后暴露。

## Handshake 类 bug

### VALID 依赖 READY

症状：

```text
VALID=0, READY=0，双方互相等待
```

原因：发送方错误地等待 `READY=1` 才拉高 `VALID`。

修复：发送方只要有 payload 就应能拉高 `VALID`，并在 `VALID=1 && READY=0` 时保持 payload 稳定。

### Payload 不稳定

症状：

```text
AWVALID=1, AWREADY=0 时 AWADDR 变化
```

后果：接收方最终握手时拿到错误地址或混合属性。

## Burst 类 bug

### AxLEN 理解错误

`AxLEN` 是 beat 数减 1，不是 beat 数。

错误：

```text
需要 4 beats，却设置 AxLEN=4
```

结果：对端期待 5 beats，`LAST` 不匹配。

### LAST 错误

常见形式：

- `WLAST` 早一拍。
- `WLAST` 晚一拍。
- `RLAST` 丢失。
- single-beat burst 未拉 `LAST`。

后果：transaction 卡死、后续 beat 错位、outstanding entry 不释放。

### 跨 4KB 边界

症状：某些 burst 在 interconnect 或 DDR controller 处返回错误，或者仿真 assertion 失败。

检查：

```text
end_addr = start_addr + (AxLEN + 1) * (2 ^ AxSIZE) - 1
start_addr[31:12] == end_addr[31:12]
```

## WSTRB/Byte Lane bug

常见形式：

- AXI4-Lite slave 忽略 `WSTRB`。
- Unaligned write 的 byte lane 映射错误。
- Width converter 拆分 `WSTRB` 错误。
- `TKEEP` 到 `WSTRB` 映射错误。

症状：部分字节被错误覆盖，尤其是 byte/halfword access 或 packet last beat。

## AW/W 解耦 bug

AXI 写地址和写数据独立。

常见错误：

- Slave 假设 AW 和 W 同周期到达。
- W 先到时没有 buffer。
- AW 已接收但 W 长时间不到导致资源泄漏。
- 多个 AW outstanding 时 W 对应关系错误。

Debug 时应分别观察 AW handshake、W handshake、内部 join 状态和 B response。

## Response bug

常见形式：

- 写事务没有 B response。
- 读错误没有完整返回所有 R beat。
- `BID/RID` 错误。
- Decode miss 没有 default error response。
- `BRESP/RRESP` 在 backpressure 下变化。

症状：master 永久等待、软件 bus fault 超时、scoreboard ID mismatch。

## ID/Outstanding bug

常见形式：

- Outstanding counter underflow/overflow。
- Response ID 不属于 active request。
- ID remap 复用 active downstream ID。
- Interconnect 未拼接 master port，多个 master ID 冲突。
- `RLAST` 丢失导致 read entry 永不释放。

这类 bug 通常只在多 outstanding 和 random latency 下暴露。

## Ordering bug

常见形式：

- Master 使用不同 ID，却假设 response 按 request 顺序返回。
- Interconnect 重排同 ID transaction。
- ID converter 改变了 ordering 语义。
- Doorbell 写早于 descriptor/data 对外可见。

Debug 需要 transaction log，而不只是看单个 channel 波形。

## CDC/Reset bug

常见形式：

- 直接跨时钟传 `VALID/READY`。
- Reset 一个 domain 后另一个 domain 仍等待 response。
- CDC FIFO pointer reset 不一致。
- Clock gating 时 `READY` 卡死为 0。
- Outstanding 未 drain 就 reset 或 power down。

这类 bug 常表现为低概率 hang，需要 random reset、clock ratio sweep 和长时间压力测试。

## Security/Protection bug

常见形式：

- Bridge 丢弃 `AxPROT`。
- Firewall 只看地址不看 master identity。
- 拒绝访问后没有 response。
- Debug master 绕过安全策略。
- SMMU stream ID 接错。

安全 bug 不一定导致功能失败，但可能导致严重安全漏洞。

## Performance bug

常见形式：

- Burst 太短。
- Outstanding 太小。
- Shared interconnect 争用严重。
- QoS 未生效。
- Width converter 窄口限制吞吐。
- Backpressure 未被性能计数器统计。

Debug 时同时看 bandwidth、latency、stall cycles、outstanding occupancy 和 DDR 侧统计。

## Debug 顺序建议

1. 看是否有 `VALID && READY` 握手。
2. 看 payload 在等待期间是否稳定。
3. 看 burst beat 数和 `LAST`。
4. 看 ID 和 outstanding table。
5. 看 response 是否返回且路由正确。
6. 看 decode/security/firewall 是否拒绝。
7. 看 CDC/reset/clock gating 状态。
8. 看性能计数器和 backpressure 来源。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `alexforencich-verilog-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `pulp-platform-axi-readme`
- `zipcpu-wb2axip-readme`

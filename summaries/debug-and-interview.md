# Debug and Interview

## Debug 顺序

1. 看 `VALID && READY` 是否发生。
2. 看 payload 在等待期间是否稳定。
3. 看 burst beat 数和 `LAST`。
4. 看 `WSTRB/TKEEP` byte lane。
5. 看 ID 和 outstanding table。
6. 看 response 是否返回。
7. 看 decode/security/firewall 是否拒绝。
8. 看 CDC/reset/clock gating。
9. 看 performance counter 和 backpressure。

## 高频问题

- 为什么 `VALID` 不能等 `READY`？
- AXI4 为什么没有 `WID`？
- `AxLEN` 表示什么？
- `WSTRB` 如何处理 unaligned write？
- `SLVERR` 和 `DECERR` 区别是什么？
- Outstanding transaction 如何释放？
- AXI ordering 和 cache coherency 有什么区别？
- AXI4-Lite 为什么不适合大块数据搬运？
- AXI4-Stream 的 `TLAST/TKEEP` 有什么作用？

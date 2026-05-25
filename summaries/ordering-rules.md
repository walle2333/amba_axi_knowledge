# Ordering Rules

## 快速判断

分析 ordering 时先问：

- Read 还是 write？
- 同 ID 还是不同 ID？
- 同一个 subordinate/地址区域还是不同目标？
- 是否涉及 barrier、exclusive、atomic、coherency？

## 工程规则

- 同 ID 通常需要更强 ordering，具体规则以 Arm 规格为准。
- 不同 ID 通常允许更多并发和乱序返回。
- Master 必须用 `RID/BID` 匹配 response，不能只靠返回顺序。
- AXI ordering 不等于 cache coherency。
- Doorbell/descriptor 场景需要 memory barrier、cache maintenance 或 coherent path。
- ID remap/serialization 不能破坏上游 ordering 语义。

## 常见风险

- 使用不同 ID 却假设 response 顺序等于 request 顺序。
- Interconnect 重排同 ID transaction。
- DMA descriptor 对外不可见时已经写 doorbell。
- Non-coherent DMA 场景误以为 AXI ordering 能刷新 cache。

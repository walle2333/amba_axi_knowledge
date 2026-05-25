# SoC Design Checklist

## Topology

- [ ] Master/slave 清单完整。
- [ ] 高带宽数据面和低速控制面分层。
- [ ] Memory map 唯一 decode，无意外重叠。
- [ ] 未映射地址有 default error response。

## Interface Parameters

- [ ] 每个端口的 data width 已确认。
- [ ] 每个端口的 ID width 已确认。
- [ ] Outstanding depth 已确认。
- [ ] Burst capability 已确认。
- [ ] `AxCACHE/AxPROT/AxQOS/AxUSER` 处理策略已确认。

## Integration

- [ ] CDC/clock converter 放置清楚。
- [ ] Reset 和 low-power drain 策略清楚。
- [ ] Width converter、ID converter、protocol converter 已验证。
- [ ] Security/firewall/SMMU 规则覆盖所有 master。
- [ ] Coherency 和 cache maintenance 策略清楚。
- [ ] QoS 和性能预算有验证计划。

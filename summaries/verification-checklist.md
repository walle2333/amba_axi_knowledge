# Verification Checklist

## Protocol

- [ ] `VALID` 不依赖 `READY`。
- [ ] Payload 在 `VALID=1 && READY=0` 时稳定。
- [ ] `WLAST/RLAST` 与 `AxLEN` 匹配。
- [ ] Burst 不跨 4KB boundary。
- [ ] `BID/RID` 匹配 active request。

## Stress

- [ ] 每个 channel 独立 random backpressure。
- [ ] 多 outstanding。
- [ ] 同 ID 和不同 ID 混合。
- [ ] Narrow/unaligned transfer。
- [ ] Error response injection。
- [ ] Reset during active transaction。

## System

- [ ] CPU AXI4-Lite 配置外设。
- [ ] DMA read/write DDR。
- [ ] Stream `TLAST/TKEEP`。
- [ ] Security violation。
- [ ] Non-coherent DMA cache maintenance。
- [ ] QoS/performance stress。

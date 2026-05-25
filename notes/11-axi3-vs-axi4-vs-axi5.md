# 11. AXI3 vs AXI4 vs AXI5

## 目的

AXI 协议随 AMBA 版本演进。实际 SoC 和 IP 集成时，可能同时遇到 AXI3 legacy IP、AXI4 主流 IP、AXI5/AMBA 5 扩展特性。本篇整理三者的工程差异和集成注意点。

完整细节必须以对应 Arm 官方规格为准。本篇侧重实际设计和集成中最常遇到的差异。

## 总览

| 项目 | AXI3 | AXI4 | AXI5/AMBA 5 相关扩展 |
| --- | --- | --- | --- |
| 主流程度 | legacy | 当前广泛使用 | 新系统和特定扩展 |
| Burst length | 较短 | 最多 256 beats | 延续并扩展属性 |
| Write data interleaving | 支持 | 不支持 | 不回到 AXI3 模型 |
| W channel ID | 有 `WID` | 通常无 `WID` | 关注新增扩展信号 |
| Atomic operation | 非主要特性 | 非 AXI4 基础特性 | 引入 ATOPs 等扩展 |
| 工程复杂度 | W interleaving 复杂 | 更易实现和验证 | 扩展能力更强，验证更复杂 |

## AXI3

AXI3 是较早版本，可能出现在旧 IP、旧 FPGA/SoC 平台或 legacy subsystem 中。

AXI3 的一个重要特性是支持 write data interleaving。也就是说，不同 write transaction 的 W data beat 可以按 ID 交织出现。

简化示例：

```text
AWID=1, AWADDR=A
AWID=2, AWADDR=B

WID=1, data beat 0
WID=2, data beat 0
WID=1, data beat 1
WID=2, data beat 1
```

这能提升某些场景下的并发灵活性，但显著增加 subordinate、interconnect、width converter 和 verification 复杂度。

## AXI4

AXI4 是目前最常见的 memory-mapped AXI 版本。与 AXI3 相比，一个关键变化是移除了 write data interleaving。

AXI4 中 W channel 通常没有 `WID`，写数据必须按事务顺序发送，不能交织不同 write transaction 的 W beats。

工程影响：

- subordinate 不需要根据 `WID` 重新组装多个 write data stream。
- interconnect 的 write data routing 简化。
- verification 状态空间减少。
- AW/W 解耦仍然存在，slave 仍需处理地址和数据不同周期到达。

AXI4 仍支持：

- burst transaction。
- outstanding transaction。
- transaction ID。
- QoS、protection、cache、region、user sideband。
- AXI4-Lite 和 AXI4-Stream 生态。

## AXI4-Lite 与 AXI4-Stream

AXI4 时代常见三个接口同时存在：

```text
AXI4 full       高吞吐 memory-mapped 数据面
AXI4-Lite       控制寄存器面
AXI4-Stream     无地址流数据面
```

不要因为名字都带 AXI 就混淆它们。AXI4-Stream 没有地址和 response，AXI4-Lite 没有 burst，AXI4 full 才是完整 memory-mapped 高性能接口。

## AXI5 和 AMBA 5 扩展

AXI5 属于 AMBA 5 相关演进，增加或规范了更多现代 SoC 所需能力。工程上常见讨论点包括 atomic operations，也就是 ATOPs。

PULP AXI 文档中使用 “AXI4+ATOPs” 描述其实现：基础是 AXI4，再加入 AMBA 5 中定义的 atomic operations。

ATOPs 的工程注意点：

- 不支持 ATOP 的 master 应将 `aw_atop` 置 0。
- 不支持 ATOP 的 subordinate 需要在接口文档中明确，并可通过 filter 阻挡。
- 某些 ATOP 可能同时产生 B channel response 和 R channel response。
- ATOP 不应与其他 outstanding transaction 使用相同 AXI ID。

这些规则会影响 interconnect、scoreboard、response handling 和 verification。

## 集成 Legacy AXI3 IP

如果 SoC 中存在 AXI3 IP，需要重点确认：

- 是否使用 write data interleaving。
- 是否需要 AXI3-to-AXI4 protocol converter。
- `WID` 如何处理或移除。
- Burst length、lock、cache/prot 等信号是否兼容。
- Verification VIP 是否配置成正确协议版本。

很多团队会在 SoC 边界把 legacy AXI3 转成 AXI4，降低主 interconnect 的复杂度。

## 集成 AXI5/ATOP IP

如果某个 master 可能发出 ATOP：

- 确认 interconnect 是否传播 `aw_atop`。
- 确认下游 subordinate 是否支持 ATOP。
- 不支持的路径上是否有 ATOP filter。
- Scoreboard 是否支持一个请求同时关联 B 和 R response。
- ID 分配是否避免 ATOP 与其他 outstanding transaction 冲突。

## 版本差异对 Verification 的影响

AXI3 verification 重点：

- W data interleaving。
- `WID` 与 AW ID 匹配。
- 交织写数据下的 response ordering。

AXI4 verification 重点：

- W channel 不交织。
- Burst、ID、outstanding、ordering。
- 4KB boundary、WSTRB、narrow/unaligned transfer。

AXI5/ATOP verification 重点：

- `aw_atop` 合法性。
- ATOP response 组合。
- ATOP 与 outstanding ID 约束。
- 不支持 ATOP 的路径是否被过滤。

## 常见 bug

- 把 AXI3 IP 当 AXI4 IP 接入，忽略 `WID` 和 write data interleaving。
- AXI4 slave 错误支持交织写数据，导致实现过度复杂或行为不一致。
- AXI5/ATOP 信号未接或悬空，导致不确定行为。
- Verification VIP 协议版本配置错误。
- 文档中只写 “AXI interface”，没有明确 AXI3、AXI4、AXI4-Lite、AXI4-Stream 或 AXI5 扩展。

## SoC 实践建议

- IP datasheet 必须明确协议版本和可选特性。
- SoC interconnect 主干优先统一为 AXI4 或更明确的内部协议。
- Legacy AXI3 IP 尽量通过边界 converter 隔离。
- ATOP、ACE、CHI 等高级特性不要默认透传，必须有系统级一致性和验证策略。
- Verification plan 中单独列出协议版本矩阵。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `alexforencich-verilog-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `fpganinja-taxi-readme`

# 16. Security and Protection

## 目的

AXI 访问不仅包含地址和数据，还包含 protection、security、privilege 等属性。SoC 中通常通过 `AxPROT`、TrustZone、firewall、TZC、SMMU/IOMMU、MPU/MMU 和 interconnect access control 共同实现访问保护。

## AxPROT

AXI 地址通道上的 `AxPROT` 用于携带 protection 属性。常见语义包括：

- privileged 或 unprivileged access。
- secure 或 non-secure access。
- instruction 或 data access。

具体 bit 定义以 Arm 官方规格为准。工程上要避免只把 `AxPROT` 当作无用 sideband，因为它可能决定访问是否允许。

## TrustZone 与 Secure/Non-Secure

Arm TrustZone 系统中，地址访问通常分为 secure 和 non-secure。Non-secure master 不应访问 secure-only 资源。

典型资源划分：

```text
Secure ROM
Secure SRAM
Secure peripheral registers
Non-secure DDR region
Shared memory region
```

Interconnect、TZC 或 firewall 会根据 master 身份、地址和 security 属性决定是否允许访问。

## Master 身份

安全检查不能只看地址，还需要知道访问来自哪个 master。

例子：

```text
CPU secure world  -> 允许访问 secure key store
CPU non-secure    -> 拒绝访问 secure key store
Debug master      -> 取决于 lifecycle/debug policy
DMA               -> 只能访问分配给它的 buffer
NPU               -> 只能访问模型和 tensor buffer 区域
```

有些系统会在 interconnect 中给每个 master port 绑定安全身份或 VMID/stream ID。

## Firewall

AXI firewall 或 bus firewall 用于限制 master 对地址区域的访问。

常见规则：

```text
master_id = DMA0, allowed = DDR[0x8000_0000..0x8FFF_FFFF]
master_id = NPU,  allowed = DDR[0x9000_0000..0x9FFF_FFFF]
master_id = CPU_NS, denied = Secure SRAM
```

拒绝访问时常见行为：

- 返回 `SLVERR` 或 `DECERR`。
- 记录 violation log。
- 触发安全中断。
- 在严重场景下触发系统 reset 或 lifecycle 事件。

## SMMU/IOMMU

复杂 SoC 中，DMA、GPU、NPU 等 master 可能通过 SMMU/IOMMU 进行地址转换和权限检查。

作用：

- I/O virtual address 到 physical address 转换。
- per-device address space。
- 访问权限控制。
- 隔离不同 VM/process/container。

AXI 侧可能需要携带 stream ID、substream ID 或其他 sideband 信息，供 SMMU/IOMMU 区分访问来源。

## Protection 与 Error Response

安全拒绝访问时，系统需要定义响应方式。

常见选择：

| 场景 | 可能响应 |
| --- | --- |
| 地址未映射 | DECERR |
| 地址映射存在但权限不足 | SLVERR 或 DECERR |
| Firewall 拒绝 | SLVERR/DECERR + violation log |
| SMMU page fault | error response + fault record |

具体响应需要软件可诊断。只返回泛化错误会增加 debug 难度。

## Privileged 与 User Access

`AxPROT` 中的 privilege 属性可用于区分 privileged 和 unprivileged access。外设寄存器可以基于该属性限制访问。

例子：

```text
普通用户态访问 timer control register -> 拒绝
kernel/secure firmware 访问同一寄存器 -> 允许
```

实际是否使用取决于 interconnect、CPU、MMU 和外设设计。

## Instruction 与 Data Access

`AxPROT` 还可区分 instruction fetch 和 data access。安全系统可能禁止从某些区域取指，或禁止把外设区当作可执行内存。

这类策略通常与 MPU/MMU page attributes、execute-never、firewall 配合实现。

## Debug Access

Debug master 经常有特殊权限。量产芯片中必须明确 debug access policy：

- secure debug 是否关闭。
- non-secure debug 是否可访问 secure memory。
- lifecycle state 是否影响 debug AXI access。
- violation 是否记录。

Debug 口如果绕过 AXI firewall，可能成为严重安全漏洞。

## Verification Checklist

- Secure/non-secure master 访问 secure region 是否按预期允许或拒绝。
- DMA 是否不能越界访问其他 buffer。
- Firewall violation 是否返回 response，不能挂死。
- Violation log 是否记录 master、地址、类型和时间。
- `AxPROT` 是否在 bridge/converter/interconnect 中保留。
- Reset 后 firewall 默认策略是否安全。
- Debug master 权限是否符合 lifecycle 配置。

## 常见 bug

- Bridge 或 width converter 丢弃 `AxPROT`。
- Firewall 只检查地址，不检查 master identity。
- 安全拒绝后没有 response，导致总线挂死。
- 默认配置全开放，secure boot 早期窗口可被 DMA 访问。
- Debug AXI master 绕过安全策略。
- SMMU stream ID 接错，设备访问了错误地址空间。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `arm-amba-architecture-page`
- `alexforencich-cocotbext-axi-readme`
- `pulp-platform-axi-readme`

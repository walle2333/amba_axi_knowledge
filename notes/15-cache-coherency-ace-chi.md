# 15. Cache Coherency, ACE and CHI

## 目的

AXI 本身是 memory-mapped interface，但普通 AXI4 不自动解决 cache coherency。现代 SoC 中，CPU、DMA、GPU、NPU、ISP 等 master 可能同时访问 DDR 或 shared memory。是否能看到同一份最新数据，取决于 cache coherency 架构、memory attribute、软件 cache maintenance、ACE/ACE-Lite/CHI 等协议和系统设计。

本篇从 SoC 工程角度整理 AXI 与 coherency 的关系。ACE/CHI 的完整协议细节必须以 Arm 官方规格为准。

## Coherency 问题是什么

典型 non-coherent DMA 场景：

```text
1. CPU 把数据写入 cache line，但还没有写回 DDR。
2. DMA 从 DDR 读取同一地址。
3. DMA 读到旧数据，因为最新数据还在 CPU cache 中。
```

另一个方向：

```text
1. DMA 把数据写入 DDR。
2. CPU 读取同一地址。
3. CPU cache 中已有旧 cache line，CPU 读到旧数据。
```

AXI transaction 顺序不等于 cache coherency。即使 AXI ordering 正确，cache 中的数据也可能不是最新。

## Coherent 与 Non-Coherent

| 类型 | 特点 | 软件责任 |
| --- | --- | --- |
| non-coherent access | 访问不参与 cache coherency | 软件需要 cache clean/invalidate |
| I/O coherent access | I/O master 与 CPU cache 有一定一致性支持 | 软件责任减少，但仍需正确 memory attribute |
| fully coherent access | master 作为 coherent agent 参与一致性协议 | 硬件协议维护一致性 |

很多低成本 DMA 是 non-coherent。高性能 SoC 可能让部分 accelerator 或 I/O master 具备 I/O coherency。

## AXI4 与 Cache Attribute

AXI 地址通道包含 `AxCACHE` 等属性，用于描述 cacheability、bufferability 等访问属性。实际系统中，这些属性会与 MMU/MPU、interconnect、snoop control unit、cache controller 和 memory system 共同决定访问行为。

需要注意：

- 简单 AXI slave 可能忽略 `AxCACHE`。
- Interconnect 可能使用 `AxCACHE` 影响路径或属性转换。
- CPU 发出的属性通常来自 page table memory attribute。
- DMA 的属性可能由 descriptor、寄存器或硬件固定配置决定。

## ACE

ACE 是 AXI coherency extension，用于让 coherent master 参与 snoop 和 cache coherency 事务。它在 AXI 基础上增加一致性相关通道和语义。

典型用途：

```text
CPU cluster coherent interconnect
coherent accelerator
需要硬件 snoop CPU cache 的 master
```

ACE 的完整行为包括 snoop、cache state、barrier、domain 等复杂语义，必须参考 Arm 官方 ACE 规格。

## ACE-Lite

ACE-Lite 通常用于 I/O coherent master。它比完整 ACE 简化，适合 DMA 或 accelerator 这类不一定拥有完整 cache hierarchy 的 master。

典型场景：

```text
DMA 读取 CPU cache 中最新数据，或 DMA 写入后让 CPU cache 一致性系统可见
```

是否使用 ACE-Lite 取决于 SoC coherency fabric 是否支持，以及该 master 是否接入对应一致性域。

## CHI

CHI 是 Arm 后续的 coherent hub interface，常用于现代多核、高性能 coherent interconnect 或 NoC。它比 AXI/ACE 更适合复杂 coherent fabric。

典型场景：

```text
multi-core CPU cluster
coherent NoC
large-scale cache-coherent SoC
```

本知识库后续需要补齐 Arm CHI 官方规格本地资料。目前已记录官方入口，但规格 PDF 仍需手工下载或确认访问限制。

## Non-Coherent DMA 软件流程

CPU 到 DMA：

```text
1. CPU 写 buffer。
2. CPU clean cache，将 dirty cache line 写回 DDR。
3. memory barrier，确保写回完成。
4. CPU 写 DMA descriptor/doorbell。
5. DMA 从 DDR 读取最新 buffer。
```

DMA 到 CPU：

```text
1. CPU 准备接收 buffer。
2. CPU invalidate 或避免 cache 中存在旧 line。
3. DMA 写 DDR。
4. DMA interrupt 通知 CPU。
5. CPU invalidate cache line，读取 DMA 写入的新数据。
```

具体 clean/invalidate 顺序依赖 CPU 架构、OS DMA API 和 memory attribute。

## Coherency 与 Ordering

Ordering 只约束 transaction 的观察顺序，不保证 cache 内容最新。Coherency 保证多个 observer 对同一地址的数据一致性。

例子：

```text
CPU write data -> CPU cache
CPU write doorbell -> device register
```

如果没有 cache clean 或 coherent path，device 即使按顺序看到 doorbell，也可能读不到 CPU cache 中尚未写回的数据。

## SoC 集成检查点

- 每个 master 是否 coherent。
- 每个 master 的 `AxCACHE/AxPROT` 属性来源。
- DMA buffer 使用 cacheable 还是 non-cacheable memory。
- OS/firmware 是否使用正确 DMA API。
- Interconnect 是否连接到 coherent domain。
- ACE/ACE-Lite/CHI 端口是否正确配置。
- Non-coherent master 是否有软件 cache maintenance 约束。
- Security/TrustZone 属性是否与 coherency domain 兼容。

## 常见 bug

- Non-coherent DMA 读取 CPU cache 中未写回的数据，导致读旧值。
- DMA 写 DDR 后 CPU 未 invalidate，读到旧 cache line。
- DMA descriptor clean 了，但数据 buffer 未 clean。
- 把 AXI ordering 当成 cache coherency。
- `AxCACHE` 配置错误，导致访问被当成 non-cacheable 或 cacheable。
- Coherent master 接到了 non-coherent path。
- ACE/CHI 高级信号未正确连接或被错误裁剪。

## Verification Checklist

- Non-coherent DMA 测试是否覆盖 cache clean/invalidate 缺失场景。
- Coherent master 是否能观察其他 cache master 的最新写入。
- `AxCACHE` 属性是否按 page/descriptor 配置生成。
- Snoop/filter 路径是否覆盖 reset、power down、clock gating。
- 多 master 同地址访问是否有系统级一致性测试。
- 软件 driver 是否使用正确 DMA mapping API。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `arm-amba-chi-protocol-spec`
- `arm-amba-architecture-page`
- `pulp-platform-axi-readme`
- `pulp-platform-axi-microarchitecture-paper`

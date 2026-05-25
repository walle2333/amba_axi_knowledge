# 08. Ordering Model

## 目的

AXI ordering model 定义了多个 transaction 之间允许或不允许重排的边界。它决定 master 能否依赖访问顺序，也决定 interconnect、crossbar、NoC、ID converter、DMA 和 subordinate 需要保留哪些顺序关系。

本篇先给出工程上最重要的理解框架。完整细节应以 Arm 官方 AXI/ACE 规格为准。

## 为什么需要 Ordering Model

AXI 支持 outstanding transaction 和不同 ID 的并发处理。如果没有 ordering 规则，系统虽然可能获得更高吞吐，但软件和硬件都无法判断访问结果的先后关系。

Ordering model 的目标是在性能和可预测性之间折中：

- 允许不同 ID、不同目标、不同通路并发执行。
- 对需要保持顺序的事务施加约束。
- 让 master、interconnect、subordinate 对 response 顺序有共同理解。

## 三个核心问题

分析 AXI ordering 时，先问三个问题：

1. 这些 transaction 是 read 还是 write？
2. 它们使用相同 ID 还是不同 ID？
3. 它们访问同一个 subordinate/地址区域，还是不同目标？

## 同 ID 顺序

同 ID transaction 通常需要保持更强的顺序关系。master 如果希望某一组访问按顺序被观察，就应使用相同 ID 或额外同步机制。

例子：同 ID read：

```text
ARID=2, ARADDR=A
ARID=2, ARADDR=B
```

工程直觉：返回给 master 时，同 ID read response 不应让 B 的完整响应越过 A 的完整响应。具体允许的 interleaving 和约束需以官方规格为准。

例子：同 ID write：

```text
AWID=5, AWADDR=A
AWID=5, AWADDR=B
```

如果 master 依赖 A 写先于 B 写被系统观察，应使用同 ID 并确保目标和 interconnect 支持相应 ordering。

## 不同 ID 并发

不同 ID transaction 通常允许更自由的调度和返回。这样 interconnect 和 subordinate 可以重排请求，以提高吞吐。

例子：

```text
ARID=1, ARADDR=DDR row miss
ARID=2, ARADDR=DDR row hit
```

DDR controller 可能先返回 `RID=2` 的数据，以提高效率。master 必须用 `RID` 匹配响应，不能假设返回顺序等于发起顺序。

## Read Ordering

Read ordering 的关键是 R channel 返回顺序和 `RID`。

对于多个 outstanding read：

- master 不能只按返回先后判断数据属于哪个 request。
- master 必须使用 `RID` 匹配 transaction。
- 对于 burst read，直到 `RLAST` 才表示该 read transaction 完成。

例子：不同 ID read 乱序返回：

```text
cycle 1: ARID=1, ARADDR=0x8000_0000, ARLEN=3
cycle 2: ARID=2, ARADDR=0x9000_0000, ARLEN=3

later:
RID=2 beat0
RID=2 beat1
RID=2 beat2
RID=2 beat3 RLAST=1
RID=1 beat0
RID=1 beat1
RID=1 beat2
RID=1 beat3 RLAST=1
```

如果 master 的 reorder buffer 不支持这种情况，就不能发出可能乱序返回的不同 ID read，或者需要 interconnect/subordinate 配置成保序。

## Write Ordering

Write ordering 涉及 AW、W 和 B 三个 channel。

AXI4 中 W channel 不支持 write data interleaving，因此写数据流本身按事务顺序发送。但 AW channel 可以提前发出多个写地址，B response 也可能受目标处理延迟影响。

写 ordering 需要关注：

- AW 接收顺序。
- W beat 与 AW transaction 的对应关系。
- B response 返回顺序。
- 写入对其他 master 或 memory observer 的可见顺序。

实际设计中，简单 subordinate 通常按接收顺序处理写事务。高性能 interconnect 和 DDR controller 可能更复杂，但必须遵守 AXI ordering 约束。

## Read 与 Write 之间的顺序

AXI 的 read channel 和 write channel 是相互独立的。一个 master 发出 write 后再发 read，并不自动意味着 read 一定观察到 write 的结果，除非满足规格定义的 ordering 条件或系统使用 barrier、fence、cache maintenance、软件同步等机制。

例子：

```text
write descriptor to memory
write doorbell register to device
```

这类场景通常需要软件或硬件保证 descriptor 写入对 device 可见后，再写 doorbell。否则 device 可能看到 doorbell，但读取到旧 descriptor。

在 CPU 系统中，这通常涉及 memory attribute、barrier instruction、cache maintenance 和 interconnect ordering。

## Interconnect 的 Ordering 责任

Interconnect 不能只做地址 mux。它需要在性能和顺序约束之间做正确取舍。

常见职责：

- 根据地址 decode 路由请求。
- 保持同 ID ordering。
- 对不同目标维护必要的 response routing。
- 防止不允许的重排。
- 在 ID remap/serialize 时保持原始 ordering 语义。
- 对 decode miss 返回错误响应，而不是破坏 ordering 或挂死。

alexforencich 的 AXI crossbar 文档提到 ID-based transaction ordering protection logic，PULP AXI 提供 ID remap、ID serialize、ID width converter 等模块，这些都说明 ordering 是实际 interconnect 设计的核心问题。

## ID Remap 对 Ordering 的影响

ID remap 会改变下游可见 ID，因此必须确保不会破坏上游 ordering。

错误例子：

```text
上游两个 active transaction 使用不同上游 ID
ID remap 错误地映射到同一个下游 ID
下游 response 返回后无法唯一匹配
```

正确设计通常需要：

- active remap table。
- downstream ID 分配和释放规则。
- 对无法安全并发的 transaction 施加 backpressure。
- 必要时 serialization。

## Barrier、Exclusive、Atomic

更高阶的 ordering 会涉及 barrier、exclusive access、atomic operation、ACE/CHI coherency 等主题。

PULP AXI 文档提到 AXI4+ATOPs，即 AXI4 加上 AMBA 5 atomic operations。其中特别指出支持 ATOP response 的模块需要能处理同一 ATOP 同时产生 B channel beat 和至少一个 R channel beat，并且 ATOP 不应与其他 outstanding transaction 使用相同 AXI ID。

这些内容后续会在 AXI5、ACE/CHI、cache coherency 和 atomic operation 章节展开。

## SoC 实践场景

### CPU 写配置寄存器

CPU 通过 AXI4-Lite 写外设寄存器。因为 AXI4-Lite 通常无 burst、outstanding 很小，ordering 简单。但 CPU memory ordering 和 device memory attribute 仍然重要。

### DMA 描述符

软件写 descriptor 到 DDR，再写 DMA doorbell。需要确保 DMA 读取 descriptor 时看到最新内容，可能需要 cache clean、memory barrier 和 coherent/non-coherent 配置。

### 多 Master 访问同一内存

CPU 和 DMA 同时访问同一 DDR 区域时，AXI ordering 本身不等于 cache coherency。是否看到最新数据取决于 cache coherency、memory attribute、software synchronization。

### NoC 中的多路径

NoC 可能存在多条路径或不同 latency。不同 ID transaction 可能乱序返回，NoC 必须保证不违反同 ID 或协议要求的 ordering。

## Verification Checklist

- 同 ID read response 是否按规则返回。
- 不同 ID read response 乱序时 master 是否能正确匹配。
- B response 的 ID 和顺序是否满足 write ordering 要求。
- ID remap 是否保持上游 ordering 语义。
- Backpressure 和 arbitration 是否会导致同 ID starvation。
- Decode error 是否仍按正确 ID 返回。
- Reset/flush 时 outstanding ordering 状态如何清理。

## 常见 bug

- Master 假设 response 顺序等于 request 顺序，但使用了不同 ID。
- Interconnect ID remap 破坏同 ID ordering。
- Crossbar 对同一 master 的同 ID transaction 发送到多个 slave 后无法正确保序。
- DMA doorbell 写早于 descriptor 对外可见，导致设备读取旧 descriptor。
- Non-coherent DMA 场景中误以为 AXI ordering 能解决 cache 一致性问题。
- Atomic/ATOP 使用与其他 outstanding transaction 相同 ID。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `alexforencich-verilog-axi-readme`
- `alexforencich-cocotbext-axi-readme`
- `xilinx-pg059-axi-interconnect`

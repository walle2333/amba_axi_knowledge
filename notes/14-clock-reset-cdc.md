# 14. Clock, Reset and CDC

## 目的

AXI 常常跨越不同 clock domain 和 reset domain。CDC 和 reset 处理不当会造成数据丢失、协议死锁、response 丢失、metastability 和难以复现的系统 bug。本篇整理 AXI CDC、reset sequencing、outstanding drain 和验证重点。

## 为什么 AXI 需要 CDC

典型 SoC 中，不同模块可能运行在不同频率：

```text
CPU/interconnect clock: 800 MHz
DDR controller clock: 400 MHz or PHY-related clock
peripheral clock: 50 MHz
video/ISP clock: pixel clock
debug clock: independent low-speed clock
```

这些模块之间如果通过 AXI 连接，就需要 clock converter 或 async bridge。

## 不要直接跨时钟连接 VALID/READY

`VALID`、`READY` 和 payload 不能简单用组合或单级触发器跨时钟域。AXI channel 是多 bit payload 加握手控制，必须作为完整 channel 跨域。

错误做法：

```text
source clock domain 的 AWVALID 直接接到 destination clock domain
AWADDR 用普通寄存器打一拍
AWREADY 直接返回
```

这会产生 metastability、payload/control 不一致和握手丢失。

## 常见 CDC 结构

### Async FIFO per Channel

最常见做法是每个 AXI channel 使用异步 FIFO 或等价结构：

```text
AW async FIFO
W  async FIFO
B  async FIFO
AR async FIFO
R  async FIFO
```

PULP AXI 文档中的 `axi_cdc` 使用 Gray FIFO 思路跨时钟域，是典型方法。

### Full AXI Clock Converter

AXI clock converter 将五个 channel 的 CDC、buffer、backpressure 和 reset 统一封装。实际 SoC 集成中建议优先使用经过验证的 converter，而不是临时拼接多个小 FIFO。

### AXI4-Lite Clock Converter

AXI4-Lite 并发较小，可以使用更简单的 CDC 结构，但仍必须保持五通道语义，不能破坏 AW/W/B/AR/R 的响应关系。

## Gray FIFO 基本思想

异步 FIFO 通常使用 Gray-coded pointer 跨时钟域同步读写指针，降低多 bit 同时变化造成的采样风险。

AXI CDC 不只是 FIFO 本身，还要确保：

- payload 和 control 同步进入 FIFO。
- FIFO full 正确反压 upstream `READY`。
- FIFO empty 正确控制 downstream `VALID`。
- reset 时读写指针和 valid 状态一致。

## Reset Domain 问题

AXI reset 设计需要考虑：

- reset 是否同步释放。
- 两侧 clock domain reset 顺序是否固定。
- reset 期间是否允许新 transaction。
- reset 时 outstanding transaction 如何处理。
- reset 后 response 是否可能丢失或重复。

如果一个 domain reset 而另一个 domain 仍在运行，CDC bridge 必须定义清楚行为。

## Outstanding Drain

在进入 low-power、clock gating、reset 或重新配置前，常需要 drain outstanding transactions。

Drain 的含义：

```text
停止接收新 request
等待所有已接收 request 的 response 完成
确认所有 channel idle
再关闭时钟、复位或切换配置
```

如果不 drain，可能出现：

- master 等不到 response。
- subordinate 完成写入但 response 丢失。
- reset 后旧 response 被新 transaction 误接收。

## Clock Gating

AXI 接口所在 clock domain 做 clock gating 时，需要确认：

- gating 前没有未完成握手。
- 或者接口有 isolation/retention 机制。
- `READY`/`VALID` 不会停在导致对端永久等待的状态。
- 唤醒后不会发送 reset 前遗留的无效 payload。

## CDC 与 Backpressure

CDC FIFO 满时会向 upstream 反压：

```text
downstream clock 慢或暂停
-> FIFO full
-> upstream READY=0
-> upstream 停止发送
```

如果多个 channel 的 FIFO 深度配置不合理，可能形成死锁。例如 W channel FIFO 满阻塞数据，AW channel 又因内部资源等待 W 完成而停滞。

## Reset Sequencing 示例

一个较稳妥的复位/关断流程：

```text
1. 禁止 master 发起新 transaction。
2. 等待 outstanding counter 归零。
3. 等待 AXI CDC FIFO empty。
4. 拉高 isolation 或让接口进入 idle。
5. 关闭 clock 或 assert reset。
6. 恢复时先释放 clock/reset，再允许新 request。
```

具体流程依赖 SoC power controller 和 interconnect 实现。

## Verification Checklist

- CDC 是否使用经过验证的 async FIFO 或 clock converter。
- 每个 channel 是否独立保持 payload 稳定性。
- Random clock ratio 下是否无丢失、无重复。
- Reset 在任意 transaction 中间发生时行为是否符合设计规范。
- Reset 释放后是否没有旧 response 泄漏。
- Drain 流程是否能等待所有 outstanding 完成。
- FIFO full/empty 边界条件是否正确。
- Clock gating 前后 `VALID/READY` 是否安全。

## 常见 bug

- 直接跨时钟传 `VALID/READY`。
- payload 多 bit 信号未与 valid 同步跨域。
- reset 一个 domain 后另一个 domain 永久等待 response。
- CDC FIFO 深度不足导致压力下死锁。
- reset 后 FIFO pointer 不一致。
- clock gating 时 `READY` 卡死为 0，导致上游永远等待。

## 参考资料

- `arm-amba-axi-ace-protocol-spec`
- `pulp-platform-axi-readme`
- `alexforencich-verilog-axi-readme`
- `zipcpu-wb2axip-readme`
- `fpganinja-taxi-readme`

# Glossary

## A

- **AMBA**: Advanced Microcontroller Bus Architecture，Arm 定义的一组片上总线协议族。
- **AXI**: Advanced eXtensible Interface，AMBA 协议族中的高性能 memory-mapped interface。
- **AXI4-Lite**: AXI4 的简化 memory-mapped 子集，常用于低带宽寄存器访问。
- **AXI4-Stream**: 面向流数据传输的 AXI 协议变体，不使用地址通道。

## B

- **Beat**: burst transaction 中一次数据传输。
- **Burst**: 一次地址握手后连续传输多个 beat 的事务。

## C

- **CDC**: Clock Domain Crossing，跨时钟域处理。

## H

- **Handshake**: AXI 中由 `VALID` 和 `READY` 共同完成的传输确认机制。

## I

- **ID**: AXI transaction identifier，用于区分 outstanding transactions 和 ordering 关系。

## O

- **Outstanding Transaction**: 已发起但尚未完成响应的事务。

## Q

- **QoS**: Quality of Service，用于表示事务服务质量或优先级的属性。

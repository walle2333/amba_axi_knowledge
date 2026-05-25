# AMBA AXI Knowledge Base

这是一个面向长期维护的 AMBA AXI 知识库，系统整理 AXI 基础协议、进阶规则、SoC 集成实践、验证方法、常见问题、示例和可靠原始参考资料。

内容以中文为主，关键术语保留英文，例如 `VALID/READY`、`burst`、`outstanding transaction`、`ordering`、`AXI4-Lite`、`AXI4-Stream`、`ACE`、`CHI`。

GitHub 仓库：<https://github.com/walle2333/amba_axi_knowledge>

## 当前状态

已完成第一版主干：

- `notes/00` 到 `notes/20` 已完整覆盖 AMBA AXI 基础、进阶协议、SoC 实践和验证主题。
- `summaries/` 已包含 cheatsheet、信号表、burst 示例、ordering 规则、SoC checklist、verification checklist、debug/interview 摘要。
- `examples/` 已包含 timing、transaction、RTL pattern、AXI4-Lite、AXI master、AXI4-Stream、AXI-to-APB bridge、DMA、interconnect、verification 的第一版示例说明。
- `sources/` 已保存公开可归档的厂商 PDF、开源项目 README、论文 PDF，以及 Arm 官方规格入口说明。
- `metadata/sources.yaml` 已登记资料来源、URL、本地路径、可靠性和当前状态。

仍需后续增强：

- 手工下载或确认 Arm AXI/ACE 官方规格 PDF。
- 手工下载或确认 Arm CHI 官方规格 PDF。
- 替换 AMD/Xilinx UG1037 的完整 PDF 或离线正文归档。
- 增加更多 RTL 代码、SVG/波形图、完整 DMA bring-up 案例。

## 快速入口

- 总索引：[`index.md`](index.md)
- 资料登记：[`metadata/sources.yaml`](metadata/sources.yaml)
- 术语表：[`metadata/glossary.md`](metadata/glossary.md)
- 待办事项：[`metadata/todo.md`](metadata/todo.md)
- 速查表：[`summaries/axi-cheatsheet.md`](summaries/axi-cheatsheet.md)
- 信号表：[`summaries/signal-table.md`](summaries/signal-table.md)
- SoC 检查表：[`summaries/soc-design-checklist.md`](summaries/soc-design-checklist.md)
- 验证检查表：[`summaries/verification-checklist.md`](summaries/verification-checklist.md)

## 学习路径

建议按以下顺序阅读：

1. [`00-overview.md`](notes/00-overview.md)
2. [`01-amba-family.md`](notes/01-amba-family.md)
3. [`02-axi-basics.md`](notes/02-axi-basics.md)
4. [`03-axi-channels.md`](notes/03-axi-channels.md)
5. [`04-valid-ready-handshake.md`](notes/04-valid-ready-handshake.md)
6. [`05-burst-transfer.md`](notes/05-burst-transfer.md)
7. [`06-response-and-error.md`](notes/06-response-and-error.md)
8. [`07-outstanding-and-id.md`](notes/07-outstanding-and-id.md)
9. [`08-ordering-model.md`](notes/08-ordering-model.md)
10. [`09-axi4-lite.md`](notes/09-axi4-lite.md)
11. [`10-axi4-stream.md`](notes/10-axi4-stream.md)
12. [`11-axi3-vs-axi4-vs-axi5.md`](notes/11-axi3-vs-axi4-vs-axi5.md)
13. [`12-interconnect-crossbar-noc.md`](notes/12-interconnect-crossbar-noc.md)
14. [`13-bridges-and-converters.md`](notes/13-bridges-and-converters.md)
15. [`14-clock-reset-cdc.md`](notes/14-clock-reset-cdc.md)
16. [`15-cache-coherency-ace-chi.md`](notes/15-cache-coherency-ace-chi.md)
17. [`16-security-protection.md`](notes/16-security-protection.md)
18. [`17-qos-performance.md`](notes/17-qos-performance.md)
19. [`18-verification.md`](notes/18-verification.md)
20. [`19-common-bugs.md`](notes/19-common-bugs.md)
21. [`20-soc-practice.md`](notes/20-soc-practice.md)

## 目录结构

```text
sources/    原始参考资料，按来源分类保存，不直接修改。
notes/      结构化知识笔记，中文为主。
examples/   时序、事务、RTL、SoC 场景和验证示例。
summaries/  速查表、检查表和调试总结。
metadata/   资料索引、术语表、阅读计划和待办事项。
scripts/    后续可能加入的资料处理脚本说明。
```

## 资料可靠性分级

```text
primary    Arm 官方规格书、Arm 官方技术文档、官方培训资料。
secondary  EDA/IP/FPGA/SoC 厂商公开文档、白皮书、应用说明。
tertiary   博客、课程笔记、GitHub 项目文档、论坛讨论。
```

使用原则：

- 协议合法性判断优先引用 `primary`。
- 工程实践和 IP 集成可参考 `secondary`。
- `tertiary` 只作为辅助理解，不作为关键协议结论的唯一依据。

## 已归档资料

已保存的公开资料包括：

- Xilinx AXI Interconnect product guide。
- Xilinx AXI DMA product guide。
- PULP AXI microarchitecture arXiv paper。
- PULP AXI README。
- alexforencich verilog-axi README。
- alexforencich cocotbext-axi README。
- ZipCPU wb2axip README。
- fpganinja taxi README。

受限或需要手工处理的资料：

- Arm AXI/ACE 官方规格：官方入口已记录，完整 PDF 需通过浏览器登录/接受协议后手工下载。
- Arm CHI 官方规格：官方入口已记录，完整 PDF 需通过浏览器登录/接受协议后手工下载。
- AMD/Xilinx UG1037：自动抓取只能得到 AMD docs JavaScript 页面外壳，需后续手工归档完整 PDF 或离线正文。

## 维护规则

新增资料时：

1. 将原始文件保存到 `sources/` 对应分类目录。
2. 在 `metadata/sources.yaml` 登记来源、URL、访问日期、版本、可靠性和本地路径。
3. 如果资料无法直接下载，登记 URL、访问日期、访问限制和后续处理方式。
4. 在相关笔记底部列出引用资料 ID。

新增笔记时：

- 中文为主，关键英文术语保留。
- 每篇尽量包含目的、核心概念、协议规则、示例、SoC 实践、常见 bug、verification checklist 和参考资料。
- 不确定的协议细节必须标注来源限制，不能把二手资料当作官方规则。
- 官方规格下载后，应更新 `metadata/sources.yaml` 和相关笔记参考资料。

## Git 状态

当前主干提交：

```text
a5ad561 initialize AMBA AXI knowledge base
```

当前默认分支：

```text
master
```

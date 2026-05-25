# Todo

## Sources

- [x] 收集 Arm 官方 AMBA overview 页面和资料入口。
- [x] 收集 AXI/ACE/CHI 官方规格入口和下载限制说明。
- [x] 收集 AMD/Xilinx AXI 相关公开文档入口。
- [ ] 替换 AMD/Xilinx UG1037 HTML 外壳为完整 PDF 或离线正文归档。
- [ ] 收集 EDA/IP 厂商关于 AXI verification、VIP、interconnect 的公开资料。
- [x] 收集高质量开源 AXI RTL 项目文档。
- [x] 收集 PULP AXI microarchitecture 论文。
- [x] 收集 cocotbext-axi 验证资料。
- [x] 收集 Xilinx AXI Interconnect product guide。
- [x] 收集 Xilinx AXI DMA product guide。
- [x] 收集 ZipCPU AXI/formal/bridge 相关开源资料。
- [x] 收集 taxi AXI components 开源资料。

## Source Gaps

- [ ] 手工下载或确认 Arm `IHI0022` AXI/ACE 官方规格 PDF。
- [ ] 手工下载或确认 Arm `IHI0051` CHI 官方规格 PDF。
- [ ] 补充 Arm AXI4-Stream 官方规格入口和下载说明。
- [ ] 补充 Synopsys/Cadence/Siemens 等厂商 AXI VIP 或 verification 公开资料。
- [ ] 补充 AMD/Xilinx 最新 AMD docs 门户中 UG1037 的完整离线版本。

## Notes

- [x] 完成 `00-overview.md` 第一版。
- [x] 完成 `01-amba-family.md` 第一版。
- [x] 完成 `02-axi-basics.md` 第一版。
- [x] 完成 `03-axi-channels.md` 第一版。
- [x] 完成 `04-valid-ready-handshake.md` 第一版。
- [x] 完成 `05-burst-transfer.md` 第一版。
- [x] 完成 `06-response-and-error.md` 第一版。
- [x] 完成 `07-outstanding-and-id.md` 第一版。
- [x] 完成 `08-ordering-model.md` 第一版。
- [x] 完成 `09-axi4-lite.md` 第一版。
- [x] 完成 `10-axi4-stream.md` 第一版。
- [x] 完成 `11-axi3-vs-axi4-vs-axi5.md` 第一版。
- [x] 完成 `12-interconnect-crossbar-noc.md` 第一版。
- [x] 完成 `13-bridges-and-converters.md` 第一版。
- [x] 完成 `14-clock-reset-cdc.md` 第一版。
- [x] 完成 `15-cache-coherency-ace-chi.md` 第一版。
- [x] 完成 `16-security-protection.md` 第一版。
- [x] 完成 `17-qos-performance.md` 第一版。
- [x] 完成 `18-verification.md` 第一版。
- [x] 完成 `19-common-bugs.md` 第一版。
- [x] 完成 `20-soc-practice.md` 第一版。

## Next Notes Improvements

- [ ] 为 `04-valid-ready-handshake.md` 添加可渲染时序图或 SVG。
- [ ] 为 `05-burst-transfer.md` 添加地址计算表格和更多 4KB boundary 示例。
- [ ] 为 `07-outstanding-and-id.md` 添加 ID remap 示例图。
- [ ] 为 `12-interconnect-crossbar-noc.md` 添加典型 SoC 拓扑图。
- [ ] 为 `18-verification.md` 添加 SystemVerilog assertion 示例。
- [ ] 为 `20-soc-practice.md` 添加完整 DMA bring-up 案例。
- [ ] 为 `04-valid-ready-handshake.md` 添加可渲染时序图或 SVG。

## Examples

- [ ] 添加 VALID/READY timing diagram。
- [ ] 添加 AXI4-Lite slave 示例说明。
- [ ] 添加 INCR burst transaction walkthrough。
- [ ] 添加 AXI-to-APB bridge 场景说明。

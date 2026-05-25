# Arm Official Source Notes

本文件记录 Arm 官方 AMBA/AXI 资料入口、自动抓取限制和后续人工下载说明。

## AMBA Architecture Page

- URL: https://developer.arm.com/Architectures/AMBA
- Reliability: primary
- Status: referenced
- Accessed: 2026-05-24
- Note: Arm Developer 页面通过 JavaScript 应用加载，自动抓取只能得到有限外壳内容。该入口仍作为官方 AMBA 资料索引来源保留。

## AMBA AXI and ACE Protocol Specification

- URL: https://developer.arm.com/documentation/ihi0022/latest
- Reliability: primary
- Status: access-limited-manual-download-required
- Accessed: 2026-05-24
- Note: 该文档是 AXI/ACE 协议规则的核心来源。已尝试自动抓取，Arm Developer 只返回动态页面外壳，无法得到完整规格正文。后续需要通过浏览器登录/接受协议后手工下载 PDF。

## AMBA CHI Specification

- URL: https://developer.arm.com/documentation/ihi0051/latest
- Reliability: primary
- Status: access-limited-manual-download-required
- Accessed: 2026-05-24
- Note: CHI 与 SoC cache coherency、ACE 演进关系密切。已尝试自动抓取，Arm Developer 只返回动态页面外壳，无法得到完整规格正文。后续需要通过浏览器登录/接受协议后手工下载 PDF。

## Use Policy

- 官方规格下载后保存到本目录，不覆盖该说明文件。
- 若存在多个版本，文件名应带版本号或发布日期。
- 笔记中涉及协议合法性判断时，优先引用官方规格。

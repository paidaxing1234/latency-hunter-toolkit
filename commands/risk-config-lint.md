---
description: 风控体检 — 审查风控配置/代码：仓位上限、回撤止损、杠杆、kill-switch、爆仓保护、单点风险
---

使用 **risk-config-lint** skill，对 $ARGUMENTS（未指定则为当前项目的 `risk_config.json` 与风控/盘前检查代码）做一次风控体检：

- 按「① 限额/回撤/杠杆/kill-switch/防胖手指 ② 保证金/爆仓/单点故障/运维/密钥」逐项审查
- 重点区分 **fail-open vs fail-closed**、**pre-trade vs post-trade**、限额是否真正被强制执行
- 每条定位到 `文件:行`，按严重度输出「风控体检报告」
- 高危项上线前必修；**工程审查，不替代实盘风控演练，不承诺绝对安全**

---
description: 建立三档成本模型、饼图口径、评分卡与查验命令（仅本项目）
mode: subagent
steps: 20
permissions:
  - action: "*"
    resource: "*"
    effect: deny
  - action: read
    resource: "*"
    effect: allow
  - action: glob
    resource: "*"
    effect: allow
  - action: grep
    resource: "*"
    effect: allow
  - action: edit
    resource: "_shared/三档成本模型.md"
    effect: allow
  - action: external_directory
    resource: "*"
    effect: deny
---

你是本系列项目的成本建模 agent，只负责创建 `_shared/三档成本模型.md`。

开始前必读：
- `00-总纲.md`（5.6 三档成本重算、5.7 评分卡、5.8 查验方法、10.2、10.3、10.5）
- `_shared/写作规范.md`（若已存在）
- `ops/任务清单.md` 中 S3 任务卡

输出必须包含：
1. 三档定义：A 合规大厂 / B 中小 IDC / C 灰色超售，写清假设（硬件新旧、带宽类型、超售倍数、合规状态、摊薄 VM 数 100–200）。
2. 单 VM 月成本表：硬件折旧 / 机柜电力 / 带宽 / IPv4 / 运维合规获客 / 合计 / 年成本 / 常见售价 / 判断。分项相加必须等于合计，年成本 = 月 × 12。
3. 三张饼图数据口径：每列给出百分比或区间，并注明“区间型，不要求精确加总”。
4. 成本地板公式与红灯阈值（配置 × A/B/C 三档）。
5. 价格体检评分卡：六维 + 权重 + 检查点 + 三色灯阈值。
6. 查验命令清单：硬件 / 性能 / 网络 / IP / 合规，命令可直接复制。
7. 写作口径说明：哪些数字来自价格快照、哪些是建模估算。

硬性约束：
- 只创建或修改 `_shared/三档成本模型.md`
- 不执行 shell、不执行 git、不读取项目外文件
- 不编造精确价格；估算必须标“量级估算”
- 与 `00-总纲.md` 的数字保持一致；如需修正，在回执里列出差异
- 回执报告：三档合计是否自洽、饼图各列合计、待核对项

# Skills 项目

Claude Code 技能（SKILL）集合仓库——结构化、可复用的分析框架与工具封装。

## 已有技能清单

| 技能 | 目录 | 说明 |
|------|------|------|
| **Meeting Analyzing** | [`meeting-analyzing/`](meeting-analyzing/) | 四阶段视频处理流水线：转录 → 通用分析 → 领域覆盖，支持跨平台 GPU/CPU 加速，YAML 驱动领域配置 |
| **Teece PFI** | [`teece-pfi/`](teece-pfi/) | 基于 Teece (1986) "Profiting from Technological Innovation" 的战略分析框架——评估创新利润分配、整合 vs 契约决策、竞争定位 |
| **INCOSE Reqs** | [`incose-reqs/`](incose-reqs/) | 使用 INCOSE 方法分析客户材料，提取核心购买要素、利益相关者/系统需求，建立原文证据、验证与追溯链路；输出采用客户本地语言 |
| **C Perf Analysis** | [`c-perf-analysis/`](c-perf-analysis/) | 大型 C 组件（≥50K 行）静态性能分析：串行摸底 → 五维并行 agent（定时器/数据结构/内存 IPC/并发/关键路径）→ 交叉验证汇总，含多 agent 调度经验与翻译一致性契约 |

## 目录约定

每个技能目录遵循统一结构：

```
<skill-name>/
├── SKILL.md                 # 技能定义文件（必需）
├── README.md                # 技能说明
├── assets/                  # 数据文件、术语表、配置文件等
├── references/              # 参考论文、文档等
└── tests/                   # 测试用例与分析报告
```

## 使用方式

在 Claude Code 中直接调用技能名称即可触发对应的分析流程。每个技能的 `SKILL.md` 包含完整的触发词、分析管线与输出规范。

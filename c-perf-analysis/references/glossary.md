# Glossary & Contract Data (bilingual by design)

This file is translation-contract DATA consumed by Phase 4 translation agents and by
the annotation/severity vocabulary — the CN entries are data, not instruction prose;
do not translate them away.

## Annotation tags

| Tag | Meaning |
|---|---|
| `[verified: file:line]` | read and confirmed in source |
| `[inferred]` / `[推断]` | derived estimate; show arithmetic or reasoning |
| `[external, not read]` / `[外部，未读取]` | definition/implementation outside allowed scope |
| `[not analyzed]` / `[未分析]` | declared gap — never silently dropped |
| `[jemalloc]` (or `[allocator-name]`) | third-party behavior assumption, not source-verified |

## EN↔CN terminology

> Bilingual by design: this table is translation-contract DATA consumed by Phase 4
> translation agents, not skill instruction prose. The CN column must stay Chinese.

| EN | CN | Note |
|---|---|---|
| size class | 尺寸类 | |
| slab / bin / tcache | 保留英文 | allocator jargon |
| internal / external fragmentation | 内碎片 / 外碎片 | |
| watermark | 水位 | |
| backpressure | 反压 | |
| interning / interned | 驻留（去重共享） | |
| pseudo-thread | 伪线程 | scheduler-based coroutine |
| churn | 震荡/抖动 | |
| drain rate / reaper | 排空速率 / 回收器 | |
| critical section | 临界区 | |
| head-of-line blocking | 队头阻塞 | |
| metric definition / 口径 | 口径 | reconciliation context |
| retention cap | 保留上限 | pool semantics |
| bounded instance | 有界实例 | clamped scenario |
| wall-clock | 墙钟时间 | vs CPU time |
| cache miss | cache miss | keep EN, industry-standard usage |

# 阶段三报告：目标项目侧接入

状态：已完成
日期：2026-08-19
分支：`master`（archon-test 仓库）、`codex/complexity-reduction-integration`（Archon 仓库）

## 概述

阶段三的目标是把 `archon-test` 目标项目接入 ComplexityReduction Lean 库，
并激活阶段二在 Archon 源码侧添加的 prover mode、subagent 描述符、技能参考
和搜索脚本。共完成 5 个子步骤 + 2 个附带修复，涉及 6 个新增/修改文件
（git tracked）和 4 个 `.archon/` 配置文件（gitignored）。

## 完成步骤

### 3.1 Lakefile 依赖

在 `lakefile.lean` 中添加 `require ComplexityReduction from "../ComplexityReduction"`。
`lake update` 和 `lake build ArchonTest` 均通过。

### 3.2 冒烟导入

创建 `ArchonTest/ComplexityReductionSmoke.lean`，对 `ComplexityReduction.Targets`
的四个关键符号执行 `#check`：

- `NPCTargetDescriptor`（类型）
- `threeSATDecisionProblem`（值）
- `threeSATTarget`（描述符）
- `localTargets`（列表）

构建通过，全部符号可见。同时添加 `#print axioms` 检查，验证 axiom-clean
（详见验证清单第 8 项）。

### 3.3 库参考文档

创建 `references/ComplexityReduction.md`，记录库的模块布局、关键类型和搜索
工作流。更新 `references/summary.md` 添加新参考条目。

### 3.4 持久化用户提示

在 `.archon/USER_HINTS.md` 中添加四条持久化提示：

1. 证明某问题 NP 完全前，先用搜索脚本或参考文档查 ComplexityReduction 是否已有
   对应编码、归约或目标描述符。
2. 新目标必须从 `ComplexityReduction.Targets.localTargets` 选择对应描述符，
   不要凭空构造 NPC 目标。
3. 外部 Schaefer bridge 子模块（`Targets.Schaefer`、`Targets.SchaeferFiniteDomain`）
   默认不用于本地推理，除非显式 opt-in。
4. 每个产物必须 axiom-clean，禁止 `sorry`；`#print axioms` 只允许 `propext`、
   `Classical.choice`、`Quot.sound`。

### 3.5 子代理描述符与配置

创建 `.archon/subagents/complexity-reduction-advisor.md` — 一个只读、不可派生
的子代理，在证明开始前检查 CR 库中是否有可复用的声明。

在 `.archon/config.json` 中启用：

- 添加到 `subagents.enabled`（从 `[]` 改为 `["complexity-reduction-advisor"]`）
- 添加到 `subagents._available`
- 添加到 `subagents._recommended_plan_phase`

### 构建模式补全

将 `complexity-reduction-build.md` 从 Archon 源码复制到目标项目的
`.archon/prover-modes/`（目标项目在阶段二添加此文件之前就已初始化，因此缺失）。

## 验证清单

| # | 检查项 | 结果 |
|---|--------|------|
| 1 | `lake build complexityreduction` | 通过（7950 个构件，退出码 0） |
| 2 | `lake build ArchonTest` | 通过 |
| 3 | 冒烟文件 `#check` 符号 | 通过（4/4 可见） |
| 4 | `archon doctor` 识别打包数据 | 通过（全部 OK） |
| 5 | 构建模式已复制到目标项目 | 通过 |
| 6 | 子代理被注册表加载 | 通过 |
| 7 | `archon loop --dry-run` | 通过（advisor 已启用，loop 正常启动） |
| 8 | `#print axioms` 声明检查 | 通过（axiom-clean） |

### Axiom-clean 详情

```text
'NPCTargetDescriptor' does not depend on any axioms
'threeSATDecisionProblem' depends on axioms: [propext, Classical.choice, Quot.sound]
'threeSATTarget' does not depend on any axioms
'localTargets' does not depend on any axioms
```

仅出现 Mathlib 三个标准公理，符合 USER_HINTS 中的策略要求。

### Dry-Run 输出

```text
Subagents: 1 of 16 enabled: complexity-reduction-advisor
```

Loop 正确识别 advisor 为唯一启用的子代理。

## 附带修复

修复了两个预存的 Windows 编码 bug，这两个 bug 阻塞了 `archon doctor` 和
`archon loop` 在 GBK 区域设置系统上的运行：

- `src/archon/commands/doctor.py`：`_parse_stage` 读取 PROGRESS.md 时
  添加 `encoding="utf-8"`。
- `src/archon/state/progress.py`：`read_stage` 读取 PROGRESS.md 时
  添加 `encoding="utf-8"`。

同时创建了 `scripts/verify_phase3_integration.py` 验证脚本，检查 prover-mode
目录、模式体加载、子代理注册表和 config.json 一致性，4/4 项通过。

## 变更文件（archon-test）

| 文件 | 状态 | 说明 |
|------|------|------|
| `lakefile.lean` | 修改 | 添加 CR 依赖 |
| `lake-manifest.json` | 修改 | `lake update` 自动更新 |
| `ArchonTest/ComplexityReductionSmoke.lean` | 新增 | 冒烟导入 + axiom 检查 |
| `references/ComplexityReduction.md` | 新增 | 库参考文档 |
| `references/summary.md` | 修改 | 添加 CR 参考条目 |
| `.archon/subagents/complexity-reduction-advisor.md` | 新增（gitignored） | 子代理描述符 |
| `.archon/prover-modes/complexity-reduction-build.md` | 新增（gitignored） | 构建模式 |
| `.archon/USER_HINTS.md` | 修改（gitignored） | 持久化提示 |
| `.archon/config.json` | 修改（gitignored） | 启用子代理 |

## 提交记录

| 仓库 | 提交 | 内容 |
|------|------|------|
| archon-test | `efd2230` | 阶段三集成（lakefile、冒烟文件、参考文档、报告） |
| archon-test | `52e7ed5` | 将 PHASE3_REPORT.md 移至项目根目录 |
| Archon | `8925c36` | 编码修复 + Phase 3 验证脚本 |

## 状态

阶段三已完成。全部 8 项验证通过。变更已提交到本地，未推送。

# 阶段二报告：Archon 源码侧改动

状态：已完成（含收尾修正）
日期：2026-08-19
分支：`codex/complexity-reduction-integration`（Archon 仓库）

## 概述

阶段二的目标是把 ComplexityReduction 作为"一等知识"固化进 Archon 的打包数据。
共完成 4 项改动，涉及 4 个新增文件和 5 个修改文件，分布在 Archon 的 skill 文档、
搜索脚本、prover-mode 和 Python 代码四个层面。

## 提交记录

| 提交 | 内容 | 文件 |
| --- | --- | --- |
| `964c315` | 2.1 库参考文档 | `references/complexity-reduction-library.md`（新增），`SKILL.md`（修改） |
| `33cf065` | 2.3 搜索脚本 | `lib/scripts/search_complexity_reduction.sh`（新增） |
| `34aa493` | 2.2 构建模式 | `prover-modes/complexity-reduction-build.md`（新增） |
| `8401e80` | 2.4 Lake 依赖提示 | `lake.py`、`project.py`、`semantic_pass.py`（均修改） |
| `e573680` | 2.4 Lake 依赖修正 | `lake.py`（修改），`test_init_mathlib_no_update.py`（修改），`test_complexity_reduction_lake.py`（新增） |

## 收尾修正

`8401e80` 中的 `add_complexity_reduction_dependency()` 在 2.4 收尾阶段被发现有两个需要修正的问题：

- Lean 格式错误地生成 `from Path "..."`，不符合 Lake 4.29 语法，改为 `from "..."`。
- 依赖名错误地使用小写 `complexityreduction`，改为标准包名 `ComplexityReduction`。

同次提交给新的检测/写入逻辑补上回归测试，并修复受 `LakefileInfo` 新字段影响的既有测试。

## 2.1 库参考文档

**文件：** `src/archon/.archon-src/skills/lean4/skills/lean4/references/complexity-reduction-library.md`

覆盖内容：

- **稳定导入面** — 8 个聚合模块（Core/SAT/CSP/Schaefer/Karp21/Bridges/Certificates/Targets）+ 根导入 `import ComplexityReduction`，以及独立模块 `ComplexityReduction.Templates`。
- **符号表** — 7 个模块的公开符号，全部经 `#check` 验证。标注了 `Bridges` 和 `Certificates` 的声明不在同名命名空间下（如 `CostedTMSound` 直接在 `ComplexityReduction` 下，没有 `Bridges.CostedTMSound`）。
- **Targets 目录与信任边界** — 默认可用的 7 个子模块 vs 需要 explicit opt-in 的 2 个 Schaefer bridge 子模块（`Targets.Schaefer`、`Targets.SchaeferFiniteDomain`），附模块名歧义原因说明。
- **Templates 使用场景** — 8 个模板子模块的对照表（Lists/FormulaAppend/Renaming/Substitution/Blocks/Structural/StrictLists/SubsetLifting）。
- **构建与验证** — `lake build` 命令、axiom-clean 检查、双嵌套目录结构说明。路径用通用描述，不含硬编码绝对路径。

在 `SKILL.md` 的 References 部分新增 **Libraries** 分类索引。

## 2.3 搜索脚本

**文件：** `src/archon/.archon-src/skills/lean4/lib/scripts/search_complexity_reduction.sh`

参考 `search_mathlib.sh` 结构，支持三种搜索模式：

- `name` — 按声明名搜索（`^(theorem|lemma|def|class|structure|inductive|abbrev).*$QUERY`）
- `type` — 按类型签名搜索（`: .*$QUERY`）
- `content` — 按文件内容搜索（全文匹配）

配置：

- 通过 `COMPLEXITY_REDUCTION_PATH` 环境变量指定 CR 源码路径，默认 `../ComplexityReduction/ComplexityReduction`（内层目录）。
- 优先使用 `rg`，不可用时自动退回 `grep`。

验证：用 Git Bash 对真实 CR 源码测试三种模式全部通过。

## 2.2 构建模式

**文件：** `src/archon/.archon-src/prover-modes/complexity-reduction-build.md`

Frontmatter 对齐 `mathlib-build.md`（`name`/`description`/`compatible_stages`/`read_blueprint`/`dispatcher_notes`）。

正文核心要求：

1. **先查 CR 既有声明** — 第 2 节引导使用搜索脚本 + 参考文档符号表，列出 5 个自查问题（源问题是否已有编码、目标是否在 Targets 中、是否有可组合的 CostedReduction、是否有匹配模板、transfer 定理能否闭合）。
2. **优先用归约/模板/bridge** — 第 4 节明确 `CostedReduction.comp`、`Templates.*`、`NPCompleteEnc.transfer`、`composeTMKarp` 的优先级，避免从零证明。
3. **每声明 axiom-clean** — Invariant 节 + 第 3 节验证步骤，`#print axioms` 只允许 `{propext, Classical.choice, Quot.sound}`。
4. **停止前给出可继续的分解** — 第 7 节附具体示例，要求给出精确的声明类型和下一步方法。

第 5 节 Targets 信任边界与 2.1 参考文档呼应，提醒 Schaefer bridge 需 explicit opt-in。

## 2.4 Lake 依赖提示

**文件：** `src/archon/commands/tooling/lake.py`、`src/archon/commands/tooling/project.py`、`src/archon/commands/init/steps/semantic_pass.py`

### lake.py

- `LakefileInfo` 新增 `has_complexity_reduction: bool` 字段。
- `lakefile_info()` 通过正则检测 CR 依赖：
  - lean 格式：`require\s+[Cc]omplexity[Rr]eduction\b`
  - toml 格式：`name\s*=\s*"[Cc]omplexity[Rr]eduction"`
- 新增 `add_complexity_reduction_dependency(source_path, if_absent=True)` 方法，支持本地路径依赖的 lean/toml 两种格式写入。Lean 格式使用 `require ComplexityReduction from "../ComplexityReduction"`，TOML 使用 `name = "ComplexityReduction"` + `path = "../ComplexityReduction"`。

### project.py

- `ProjectLayout` 新增 `has_complexity_reduction: bool` 字段。
- `inspect()` 从 `LakefileInfo` 透传该字段。

### semantic_pass.py

- bootstrap summary 的 layout dict 新增 `has_complexity_reduction` 键，供 Claude 在 init 阶段感知 CR 依赖状态。

### 验证

用 Archon venv 测试：
- ComplexityReduction 仓库：`has_mathlib=True, has_cr=True`（toml 格式）
- archon-test 项目：`has_mathlib=True, has_cr=False`（lean 格式）

检测逻辑大小写兼容；`tests/test_complexity_reduction_lake.py` 与 `tests/test_init_mathlib_no_update.py` 合计 18 个测试全部通过，覆盖 lean/toml 检测、写入语法、幂等性与无 lakefile 报错。

### 设计决策

按计划"先加 CR 专用识别，后续再泛化为 `declared_dependencies` 列表"，这一步只做识别和展示，不接入 bootstrap 自动添加 CR 依赖的流程。`add_complexity_reduction_dependency()` 方法已就绪，阶段三目标项目接入时可直接调用。

## 已知限制

- `add_complexity_reduction_dependency()` 尚未接入 `BootstrapOptions` 和 `_ensure_*` 流程，阶段三会用到。
- 搜索脚本在 WSL bash 环境下因 `rg` 路径问题不可用（Codex 内置 rg 无权限），需用 Git Bash 或原生 Linux/macOS。这是测试环境限制，不影响脚本本身。
- `semantic_pass.py` 的 `has_complexity_reduction` 仅展示到 bootstrap summary，尚未在 `doctor` 命令中添加版本不匹配或缺依赖的提示。后续可补。

## 阶段二完成，可进入阶段三（目标项目侧接入）

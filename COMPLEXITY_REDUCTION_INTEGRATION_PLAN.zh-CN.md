# ComplexityReduction 与 Archon 适配计划

## 背景

目标是让 Archon 在需要 ComplexityReduction 库的形式化任务中，不只是依赖 agent 自行阅读源码，而是拥有明确的依赖识别、符号参考、证明模式和搜索手段。

当前目录布局：

- `D:\code\Lean\Archon`：Archon 主源码。
- `D:\code\Lean\archon-test`：现有 Archon 目标项目。
- `D:\code\Lean\ComplexityReduction`：Lean 4 计算复杂度归约库。

`D:\code\Lean\scripts` 已由用户清空。它与 Archon 自身的
`src/archon/.archon-src/scripts` 无关，不需要做任何迁移。

## 关键约定

- ComplexityReduction 不复制进 Archon 仓库。
- ComplexityReduction 不使用 Archon `peers` 机制，因为它不是 Archon 项目，而是 Lake 依赖库。
- 目标项目与 ComplexityReduction 必须保持 `leanprover/lean4:v4.29.0`，mathlib 锁定 `v4.29.0`。
- 所有落库内容默认拒绝 `sorry`，并通过 `#print axioms` 检查，避免引入 `sorryAx`。

## 阶段一：冻结并验证依赖边界

1. 确认 ComplexityReduction 与目标项目的 lean-toolchain 一致。
2. 确认 ComplexityReduction 与目标项目的 mathlib 版本一致。
3. 在 ComplexityReduction 中维护一个稳定的 Archon 导入面，建议覆盖：
   - `Core`
   - `SAT`
   - `CSP`
   - `Schaefer`
   - `Karp21`
   - `Bridges`
   - `Certificates`
   - `Targets`
4. 验证：
   - `lake build complexityreduction`
   - 最小冒烟文件 `#check` 关键公开符号
   - 记录哪些 `Targets` 默认可用，哪些需要 explicit opt-in。

## 阶段二：Archon 源码侧改动

目标是把 ComplexityReduction 作为一等知识固化到 Archon 的打包数据中。

### 2.1 增加库参考文档

新增或维护：

```text
src/archon/.archon-src/skills/lean4/skills/lean4/references/complexity-reduction-library.md
```

内容至少包含：

- 稳定导入面。
- 核心定义和定理符号表。
- `Targets` 目录中的可用目标与信任边界。
- 外部 Schaefer bridge 的 explicit opt-in 规则。
- 常见模板 `Templates` 的使用场景。

### 2.2 增加 ComplexityReduction 构建模式

新增：

```text
src/archon/.archon-src/prover-modes/complexity-reduction-build.md
```

frontmatter 与现有 `mathlib-build.md` 保持一致，正文要求：

- 先查 ComplexityReduction 是否已有对应定义和定理。
- 优先通过归约、模板和 bridge 完成目标。
- 每个新声明都必须编译通过且 axiom-clean。
- 停止前必须给出下一次可继续的分解。

### 2.3 增加 ComplexityReduction 搜索脚本

新增：

```text
src/archon/.archon-src/skills/lean4/lib/scripts/search_complexity_reduction.sh
```

功能参考 `search_mathlib.sh`，支持：

- `name`：按声明名搜索。
- `type`：按类型签名搜索。
- `content`：按文件内容搜索。

搜索根默认指向 ComplexityReduction 源码位置，并允许通过环境变量覆盖。

### 2.4 扩展 Lake 依赖提示

当前 [lake.py](src/archon/commands/tooling/lake.py) 只通过 `has_mathlib: Bool` 记录 mathlib。

建议分两步：

1. 先增加 `has_complexity_reduction` 或等价的 CR 专用识别。
2. 后续再考虑泛化为 `declared_dependencies` 列表，统一支持自定义 Lean 依赖。

同步更新：

- `ProjectLayout` 与 `semantic_pass` 对依赖状态的展示。
- `init` 或 `doctor` 中对版本不匹配和缺依赖的提示。

## 阶段三：目标项目侧接入

以 `D:\code\Lean\archon-test` 为默认目标项目。

### 3.1 添加 Lake 本地依赖

在目标项目 lakefile 中增加指向 `../ComplexityReduction` 的本地依赖，保持现有 mathlib 不变。

同步确认：

- 工具链仍为 `v4.29.0`。
- 本地依赖不会引起重复的 mathlib 解析。

### 3.2 导入 ComplexityReduction

在目标项目的对应 Lean 文件中导入稳定的 CR 入口面，例如：

```lean
import ComplexityReduction.Targets
```

如果需要更宽的符号面，再按需导入 `Core`、`SAT`、`CSP`、`Schaefer` 或 `Karp21`。

### 3.3 增加项目内参考文献

新增：

```text
D:\code\Lean\archon-test\references\ComplexityReduction.md
```

与 Archon 打包数据中的库参考保持一致，并记录目标项目当前需要的具体目标。

### 3.4 增加持久用户提示

在目标项目的 `.archon/USER_HINTS.md` 持久区加入：

- 优先使用 ComplexityReduction 已有声明。
- 新目标必须从 `Targets` 目录选择对应描述符。
- 外部 Schaefer bridge 默认不用于本地推理，除非显式 opt-in。
- 每个产物必须 axiom-clean，禁止 `sorry`。

### 3.5 配置 ComplexityReduction 顾问子代理

新增：

```text
D:\code\Lean\archon-test\.archon\subagents\complexity-reduction-advisor.md
```

frontmatter 示例：

```yaml
---
name: complexity-reduction-advisor
description: "Check ComplexityReduction API, target descriptors, and proof template fit."
read_only: true
mandatory: plan
---
```

然后在 `.archon/config.json` 的 `subagents.enabled` 中加入：

```json
"enabled": ["complexity-reduction-advisor"]
```

如希望该项目每次计划阶段都使用它，可再将其加入 `_recommended_plan_phase`。

## 验证清单

1. `lake build complexityreduction` 通过。
2. `lake build ArchonTest` 通过，确认本地依赖已解析。
3. 冒烟文件可以 `#check` ComplexityReduction 关键符号。
4. `archon init` 或 `archon doctor` 能发现新打包数据。
5. `.archon/prover-modes/complexity-reduction-build.md` 被复制到目标项目。
6. `complexity-reduction-advisor` 被 Registry 正常加载。
7. 在 `PROGRESS.md` 放一个最小目标，用 `complexity-reduction-build` 模式跑一轮 `archon loop`。
8. 最终对新增声明执行 `#print axioms`，确认无 `sorryAx`。

## 暂不执行的事项

- 不把 ComplexityReduction 源码复制进 Archon。
- 不把 ComplexityReduction 注册为 Archon peer。
- 不在第一轮就做完整的通用依赖配置系统，CR 专用识别足够后再逐步泛化。

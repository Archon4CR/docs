# Archon + ComplexityReduction 归约问题测试方案

## 目标

使用 `problems/problems_clean.json` 中整理的 NP 完全性归约问题集，
测试修改后的 Archon（集成 CR advisor 子代理、搜索脚本、参考文档、prover-mode）
在归约证明上的表现。

## 路线选择：路线 A（sorry stub + 标准 loop + CR advisor）

### 为什么选路线 A

| 维度 | 路线 A | 路线 B (complexity-reduction-build) |
|------|--------|-------------------------------------|
| 测试对象 | CR 知识是否帮助标准证明流程 | CR-build prover-mode 本身 |
| 预处理 | 需要写 blueprint + 生成 stub | 只需 PROGRESS.md 描述目标 |
| 证明目标 | 填充 sorry stub（闭合已有声明） | 从零构建 CostedReduction 证书 |
| 与 CR 的关系 | advisor 辅助查找已有编码/模板 | mode 本身就是在构建 CR 设施 |
| 适用场景 | 验证集成效果（本次目标） | 验证 build mode 的从零构建能力 |

路线 A 测试的是"CR 知识注入是否提升了 prover 的表现"，
这正是我们三阶段集成工作的核心目标。

### 不选路线 B 的原因（本次）

`complexity-reduction-build` 模式要求不输出 sorry，
因为它构建的是归约基础设施（`CostedReduction` 证书），而非闭合已有 stub。
一个带 sorry 的 `CostedReduction` 作为证书是毫无意义的。
该模式测试的是从零构建能力，与本次"测试 CR 集成效果"的目标不完全对齐。

---

## 完整工作流

```
problems_clean.json (自然语言题目)
        |
        v
  1. 选题（3-5 题，混合难度）
        |
        v
  2. 写 blueprint .tex 章节（半形式化 LaTeX + \lean{} 标注）
        |
        v
  3. Archon formalize 模式 -> 生成 Lean sorry stub（编译通过）
        |
        v
  4. Archon prove 模式 + CR advisor -> 填充证明
        |
        v
  5. 检查 dispatch.jsonl / task_results 确认 CR 设施起效
```

### 第 1 步：选题

从 `problems_clean.json`（63 题）中选取 3-5 题，优先选择：
- CR 库已有相关编码的问题（3SAT 系列）
- 混合难度（easy / medium / hard）
- 有完整 solution 的问题（便于验证正确性）

**初选清单：**

| ID | 归约 | 难度 | 有解 | 选择理由 |
|----|------|------|------|----------|
| 01-001 | SAT -> DOUBLE-SAT | easy | yes | 最简单的入门题，CR 有 SAT 编码 |
| 02-004 | 3SAT -> NAE-3SAT | hard | yes | 经典归约，CR 有 3SAT 编码 |
| 06-017 | CLIQUE -> INDEPENDENT SET | -- | yes | 图论归约，解法简洁（补图） |

可扩展候选：
- 06-018: INDEPENDENT SET -> VERTEX COVER（与 06-017 形成链）
- 02-007: VERTEX COVER -> SET COVER（经典覆盖问题）

### 第 2 步：写 blueprint .tex

为每道题写一个 blueprint 章节（LaTeX），包含：
- `\section{...}` 章节标题
- `\begin{theorem}...\end{theorem}` 定理块，内含 `\lean{declName}` 标注
- `\begin{proof}...\end{proof}` 证明草稿

文件命名规则：`blueprint/src/chapters/ReductionTest_<Name>.tex`
（对应 Lean 文件 `ReductionTest/<Name>.lean`）

blueprint 中需要意识到 CR 库的类型体系：
- `EncodedDecisionProblem` -- 基本决策问题类型
- `CostedReduction A B` -- 归约证书（含 `f`、`correct`、`costed`）
- `SAT.threeSATDecisionProblem` -- 已有的 3SAT 编码

### 第 3 步：生成 Lean sorry stub

两种方式：

**方式一（推荐）：交给 Archon formalize 模式**

Archon 的 `formalize` prover-mode 专门为此设计：
- 读 blueprint `.tex` 章节
- 为每个 theorem/lemma/definition 块生成带 `sorry` 的 Lean 声明
- 验证文件编译通过

只需在 PROGRESS.md 中将阶段设为 `autoformalize`，
formalize 模式会自动从 blueprint 生成 stub。

**方式二：手动写 stub**

直接在 `.lean` 文件中写声明签名 + `sorry`。

推荐方式一，因为它同时测试了 Archon 的 formalize 能力。

### 第 4 步：运行 Archon prove + CR advisor

在 `.archon/config.json` 中：
- `subagents.enabled: ["complexity-reduction-advisor"]`
- `harness: "codex"`（或用户偏好的 harness）
- prover 阶段自动使用 `prove` 模式

运行命令：
```powershell
archon loop .
```

CR advisor 会在 plan 阶段被派发（参见下文"CR 设施验证"），
检查 CR 库中是否有可复用的编码/模板/定理，输出报告供 planner 参考。

### 第 5 步：验证 CR 设施是否起效

详见下文"CR 设施验证方案"。

---

## 项目结构

```
reduction-test/
|-- lakefile.lean              # require ComplexityReduction from "../ComplexityReduction"
|-- lean-toolchain             # leanprover/lean4:v4.29.0
|-- ReductionTest.lean         # 根模块 import
|-- ReductionTest/
|   |-- DoubleSAT.lean         # SAT -> DOUBLE-SAT 归约
|   |-- NAE3SAT.lean           # 3SAT -> NAE-3SAT 归约
|   `-- IndependentSet.lean    # CLIQUE -> INDEPENDENT SET 归约
|-- blueprint/
|   `-- src/
|       |-- content.tex        # \input 各章节
|       |-- chapters/
|       |   |-- ReductionTest_DoubleSAT.tex
|       |   |-- ReductionTest_NAE3SAT.tex
|       `-- ReductionTest_IndependentSet.tex
|       `-- macros/
|           `-- common.tex
|-- references/
|   `-- ComplexityReduction.md # CR 库参考文档（从 archon-test 复制）
|-- .archon/
|   |-- config.json            # harness: codex, CR advisor enabled
|   |-- subagents/
|   |   `-- complexity-reduction-advisor.md
|   |-- prover-modes/
|   |   |-- prove.md
|   |   `-- formalize.md
|   |-- AGENTS.md
|   |-- PROGRESS.md
|   `-- ...
`-- scripts/                   # 临时脚本（按 AGENTS.md 约定）
```

---

## CR 设施验证方案

### 信号 1：dispatch.jsonl（最硬证据）

每次子代理被调度时，`Subagent.run()` 会向
`.archon/logs/iter-NNN/dispatch.jsonl` 追加 JSON 行：

```json
{"event":"dispatch_start","role":"complexity-reduction-advisor","slug":"...","write_domain":[...]}
{"event":"dispatch_end","role":"complexity-reduction-advisor","slug":"...","ok":true,"duration_s":42}
```

检查方法：
```powershell
Get-Content .archon\logs\iter-001\dispatch.jsonl |
  Where-Object { $_ -match "complexity-reduction-advisor" }
```

如果这里有记录，说明 planner 确实派发了 CR advisor。

### 信号 2：task_results 报告（内容证据）

advisor 的实际输出在
`.archon/task_results/complexity-reduction-advisor-<slug>.md`，
包含 verdict（REUSE / PARTIAL / BUILD_NEW）和具体发现。

### 信号 3：advice 持久建议

`.archon/advice/<slug>.md`，跨迭代持久，供后续迭代参考。

### 信号 4：mandatory 审计（强制保障）

`complexity-reduction-advisor.md` 的 frontmatter 目前**没有** `mandatory` 字段，
审计不会对它的缺席发出警告。

**建议改进**：给 advisor frontmatter 加上 `mandatory: ["plan"]`，
这样 `check_mandatory_dispatched` 会在 `dashboard.log` 中打印警告，
并要求 planner 在 `iter/iter-NNN/plan.md` 的 `## Subagent skips` 段落
提供跳过理由。

### 信号 5：plan 侧车引用

检查 `iter/iter-NNN/plan.md` 是否引用了 advisor 的发现。

### 实际验证流程

每次迭代后：
1. 查 `dispatch.jsonl` -- 确认 advisor 被派发
2. 查 `task_results/` -- 确认 advisor 输出了有意义的内容
3. 查 `plan.md` -- 确认 planner 消费了 advisor 报告
4. （可选）查 `prove` 的 `task_results` -- 确认 prover 使用了 CR 编码

---

## 从自然语言生成 Lean stub：能否交给 Archon？

可以。Archon 的 `formalize` prover-mode 专门为此设计。

### formalize 模式工作流

1. 读 blueprint `.tex` 章节（含 `\lean{}` 标注）
2. 为每个 `\begin{theorem}` / `\begin{lemma}` / `\begin{definition}` 块
   生成带 `sorry` 的 Lean 声明
3. 验证文件编译通过（所有 sorry 在位）
4. 输出到 `task_results/<file>.md`

### 分工

| 步骤 | 谁做 | 产出 |
|------|------|------|
| 选题 + 写 blueprint .tex | 我们 | LaTeX 章节，含 `\lean{declName}` 标注 |
| 生成 Lean sorry stub | Archon `formalize` 模式 | 编译通过的 `.lean` 文件 |
| 填充证明 | Archon `prove` 模式 + CR advisor | 完整证明 |

### 前提条件

- blueprint 中的问题描述需要意识到 CR 库的类型体系
  （`CostedReduction`、`EncodedDecisionProblem` 等）
- `\lean{}` 标注需要给出声明名（由我们决定命名）

### 额外资源

`lean-scaffolder` 子代理功能类似但由 planner 主动派发，
会注入策略注释（`/- Planner strategy: ... -/`）。
可作为补充。

---

## 待办事项

- [ ] 创建 `reduction-test` 项目（lakefile.lean、ReductionTest.lean、lean-toolchain）
- [ ] 写 3 道题的 blueprint .tex 章节
- [ ] 配置 `.archon/`（config.json、subagents、prover-modes、AGENTS.md）
- [ ] 复制 references/ComplexityReduction.md
- [ ] 给 CR advisor frontmatter 加 `mandatory: ["plan"]`
- [ ] 运行 `archon init .` 初始化项目状态
- [ ] 运行 `archon loop .` 开始测试
- [ ] 迭代后检查 CR 设施验证信号

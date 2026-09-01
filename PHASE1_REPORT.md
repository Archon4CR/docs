# 阶段一报告：冻结并验证依赖边界

状态：已完成
日期：2026-08-17
分支：`codex/archon-integration-entrypoints`

## 1. 工具链与依赖版本一致性

| 项目 | 值 |
| --- | --- |
| `lean-toolchain` | `leanprover/lean4:v4.29.0` |
| mathlib `rev`（lake-manifest） | `8a178386ffc0f5fef0b77738bb5449d50efeea95`（对应 `v4.29.0` 标签） |
| `lakefile.toml` `require.mathlib.rev` | `v4.29.0` |
| `defaultTargets` | `["complexityreduction"]` |

结论：ComplexityReduction 的 lean-toolchain 与 mathlib 版本自洽，均为 `v4.29.0`。
目标项目（`archon-test`）接入前需保持同一 toolchain 与 mathlib 锁定版本，避免重复解析。

## 2. 稳定的 Archon 导入面

根模块 `ComplexityReduction.lean` 已聚合以下 8 个入口面，目标项目只需
`import ComplexityReduction` 即可获得全部公开符号：

| 入口面 | 聚合模块 | 子模块数 | 用途 |
| --- | --- | --- | --- |
| `Core` | `ComplexityReduction.Core` | — | `EncodedDecisionProblem` 等基础类型 |
| `SAT` | `ComplexityReduction.SAT` | 134 导入 | 3SAT 判定问题、Cook-Levin 包装 |
| `CSP` | `ComplexityReduction.CSP` | 88 导入 | 布尔 CSP 公式、约束、编码 |
| `Schaefer` | `ComplexityReduction.Schaefer` | 32 导入 | Schaefer 二分法定理、可处理条件 |
| `Karp21` | `ComplexityReduction.Karp21` | 318 导入 + 定义 | 21 个 Karp NPC 问题枚举与描述符 |
| `Bridges` | `ComplexityReduction.Bridges` | 57 导入 | Costed→TM 桥接、可靠性证明 |
| `Certificates` | `ComplexityReduction.Certificates` | 5 导入 | 归约证书（costed / log-space） |
| `Targets` | `ComplexityReduction.Targets` | 11 导入 | NPC 目标目录与描述符 |

注意事项：

- `Karp21.lean` 既是聚合模块也承载 `Problem` / `ProblemDescriptor` 等 21 项定义，
  共 744 行（320 行导入 + 424 行定义）。
- `Bridges` 与 `Certificates` 的实际声明并非位于同名 namespace，而是直接位于
  `ComplexityReduction` 命名空间下（如 `CostedTMSound`、`CostedReduction`）。
  Archon 参考文档中需按真实全限定名给出符号表。
- `PackagedEdgeTransfers.lean` 曾因 `import ComplexityReduction.Karp21` 形成循环
  （`Karp21 → PackagedEdgeTransfers → Karp21`），已改为 44 条直接子模块导入，
  循环已消除，全量构建通过。

## 3. 构建验证

- `lake build ComplexityReduction`：全量 3982 个 job 编译成功，仅余 warning
  （未用 simp 参数、`simpa` vs `simp`、未用变量），无 error。
- 历史修复：
  - `ClauseFold.lean` kernel 深递归：拆分为 4 个 private 引理并用 `omega` 闭合
    （commit `5b8c705`）。
  - 聚合模块创建与循环导入修复（commit `b9dcb2b`）。

## 4. 冒烟测试

冒烟文件位于 `scripts/smoke_test.lean`（`scripts/` 在 `.gitignore` 中，不落库），
通过 `import ComplexityReduction` 对 8 个入口面的代表性公开符号执行 `#check`：

```text
Karp21.Problem                                      : Type
Karp21.ProblemDescriptor                            : Type
Karp21.problemDescriptors                           : List Karp21.ProblemDescriptor
Karp21.Problem.all                                  : List Karp21.Problem
(Karp21.Problem.all.length = 21)                    : Prop
SAT.threeSATDecisionProblem                         : EncodedDecisionProblem
SAT.TMNPCompleteEncWithCheckedSuffixValidity        : Prop
CSP.Constraint                                      : Type
CSP.BoolLanguage                                    : Type 1
CSP.BoolRel                                         : Type
Schaefer.TractableConditionKind                     : Type
Schaefer.RelationSatisfies                          : Prop
CostedTMSound                                       : Type 1
CostedReduction                                     : Type
Targets.NPCTargetDescriptor                         : Type
Targets.threeSATTarget                              : Targets.NPCTargetDescriptor
```

运行方式（需在仓库根目录）：

```powershell
lake env lean scripts/smoke_test.lean
```

结果：全部 16 个 `#check` 通过，退出码 0，无 `unknownIdentifier`。

## 5. Targets 默认可用与 explicit opt-in

`Targets` 聚合模块导入以下 9 个子模块（两个 Schaefer bridge 子模块不在其中）：

默认可用（随 `import ComplexityReduction.Targets` 直接进入作用域）：

- `Targets.ThreeSAT` — 本地 3SAT 目标。
- `Targets.ThreeSATCompleteBoundary` — 3SAT 完备边界。
- `Targets.NPCTargetCatalog` — 初始 NPC 目标目录（`NPCTargetDescriptor`、
  `threeSATTarget`、`localTargets` 等）。
- `Targets.Karp21Catalog` — Karp 21 目录。
- `Targets.CountingCSP` / `Targets.FiniteDomainCSP` — 计数与有限域 CSP 目标。
- `Targets.HColoringCatalog` / `Targets.HColoringComplexityCatalog` — H-着色目录。
- `Targets.NPCTransfer` — NPC 传递。

需要 explicit opt-in（外部 Schaefer bridge，默认不用于本地推理）：

- `Targets.Schaefer` — 仅记录外部 Schaefer 二分法项目的边界元数据。其模块头明确
  说明：兄弟项目 `../schaefer_dichotomy_lean` 同样命名为 `ComplexityReduction`，
  作为 Lake 依赖会产生模块名歧义，因此本仓库不导入其定理，只保留 audit-only 的
  桥接元数据。该模块仍被 `NPCTargetCatalog` 间接导入（仅引用
  `ExternalBridgeKind.trustedBoundary.asString` 字符串常量），但它不搬运外部定理。
- `Targets.SchaeferFiniteDomain` — Schaefer 有限域相关，同样属外部 bridge 范畴。
  该模块不在聚合模块中，且默认集内无任何模块导入它，需显式
  `import ComplexityReduction.Targets.SchaeferFiniteDomain` 才会进入作用域。

信任边界的实质保证是**数据级**而非纯导入级：`NPCTargetCatalog` 中
`localTargets = [threeSATTarget]`、`initialTargets = localTargets`，默认目录只含本地
3SAT 目标；外部 Schaefer 描述符（`schaeferBridgeTargets`）均带
`requiresExplicitOptIn = true` 与 `artifactEligibleByDefault = false`，只有
`targetsWithExternalBridges = localTargets ++ schaeferBridgeTargets` 才包含它们。
因此即使 `Targets.Schaefer` 的元数据类型随 `NPCTargetCatalog` 进入作用域，外部
bridge 目标也默认不可用于生成产物。

## 6. 已知限制与后续

- 聚合模块目前是纯导入集合，未在同名 namespace 下 re-export 符号；
  Archon 符号表需按真实全限定名（见第 4 节）维护。
- 冒烟文件不落库（受 `.gitignore` 的 `/scripts` 规则约束）；
  如需纳入 CI，可单独建立 `scripts/` 之外的测试 target，留待阶段二决定。
- 阶段一完成，可进入阶段二（Archon 源码侧改动）。

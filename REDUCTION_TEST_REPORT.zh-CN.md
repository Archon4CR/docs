# Archon + ComplexityReduction 归约问题测试报告

状态：已完成
日期：2026-08-23
分支：`complexity-reduction-integration`（Archon 仓库）

## 1. 测试目标

使用 `problems/problems_clean.json` 中整理的 NP 完全性归约问题集，
测试修改后的 Archon（集成 CR advisor 子代理、搜索脚本、参考文档、prover-mode）
在归约证明上的表现。核心问题：**CR 知识注入是否提升了 prover 的自主证明能力？**

测试路线：路线 A（sorry stub + 标准 loop + CR advisor），详见
[REDUCTION_TEST_PLAN.md](REDUCTION_TEST_PLAN.md)。

## 2. 选题

| ID | 归约 | 难度 | 选择理由 |
|----|------|------|----------|
| 01-001 | SAT -> DOUBLE-SAT | easy | 最简单入门题，CR 有 SAT 编码 |
| 02-004 | 3SAT -> NAE-3SAT | hard | 经典归约，CR 有 3SAT 编码 |
| 06-017 | CLIQUE -> INDEPENDENT SET | medium | 图论归约，解法简洁（补图） |

三道题覆盖了 CR 库的三种典型复用场景：完全复用、全新构建、部分复用。

## 3. 测试流程

````
problems_clean.json (自然语言题目)
  -> 选题 3 道
  -> 写 blueprint .tex 章节（半形式化 LaTeX + \lean{} 标注）
  -> Archon formalize 模式 -> 生成 Lean sorry stub（iter-014 完成）
  -> Archon prove 模式 + CR advisor -> 填充证明（iter-015~018）
  -> Archon polish 模式 -> golf 优化 + 清理（iter-020~021）
  -> lake build 绿色，0 sorry，0 额外公理
````

- 项目路径：`reduction-test/`
- Harness：codex（Codex CLI 子进程）
- 迭代总数：21 轮（iter-001 ~ iter-021）

## 4. CR Advisor 验证（核心测试目标）

CR advisor 在 iter-001 和 iter-004 被派发共 5 次，生成 3 份详细报告。
报告存档于 `logs/iter-006/task_results-archive/`。

### 4.1 派发记录

| 迭代 | slug | 耗时 | 状态 |
|------|------|------|------|
| iter-001 | nae3sat | 2624s | ok |
| iter-001 | double-sat | （超时未记录） | 超时 |
| iter-001 | independent-set | （超时未记录） | 超时 |
| iter-004 | independentset | 125s | fail（首次） |
| iter-004 | independentset | 2483s | ok（重试） |
| iter-004 | doublesat | （超时未记录） | 超时 |

iter-001 的 3 个并发派发中仅 nae3sat 在 44 分钟内完成，
double-sat 和 independent-set 因供应商网络超时未返回。
iter-004 重新派发后 independentset 成功（重试 1 次后），
doublesat 最终也完成了报告。

### 4.2 Advisor 判定与最终证明对照

| 题目 | Advisor 判定 | 关键建议 | 最终证明是否采纳 |
|------|-------------|----------|------------------|
| NAE3SAT | **REUSE** | 直接 re-export `threeSATToNAE3CostedReduction` | 是，证明仅 1 行 `exact (…satisfiable_iff).symm` |
| DoubleSAT | **BUILD_NEW** | 自定义 `EncodedDecisionProblem`，映射 `phi ++ [[pos y, neg y]]`，用 `satisfies_append` + `clause_vars_lt_varBound` | 是，全部 API 均被采用 |
| IndependentSet | **PARTIAL** | CLIQUE 复用 CR，IS 从零构建，`complementGraph` 来自 `Karp21.VertexCover`，`of_emptyOutputEncoding` | 是，证明用 `hasUndirectedEdge_complementGraph_iff` + `of_emptyOutputEncoding` |

**结论：CR advisor 的知识注入有效。** 三道题的最终证明策略与 advisor
报告完全一致。advisor 准确区分了三种复用程度（REUSE / BUILD_NEW / PARTIAL），
prover 在 advisor 指引下正确选择了 CR 库 API，避免了盲目搜索。

### 4.3 Advisor 报告内容示例

以 IndependentSet（PARTIAL）为例，advisor 报告包含：

- 源问题 CLIQUE 在 CR 中的完整定位（文件、行号、类型签名）
- 目标问题 INDEPENDENT SET 在 CR 中不存在（全局搜索 0 匹配）
- 可复用基础设施：`complementGraph`、`complementEdges`、`hasUndirectedEdge_complementGraph_iff`
- 构造路径：`CostedReduction.ofCostedMap` + `CostedMap.of_emptyOutputEncoding`
- 与已证归约的对比：比 `Karp21.VertexCover` 的 CLIQUE->VC 更简单（无预算变换）

## 5. 证明结果

### 5.1 总览

| 文件 | sorries | 完成迭代 | 公理 | lake build |
|------|---------|----------|------|------------|
| NAE3SAT.lean | 0 | iter-015 | 标准三公理 | 绿色 |
| DoubleSAT.lean | 0 | iter-017 | 标准三公理 | 绿色 |
| IndependentSet.lean | 0 | iter-018 | 标准三公理 | 绿色 |

标准三公理：`propext`、`Classical.choice`、`Quot.sound`（无 `sorryAx`、`admit`）。
`lake build` 全量构建 8403 jobs，exit 0。

### 5.2 NAE3SAT.lean — 3SAT -> NAE-3SAT（REUSE）

最简单的案例。CR 库 `CSP.Hardness.NAE3SAT` 已有完整的 `threeSATToNAE3CostedReduction`
和 `threeCNFToNAE3Formula_satisfiable_iff`。prover 只需 re-export 并用 1 行 `exact` 闭合。

```lean
theorem threeSATToNAE3SAT_correct (phi : SAT.ThreeCNF) :
    SAT.threeSATDecisionProblem.isYes phi <->
    nae3SATDecisionProblem.isYes (CSP.Hardness.NAE3SAT.threeCNFToNAE3Formula phi) := by
  exact (CSP.Hardness.NAE3SAT.threeCNFToNAE3Formula_satisfiable_iff phi).symm
```

完成：iter-015，2 sessions（28 + 103 turns）。

### 5.3 DoubleSAT.lean — SAT -> DOUBLE-SAT（BUILD_NEW）

中等难度。CR 库无 DOUBLE-SAT 编码，需从零构建。
归约映射：在 CNF 后追加恒真子句 `[pos y, neg y]`（`y = CNF.varBound phi` 为新变量）。

证明分两部分：
- **正确性**（`satToDoubleSAT_correct`，~80 行）：前向用 `Clause.setVar` 在新变量上产生两个不同赋值；后向从 `satisfies_append` 提取原公式可满足性。
- **尺寸界**（`LinearSizeBound.intro_with 3 9`）：追加子句贡献 `2y + 9`，`y <= inputSize(phi)`，总计 `<= 3*inputSize + 9`。

关键转折：iter-016 卡在错误的尺寸常量 `2 4`（iter-015 stub 中的占位值）。
iter-017 planner 通过编码尺寸计算诊断出正确常量为 `3 9`，prover 随后闭合。

完成：iter-017，5 sessions（155 + 60 + 98 + ... turns）。

### 5.4 IndependentSet.lean — CLIQUE -> IS（PARTIAL）

最难案例。CLIQUE 在 CR 库中已编码，IS 需从零构建。
归约映射：取补图（`complementGraph`），保持预算 `k`。

此题经历了最多波折：

| 迭代 | Mode | Sessions | Turns | 文件改动 | 结果 |
|------|------|----------|-------|----------|------|
| iter-015 | prove | 3 | 175 | 0 | 卡住：prover 推导出完整证明但从未写入文件 |
| iter-016 | fine-grained | 3 | 137 | 0 | 卡住：pure 文件读取，idle_timeout 退出 |
| iter-017 | fine-grained | 3 | — | 0 | 卡住：后端 0-token 问题 |
| iter-018 | complexity-reduction-build | 1 | 184 | 5 | 解决 |

iter-015 的 prover 在 86K 字符的 thinking block 中推导出了完整证明，
但从未调用 `apply_patch` 写入文件——这是"想清楚但没写出"的典型案例。

iter-018 切换到 `complexity-reduction-build` mode 后 1 session 即完成。
该 mode 要求不输出 sorry，强制 bottom-up 组装 CR 证书，
prover 被迫在每一步要么写出可编译的代码、要么不写。

关键证明模式：

```lean
theorem cliqueToIndependentSet_correct (I : CliqueInput) :
    Clique I <-> IndependentSet (cliqueToIndependentSetMap I) := by
  unfold Clique IndependentSet PairwiseAdjacent cliqueToIndependentSetMap
  constructor
  · -- 前向：clique 的 witness 直接复用，邻接 -> 补图非邻接
    rintro (vs, hlen, hnodup, hbounds, hadj)
    refine (vs, hlen, hnodup, hbounds, ?_)
    rintro u hu v hv huv
    rw [Karp21.VertexCover.hasUndirectedEdge_complementGraph_iff]
    simp [hadj u hu v hv huv]
  · -- 反向：by_contra，补图非邻接 -> 邻接矛盾
    rintro (vs, hlen, hnodup, hbounds, hnonadj)
    refine (vs, hlen, hnodup, hbounds, ?_)
    rintro u hu v hv huv
    by_contra h
    apply hnonadj u hu v hv huv
    rw [Karp21.VertexCover.hasUndirectedEdge_complementGraph_iff]
    exact (hbounds u hu, hbounds v hv, huv, h)
```

## 6. 测试过程中发现的问题及修复

### 6.1 Archon 源码修复（已提交）

| # | 问题 | 影响 | 修复 | Commit |
|---|------|------|------|--------|
| Fix A | `fine-grained.md` 等 mode 文件未被 `archon init` 复制 | prover 拿到空 system prompt | `runners.py` fallback 到 `prove.md` + 手动补全 5 个文件 | `5a6bc7b` |
| Fix B | prover 只探索不写代码 | IndependentSet 7 sessions 0 产出 | `prove.md` 加入 write-first protocol | `5a6bc7b` |
| Fix C | `is_complete` 不识别 `polish (complete)` | loop 无法自动停止 | `progress.py` 改为全文匹配 + 大小写不敏感 | `343f2ef` |

### 6.2 项目配置修复（未提交到 Archon）

| 问题 | 修复 |
|------|------|
| `features.plugins=false` 阻断 codex 子进程的 `exec_command` | 从 `.archon/config.json` 移除该配置项 |
| prompt variant 引用不存在的 `read_file` 工具 | 改为 `exec_command` + shell 读取 |
| 5 个 ArchonTest 残留文件误导模型 | 重写为 reduction-test 实际状态 |
| `.archon/prover-modes/` 只有 2 个文件（应有 7 个） | 从 Archon 源码手动复制缺失的 5 个 |

### 6.3 非 Archon 问题

| 问题 | 说明 |
|------|------|
| 供应商网络中断（iters 7-13） | 后端 API 不稳定，导致多轮空转 |
| `sync_leanok` Windows false-negative | 每轮删除所有 `\leanok` 标记，需手动恢复 |
| CRLF 行尾 | DoubleSAT.lean 有 CRLF，apply_patch 在 CRLF 文件上失败 |
| DoubleSAT 尺寸常量错误 `2 4` | formalize 阶段生成的 stub 占位值，planner 在 iter-017 纠正为 `3 9` |

## 7. 迭代时间线

| 迭代 | 阶段 | 关键事件 |
|------|------|----------|
| 001-004 | autoformalize | CR advisor 派发（5 次），3 份报告生成 |
| 005-006 | autoformalize | formalize 继续 |
| 007-013 | （空转） | 供应商网络中断 |
| 014 | autoformalize | formalize 完成：14 声明，6 sorry |
| 015 | prover | 首轮：NAE3SAT 解决，DoubleSAT 部分，IS 卡住（0 产出） |
| 016 | prover | DoubleSAT 卡在错误常量 `2 4`，IS 继续卡住 |
| 017 | prover | planner 纠正常量为 `3 9`，DoubleSAT 解决；IS 切 fine-grained 仍失败 |
| 018 | prover | IS 切 `complexity-reduction-build` mode，1 session 解决 |
| 019 | （空转） | — |
| 020 | polish | golf 优化 + 清理脚手架注释 + 修复 IS 重复声明 |
| 021 | polish | coverage debt 清理，项目标记 COMPLETE |

## 8. 统计

- 总迭代数：21
- 有效证明迭代：4（iter-015, 016, 017, 018）
- 总 sorry 数：6 -> 0
- 最终 `lake build`：8403 jobs，exit 0
- 公理：仅 `propext`、`Classical.choice`、`Quot.sound`
- CR advisor 派发：5 次（iter-001 x 3，iter-004 x 2），3 份报告成功
- Archon 源码修复：3 个（Fix A/B/C），2 次 commit

## 9. 结论

1. **CR advisor 知识注入有效。** 三道题的最终证明策略与 advisor 报告完全一致，
   advisor 准确区分了 REUSE / BUILD_NEW / PARTIAL 三种复用程度，
   prover 在 advisor 指引下正确使用了 CR 库 API。

2. **`complexity-reduction-build` mode 在困难场景下有效。** IndependentSet 在
   默认 `prove` 和 `fine-grained` mode 下 7 sessions 0 产出，切换到
   `complexity-reduction-build` 后 1 session 即完成。该 mode 的"不输出 sorry"
   约束迫使 prover 逐步组装可编译的 CR 证书。

3. **write-first 问题需关注。** prover 存在"想清楚但没写出"的行为模式——
   在 thinking block 中推导出完整证明但不调用 `apply_patch`。write-first protocol
   （Fix B）通过硬性规则约束了这一行为，但更系统的解决方案可能需要在
   runner 层面强制 apply_patch 调用。

4. **Archon 的自动完成检测有缺陷。** `is_complete` 原实现只检查 stage 的
   第一个词，无法识别 `polish (complete)` 等非标准格式，导致 loop 在项目
   完成后继续空转。Fix C 已修复。

5. **Windows 兼容性是持续问题。** `sync_leanok` false-negative、CRLF 行尾、
   `apply_patch` 在 CRLF 文件上失败等问题反复出现，建议增加 Windows CI。

6. **语句忠实性存在已知缺口。** 本轮证明的定理只覆盖了各归约的存在性与正确性
   （yes 实例当且仅当），未覆盖归约映射的多项式时间可计算性。详见第 10 节。

## 10. 已知局限：语句忠实性缺口（多项式时间义务未被形式化）

本轮测试的自然语言命题（"X is NP-complete via reduction from Y"）包含三项义务：
(i) 实例编码；(ii) 正确性（yes 实例当且仅当对应）；(iii) 归约映射多项式时间可计算。
loop 产出的 Lean 定理只覆盖了 (i) 和 (ii)：

- `DoubleSAT.lean` 证明的是 `SAT.CNF.Satisfiable phi` 与
  `doubleSATDecisionProblem.isYes (satToDoubleSATMap phi)` 之间的当且仅当。
  映射 `satToDoubleSATMap` 是普通 `def`（在 Lean 定义性可计算的意义上），对其时间复杂度没有任何陈述。
- `NAE3SAT.lean` 从 ComplexityReduction 库 re-export 了一个 `CostedReduction`，
  但本项目证明的定理（`threeSATToNAE3SAT_correct`）仍只是裸当且仅当；成本部分
  搭了库既有成果的便车，并非本轮测试消解的义务。
- `IndependentSet.lean` 证明了补图当且仅当，并额外用 `CostedMap.of_emptyOutputEncoding`
  组装了一个 `CostedReduction`。该构造子通过要求输出编码恒为空来记成本 0，
  ComplexityReduction 的文档字符串明确写道它"本身并不是忠实的自然语言编码定理"
  ——这个成本证书是空泛的。

换言之，本轮产出的每条形式定理都**严格弱于**其自然语言原句：一个零 sorry、
axiom-clean 的编译通过证明并不能确立原句声称的 NP 完全性，因为多项式时间义务
在形式化过程中被无声丢弃。这是自动形式化文献指认的"编译通过 ≠ 语义忠实"问题
（语句级语义弱化）的具体实例：LLM 评审对这类松弛不敏感，而当前管线的任何环节
都没有把义务编码为验证目标。

在成本义务被形式化之前——例如把带真实成本证书的 `CostedReduction`
（如 `CostedMap.of_encodedPolynomialSizeBound`）或等价的多项式时间可计算性陈述
作为强制证明目标，而非裸当且仅当——本轮结果应被解读为
"归约存在性与正确性已获形式化验证"，而非完整的 NP 完全性形式证明。后续工作：
从每道自然语言题目中抽取义务清单（编码 / 当且仅当 / 可计算性 / 成本界），
将其作为 prove loop 的硬约束，并增加自动松弛检测（如与 Lean 对照的蕴含强度检查）。
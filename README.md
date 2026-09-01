# Archon + ComplexityReduction 项目文档

本仓库收集 Archon 自动定理证明框架与 ComplexityReduction Lean 库集成项目的规划和报告文档。

## 文档索引

建议按以下顺序阅读：

1. COMPLEXITY_REDUCTION_INTEGRATION_PLAN.md — 三阶段集成总计划，描述目标、关键约定和验证清单
2. PHASE1_REPORT.md — 阶段一：冻结 ComplexityReduction 依赖边界、验证构建和导入面
3. PHASE2_REPORT.md — 阶段二：Archon 源码侧改动（库参考、搜索脚本、prover mode、Lake 依赖识别）
4. PHASE3_REPORT.md — 阶段三：目标项目侧接入（Lake 依赖、冒烟导入、子代理配置）
5. REDUCTION_TEST_PLAN.md — 归约测试方案，选题和路线选择
6. REDUCTION_TEST_REPORT.md — 归约测试结果报告，CR advisor 验证和 3 道题的完整证明过程

## 工作区仓库导航

这些文档涉及的工作区内有以下独立 git 仓库，均已上传至 GitHub：

- [ComplexityReduction](https://github.com/Archon4CR/ComplexityReduction) — Lean 4 计算复杂度归约库，提供 NPC 归约的形式化基础设施（EncodedDecisionProblem、CostedReduction、Karp21、Schaefer 等）
- [Archon](https://github.com/Archon4CR/Archon) — Python 实现的自动定理证明框架，驱动 Lean 形式化证明的迭代工作流（blueprint -> formalize -> prove -> polish）
- [problems](https://github.com/Archon4CR/problems) — NP 完全性归约题库，从大学课程资料中提取并清洗的 64 道归约题
- [reduction-test](https://github.com/Archon4CR/test) — 归约证明测试项目，包含 3 道题的完整 Lean 证明和 CR advisor 报告
- [docs](https://github.com/Archon4CR/docs) — 本仓库，项目文档

工作区根目录下的 AGENTS.md 是工具约定文件，不在任何 git 仓库中。scripts/ 目录存放临时辅助脚本，同样不在 git 管理下。

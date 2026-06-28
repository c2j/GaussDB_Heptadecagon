# 闭环 SQL 优化架构设计

> **面向对象**：Heptadecagon 工具链开发者、架构师
>
> **目标**：定义 ogexplain-analyzer 与 metamorphosis 协同实现闭环 SQL 优化的架构，将所需开发能力分解到各组件，规划分阶段落地路径。

---

## 目录

- [1. 背景与目标](#1-背景与目标)
- [2. 架构总览](#2-架构总览)
- [3. 组件职责](#3-组件职责)
- [4. 数据契约](#4-数据契约)
- [5. 诊断→重写映射设计](#5-诊断重写映射设计)
- [6. 验证引擎策略](#6-验证引擎策略)
- [7. 收敛检测设计](#7-收敛检测设计)
- [8. 开发路线图](#8-开发路线图)
- [9. 关键设计决策](#9-关键设计决策)

---

## 1. 背景与目标

### 1.1 问题陈述

GaussDB/openGauss 的 SQL 优化面临三个核心挑战：

1. **诊断与修复脱节**：ogexplain-analyzer 能诊断 28 类性能问题，但诊断结果与 metamorphosis 的重写规则之间没有自动映射。用户需要手动理解诊断、选择规则、执行重写。

2. **改写正确性无保证**：业界主流 SQL 重写工具（EverSQL、sql_exenv、QUITE）依赖 LLM 或启发式规则生成改写，无法数学证明等价性。重写后的 SQL 可能改变查询语义。

3. **缺少迭代闭环**：单次重写后无法自动验证优化效果，无法自动迭代到最优解。

### 1.2 设计目标

| 目标 | 衡量标准 |
|------|---------|
| **正确性保证** | 每一次重写都经过 QED(Z3 SMT) 或 VeriEQL(有界模型检查) 验证 |
| **诊断驱动重写** | 重写由 ogexplain 诊断结果触发，而非盲目模式匹配 |
| **自动收敛** | 多轮迭代直到指标不再改善或 Critical 问题清零 |
| **可审计** | 每次改写有 provenance（哪个诊断触发的），附带形式化证明证书 |
| **安全回滚** | 退化时自动回滚到上一版本，不留代码在破损状态 |

### 1.3 与业界对比

| 系统 | 等价性保证 | 重写确定性 | 诊断深度 | 迭代闭环 |
|------|----------|-----------|---------|---------|
| Oracle STA | SQL Profile (非证明) | 不改写 SQL | 通用 | ✅ |
| sql_exenv | LLM 判断 (不可靠) | LLM 生成 (随机) | 通用 | ✅ ReAct |
| QUITE | SQL solver | LLM 生成 (随机) | 无诊断 | ✅ FSM |
| EverSQL | ✗ | 闭源 | 通用 | ✗ |
| **本方案** | **QED(Z3) + VeriEQL** | **规则引擎 (确定性)** | **28 OG 专属规则** | **✅ 规划中** |

**核心差异化**：QED/VeriEQL 提供形式化等价性证明。这是业界最大痛点——LLM 工具靠概率保证正确性，Oracle STA 绕开问题（用 Profile 不改 SQL）。本方案用数学证明。

---

## 2. 架构总览

### 2.1 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        用户输入                                       │
│              SQL Text · Schema(DDL) · DB Connection                  │
└────────────────────────────┬────────────────────────────────────────┘
                             │
        ╔════════════════════▼═══════════════════════════════════════╗
        ║  PHASE 0 — PRE-FLIGHT                                    ║
        ║                                                          ║
        ║   ┌─────────────┐      ┌──────────────────┐               ║
        ║   │ Stats 检查   │─NO─→│ ANALYZE tables    │               ║
        ║   └──────┬──────┘      └────────┬─────────┘               ║
        ║          │YES                    │                         ║
        ║          ←───────────────────────┘                         ║
        ║          │                                              ║
        ║   ┌──────▼──────────────┐                               ║
        ║   │ EXPLAIN ANALYZE     │  rust-opengauss               ║
        ║   └──────┬──────────────┘                               ║
        ║          │                                              ║
        ║   ┌──────▼──────────────┐                               ║
        ║   │ ogexplain analyze    │                               ║
        ║   │ → Report₀ (baseline) │                               ║
        ║   └──────┬──────────────┘                               ║
        ╚══════════│════════════════════════════════════════════════╝
                   │  metrics₀ = { cost, time, critical_count, ... }
                   │
        ╔══════════▼════════════════════════════════════════════════╗
        ║  PHASE 1 — CLASSIFY                                     ║
        ║                                                         ║
        ║   Finding 路由到修复轨道:                                  ║
        ║                                                         ║
        ║   ┌────────────┐ ┌──────────┐ ┌────────┐ ┌────────────┐ ║
        ║   │ Track A    │ │ Track B  │ │Track C │ │ Track D    │ ║
        ║   │ SQL Rewrite│ │ DDL Adv  │ │Config  │ │ Architect  │ ║
        ║   │ → Phase 2  │ │ 输出建议  │ │输出建议 │ │ 记录警告    │ ║
        ║   └─────┬──────┘ └──────────┘ └────────┘ └────────────┘ ║
        ╚════════════│═════════════════════════════════════════════╝
                     │
        ╔════════════▼═════════════════════════════════════════════╗
        ║  PHASE 2 — THE VERIFY-GATED LOOP                         ║
        ║                                                         ║
        ║  ┌──────────────────────────────────────────────────┐   ║
        ║  │ ① MAP: finding.rule_id → rewrite rule(s)        │   ║
        ║  └────────────────────┬─────────────────────────────┘   ║
        ║                       │                                 ║
        ║  ┌────────────────────▼─────────────────────────────┐   ║
        ║  │ ② REWRITE: metamorphosis                         │   ║
        ║  │    RewriteContext { schema, diagnostic_hints }   │   ║
        ║  │    enabled_rules: [mapped rules]                 │   ║
        ║  │    → rewrite candidate SQL                       │   ║
        ║  └────────────────────┬─────────────────────────────┘   ║
        ║                       │                                 ║
        ║  ┌────────────────────▼─────────────────────────────┐   ║
        ║  │ ③ VERIFY: 两层等价性验证                          │   ║
        ║  │                                                  │   ║
        ║  │  VeriEQL ──→ EQ? ──→ ACCEPT                      │   ║
        ║  │      │                                           │   ║
        ║  │      └─NEQ──→ QED(Z3) ──→ EQ? ──→ ACCEPT         │   ║
        ║  │                              └─NEQ/timeout──→ REJECT │  ║
        ║  │                                     │           │   ║
        ║  │                              counterexample     │   ║
        ║  └────────────────────┬────────────────────│───────┘   ║
        ║                       │                    │           ║
        ║  ┌────────────────────▼──────────┐        │ CEGIS     ║
        ║  │ ④ RE-EVALUATE                 │   反馈反例给         ║
        ║  │    EXPLAIN ANALYZE             │   metamorphosis     ║
        ║  │    ogexplain → Reportₙ₊₁      │←─────────┘         ║
        ║  └────────────────────┬──────────┘                    ║
        ║                       │                                ║
        ║  ┌────────────────────▼──────────────────────────┐    ║
        ║  │ ⑤ CONVERGE?                                   │    ║
        ║  │   critical==0 → ✅ STOP (success)              │    ║
        ║  │   regression  → ⏪ ROLLBACK                    │    ║
        ║  │   plateau     → ⏸ STOP                        │    ║
        ║  │   max_iter    -> ⏹ STOP                       │    ║
        ║  │   otherwise   -> ↺ continue to ①              │    ║
        ║  └───────────────────────────────────────────────┘    ║
        ╚════════════════════════│════════════════════════════════╝
                                 │
        ╔════════════════════════▼════════════════════════════════╗
        ║  PHASE 3 — OUTPUT                                      ║
        ║                                                        ║
        ║   ✓ 优化后的 SQL (附带 QED 证明证书)                     ║
        ║   ✓ DDL 建议 (CREATE INDEX ...)                        ║
        ║   ✓ 配置建议 (SET work_mem ...)                        ║
        ║   ✓ 优化报告 (每轮 metrics 变化、provenance、收敛轨迹)    ║
        ╚════════════════════════════════════════════════════════╝
```

### 2.2 数据流

```
                    ┌─────────────────────────┐
                    │    rust-opengauss        │
                    │  (DB 连接 + EXPLAIN)     │
                    └───────────┬─────────────┘
                                │ EXPLAIN TEXT
                    ┌───────────▼─────────────┐
                    │  ogexplain-analyzer      │
                    │  (诊断引擎)              │
                    │                          │
                    │  输入: plan + sql_text   │
                    │  输出: DiagnosticReport  │
                    │  → findings[]            │
                    │  → suggestions[]         │
                    │  → summary{}             │
                    └───────────┬─────────────┘
                                │ findings (filtered)
                    ┌───────────▼─────────────┐
                    │  Diagnostic Mapper       │
                    │  (诊断→规则映射)         │
                    │                          │
                    │  SUBQ-001 → ["subquery-  │
                    │              to-join"]   │
                    │  TYPE-001 → ["add-       │
                    │              explicit-   │
                    │              cast"]      │
                    └───────────┬─────────────┘
                                │ enabled_rules + diagnostic_hints
                    ┌───────────▼─────────────┐
                    │  metamorphosis           │
                    │  (重写引擎)              │
                    │                          │
                    │  输入: SQL AST + hints   │
                    │       + schema           │
                    │  输出: rewritten SQL     │
                    └───────────┬─────────────┘
                                │ original + rewritten SQL
                    ┌───────────▼─────────────┐
                    │  QED / VeriEQL           │
                    │  (等价性验证)             │
                    │                          │
                    │  输入: two SQL + schema  │
                    │  输出: ProofReport       │
                    │  → Equivalent / NEQ      │
                    │  → counterexample        │
                    └───────────┬─────────────┘
                                │ verified SQL
                    ┌───────────▼─────────────┐
                    │  Convergence Detector    │
                    │  (收敛检测)              │
                    │                          │
                    │  输入: metricsₙ vs ₙ₊₁   │
                    │  输出: Continue / Stop    │
                    └─────────────────────────┘
```

---

## 3. 组件职责

### 3.1 ogexplain-analyzer（诊断层）

**当前状态**：28 条诊断规则，JSON 输出，CLI + TUI + MCP Server

| 职责 | 说明 |
|------|------|
| EXPLAIN 解析 | TEXT 格式执行计划 → 结构化 PlanNode 树 |
| 诊断规则引擎 | 28 条规则（25 经典 + 3 反模式），DFS 遍历检查 |
| 建议生成 | 跨规则综合，5 类建议（IndexOptimization 等） |
| SQL 重写（SUBQ-006） | 已内置 rewriter 模块，生成 `UPDATE...FROM` 重写 |
| DB 连接执行 EXPLAIN | `explain` 子命令直连 DB |

**需新增的能力**：

| 能力 | 改动位置 | 说明 |
|------|---------|------|
| `Finding.table` / `Finding.columns` 字段 | `analyzer/report.rs` | 结构化表名/列名，当前仅在 suggestion 文本中 |
| `optimize` 子命令 | `ogexplain-cli/src/lib.rs` | 编排完整闭环（Phase 2 核心） |
| diagnostic_hints 输出 | JSON output | 为 metamorphosis 提供结构化触发器 |

### 3.2 metamorphosis（重写层）

**当前状态**：14 条重写规则，Safe/Conditional/Manual 三级，QED + VeriEQL 验证引擎

| 职责 | 说明 |
|------|------|
| AST 重写 | 消费 ogsql-parser AST，不直接解析 SQL |
| 规则引擎 | `RewriteEngine` + `RuleRegistry` + `RewriteRule` trait |
| 安全分级 | Safe（自动）、Conditional（条件）、Manual（仅建议） |
| 收敛循环 | `max_iterations`（默认 10），自动检测 fixed-point |
| 等价性验证 | QED（Z3 SMT 完整证明）+ VeriEQL（有界模型检查） |

**需新增的能力**：

| 能力 | 改动位置 | 说明 |
|------|---------|------|
| `RewriteContext.diagnostic_hints` | `core/src/context.rs` | 接收 ogexplain 诊断上下文 |
| `add-explicit-cast` 规则 | `rules/src/add_explicit_cast.rs` | 对应 TYPE-001 诊断 |
| `suggest-trgm-index` 规则 | `rules/src/suggest_trgm_index.rs` | 对应 TYPE-004 诊断 |
| hint-driven 匹配模式 | 各规则的 `matches()` | 当 hints 存在时跳过 pattern 检测，直接信任诊断 |

### 3.3 QED / VeriEQL（验证层）

**当前状态**：已完整实现

| 引擎 | 方法 | 速度 | 完备性 | 最佳用途 |
|------|------|------|--------|---------|
| **VeriEQL** | 有界模型检查（OOPSLA 2024） | 秒级 | 不完备 | 快速筛除错误重写，提取反例 |
| **QED** | SMT 完整证明（嵌入式 Z3） | 分钟级 | 完备 | 正式认证正确重写 |

**无需新增能力**，但需优化调用策略（见 [§6](#6-验证引擎策略)）。

### 3.4 Orchestrator（编排层）

**当前状态**：不存在，需新建。

这是闭环优化的核心新增组件。有两个落地方案：

| 方案 | 优点 | 缺点 | 推荐 |
|------|------|------|------|
| **方案 A**：ogexplain 新增 `optimize` 子命令 | 复用 DB 连接能力、rewriter 模块 | ogexplain 耦合 metamorphosis 依赖 | ✅ 推荐 |
| **方案 B**：CodeRoughcollie 新增优化模块 | 已是集成层 | 偏 code review 定位 | 备选 |

**Orchestrator 职责**：

```
1. 调用 ogexplain 获取 EXPLAIN + 诊断
2. 过滤出 SQL-rewritable findings
3. 查 diagnostic→rule 映射表
4. 调用 metamorphosis 执行定向重写
5. 调用 QED/VeriEQL 验证
6. re-EXPLAIN + 指标对比
7. 收敛检测
8. 生成优化报告
```

---

## 4. 数据契约

### 4.1 DiagnosticReport（ogexplain 输出）

```json
{
  "findings": [
    {
      "rule_id": "TYPE-001",
      "severity": "Critical",
      "category": "TypeMismatch",
      "title": "疑似隐式类型转换",
      "detail": "Seq Scan on orders 含过滤条件: (status = '42')...",
      "node_line": 1,
      "node_type": "SeqScan",
      "suggestion": "WHERE status = 42 — 建议去掉引号",
      "sql_rewrite": null,
      "table": null,
      "columns": []
    }
  ],
  "suggestions": [
    {
      "related_rules": ["SCAN-001", "JOIN-001"],
      "category": "IndexOptimization",
      "message": "建议创建复合索引...",
      "confidence": 0.85
    }
  ],
  "summary": {
    "total_cost": 45000.0,
    "total_time_ms": 234.0,
    "critical_count": 2,
    "warning_count": 3,
    "spill_kb": null,
    "peak_memory_kb": null,
    "actual_rows": 500.0,
    "estimated_rows": 100000.0,
    "worst_est_ratio": 200.0
  }
}
```

**关键 gap**：`Finding.table` 和 `Finding.columns` 字段当前不存在。各规则的 `extract_*` helper 已计算这些值（如 `relation`、`filter_cols`），但在 `make_finding` 时丢弃。需在 `report.rs` 中添加字段，并在每个规则文件中填充。

### 4.2 DiagnosticHint（传递给 metamorphosis）

```json
{
  "rule_id": "SUBQ-001",
  "severity": "Warning",
  "table": "order_items",
  "columns": ["order_id"],
  "node_type": "SubqueryScan",
  "detail": "关联子查询未提升为 JOIN"
}
```

这个结构会作为 `RewriteContext.diagnostic_hints` 传入 metamorphosis。

### 4.3 RewriteResult（metamorphosis 输出）

```json
{
  "changed": true,
  "statements": ["SELECT o.*, c.name FROM orders o JOIN ..."],
  "suggestions": [],
  "match_failures": [
    { "rule_id": "subquery-to-join", "reason": "No subquery pattern found" }
  ]
}
```

### 4.4 ProofReport（验证引擎输出）

```json
{
  "result": "Equivalent",
  "engine": "verieql",
  "translate_ms": 45,
  "solve_ms": 230,
  "bound": 3,
  "counterexample": null
}
```

```json
{
  "result": "NotEquivalent",
  "engine": "verieql",
  "translate_ms": 45,
  "solve_ms": 180,
  "bound": 3,
  "counterexample": {
    "tables": [
      { "name": "orders", "rows": [{"order_id": 1, "customer_id": null}] }
    ]
  }
}
```

---

## 5. 诊断→重写映射设计

### 5.1 修复路径分类

ogexplain 的 28 条诊断按修复方式分为四类：

```
28 条诊断
├── Track A: SQL 可重写 (6 条, 21%)
│   └── → metamorphosis 规则触发
├── Track B: DDL 需求 (3 条, 11%)
│   └── → 输出 CREATE INDEX / ALTER TABLE 建议
├── Track C: 配置调优 (4 条, 14%)
│   └── → 输出 SET work_mem 建议
└── Track D: 架构层面 (15 条, 54%)
    ├── 统计信息 (3 条) → 回到 Phase 0 执行 ANALYZE
    ├── 下推/流式/网络 (4 条) → 记录警告，需人工
    └── 其他 (8 条) → 记录警告，需人工
```

### 5.2 映射表

| ogexplain 诊断 | 严重度 | Track | metamorphosis 规则 | Safety | 当前状态 |
|----------------|--------|-------|-------------------|--------|---------|
| **SUBQ-001** 子查询未提升 | Warning | A | `subquery-to-join` | Conditional | ✅ 规则已存在 |
| **SUBQ-006** 关联子查询自更新 | Warning | A | ogexplain 自带 `sql_rewrite` | — | ✅ 已实现 |
| **REW-001** 大 IN 列表 | Warning | A | `subquery-to-join` | Conditional | ✅ 规则已存在 |
| **TYPE-001** 隐式类型转换 | Critical | A | `add-explicit-cast` | Safe | ❌ 需新建 |
| **TYPE-004** LIKE 前导通配符 | Warning | A | `suggest-trgm-index` | Manual | ❌ 需新建 |
| **AGG-001** GroupAgg→HashAgg | Warning | A | `rewrite-group-agg` | Conditional | ❌ 需新建 |
| **SCAN-001** 大表全扫 | Warning | B | — | — | 输出 DDL 建议 |
| **SCAN-004** 无索引过滤 | Warning | B | — | — | 输出 DDL 建议 |
| **PART-001** 分区剪枝失效 | Warning | B | — | — | 输出 DDL 建议 |
| **MEM-001** 排序溢出 | Critical | C | — | — | 输出配置建议 |
| **MEM-004** 峰值内存高 | Warning | C | — | — | 输出配置建议 |
| **JOIN-002** Hash 连接溢出 | Critical | C | — | — | 输出配置建议 |
| **AGG-002** HashAgg 溢出 | Critical | C | — | — | 输出配置建议 |
| **STATS-001** 统计未收集 | Warning | D | — | — | 回到 Phase 0 |
| **EST-001** 估算偏差 | Critical | D | — | — | 回到 Phase 0 |
| **EST-004** 低估导致 NL | Critical | D | — | — | 回到 Phase 0 |
| **PUSH-001** 查询未下推 | Critical | D | — | — | 架构层警告 |
| **PUSH-002** 多层流式 | Critical | D | — | — | 架构层警告 |
| **NET-001** 广播大表 | Critical | D | — | — | 架构层警告 |
| **VEC-001** 行列混用 | Warning | D | — | — | 架构层警告 |
| **DIST-001** 分布列不匹配 | Warning | D | — | — | 架构层警告 |
| **SKEW-001** 数据倾斜 | Critical | D | `probe-data-skew` | Manual | ✅ 探针规则已存在 |
| **SORT-003** 重复排序 | Warning | D | — | — | 架构层警告 |
| **GEN-001** 计划过深 | Info | D | — | — | 架构层警告 |
| **JOIN-001** NL 大表 | Critical | B | — | — | DDL 建议 |
| **ANTI-003** 索引扫描放大 | Warning | D | — | — | 架构层警告 |
| **ANTI-005** 级联物化 | Warning | D | — | — | 架构层警告 |
| **ANTI-007** CN 端大排序 | Warning | D | — | — | 架构层警告 |

### 5.3 映射实现

映射逻辑在 Orchestrator 中实现为静态查表：

```rust
fn diagnostic_to_rewrite_rules(finding: &Finding) -> RewriteAction {
    match finding.rule_id.as_str() {
        "SUBQ-001" | "REW-001" => RewriteAction::Rules(vec!["subquery-to-join"]),
        "SUBQ-006" => RewriteAction::UseBuiltinRewrite,  // ogexplain 自带
        "TYPE-001" => RewriteAction::Rules(vec!["add-explicit-cast"]),
        "TYPE-004" => RewriteAction::Rules(vec!["suggest-trgm-index"]),
        "AGG-001"  => RewriteAction::Rules(vec!["rewrite-group-agg"]),
        "SCAN-001" | "SCAN-004" | "JOIN-001" => RewriteAction::DDLAdvice,
        "MEM-001" | "MEM-004" | "JOIN-002" | "AGG-002" => RewriteAction::ConfigAdvice,
        "STATS-001" | "EST-001" | "EST-004" => RewriteAction::RunAnalyze,
        "SKEW-001" => RewriteAction::Rules(vec!["probe-data-skew"]),
        _ => RewriteAction::Log,  // 架构层，记录警告
    }
}
```

---

## 6. 验证引擎策略

### 6.1 分层验证

根据规则的 Safety 级别和改写来源，路由到不同验证级别：

```
                 Rewrite Candidate
                        │
         ┌──────────────▼──────────────┐
         │  改写来源 + SafetyLevel?      │
         └──┬───────┬───────┬───────────┘
            │       │       │
         Safe    Conditional  Diagnostic-driven
         规则     规则         新规则
            │       │       │
            ▼       ▼       ▼
      VeriEQL    VeriEQL    VeriEQL
      bound=1   bound=2-3   bound=3
      (smoke    (快速       (深度
       test)    筛选)        筛选)
        │        │         │
        │        │      ┌──▼──┐
        │        │      │ QED │  ← 正式 SMT 证明
        │        │      └──┬──┘
        ▼        ▼         ▼
      accept   accept    accept
                          (仅 QED 通过才接受)
```

### 6.2 CEGIS 反馈循环

当 VeriEQL 发现 NOT EQUIVALENT 时，输出具体反例：

```rust
ProofResult::NotEquivalent {
    counterexample: Counterexample {
        tables: vec![
            TableData { name: "orders", rows: [...] },
        ]
    }
}
```

反例反馈给 metamorphosis 修正改写逻辑：

```
重写 v1 → VeriEQL → NOT_EQ (counterexample: orders.customer_id = NULL)
                           │
                           ▼
           metamorphosis 分析反例:
           "子查询→JOIN 时漏了 NULL 处理"
                           │
                           ▼
重写 v2 (加 NULL guard) → VeriEQL → EQ ✓
```

### 6.3 验证超时处理

| 超时场景 | 处理策略 |
|---------|---------|
| VeriEQL 超时 (< 10s) | 升级到 QED |
| QED 超时 (< 60s) | 简化 SQL（拆分子查询分别验证） |
| QED 仍超时 | 减小 VeriEQL bound |
| 均超时 | 标记 "未经形式化验证"，需人工确认 |

### 6.4 Rich Schema 利用

QED 的 `schema` 模块从 DDL 提取丰富约束，显著提升证明效率：

| 约束类型 | 加速效果 |
|---------|---------|
| PRIMARY KEY | JOIN 等价性证明利用参照完整性 |
| FOREIGN KEY | 缩小搜索空间 |
| NOT NULL | NOT IN → LEFT JOIN...IS NULL 安全性依赖此约束 |
| CHECK | 限制值域，加速 SMT 求解 |

**要求**：Orchestrator 必须将完整 DDL（而非仅表名+列名）传入 QED 验证引擎。

---

## 7. 收敛检测设计

### 7.1 收敛指标

从 ogexplain 的 `summary` 字段提取对比指标：

| 指标 | 字段 | 方向 |
|------|------|------|
| 优化器代价 | `total_cost` | ↓ 越低越好 |
| 执行时间 | `total_time_ms` | ↓ 越低越好 |
| Critical 诊断数 | `critical_count` | ↓ 越低越好 |
| Warning 诊断数 | `warning_count` | ↓ 越低越好 |
| 磁盘溢出 | `spill_kb` | ↓ 越低越好 |
| 峰值内存 | `peak_memory_kb` | ↓ 越低越好 |
| 估算偏差 | `worst_est_ratio` | ↓ 越接近 1 越好 |

### 7.2 停止条件

```rust
enum LoopDecision {
    Continue,
    Stop(StopReason),
}

enum StopReason {
    Success,           // critical_count == 0
    Plateau,           // 连续 N 轮改善 < 阈值
    Regression,        // 指标恶化
    MaxIterations,     // 达到迭代上限
    NoRewritableRules, // 剩余诊断均为非 SQL-rewritable
}

fn should_continue(
    prev: &SummaryRow,
    curr: &SummaryRow,
    config: &LoopConfig,
    state: &LoopState,
) -> LoopDecision {
    // 1. 无 Critical 诊断 → 成功
    if curr.critical_count == 0 {
        return Stop(StopReason::Success);
    }

    // 2. 指标恶化 → 回滚
    if curr.total_cost > prev.total_cost * 1.10 {
        return Stop(StopReason::Regression);
    }

    // 3. 平台期检测
    let improvement = (prev.total_cost - curr.total_cost) / prev.total_cost;
    if improvement < config.min_improvement_pct {
        if state.plateau_count >= config.max_plateau {
            return Stop(StopReason::Plateau);
        }
    }

    // 4. 迭代上限
    if state.iteration >= config.max_iterations {
        return Stop(StopReason::MaxIterations);
    }

    // 5. 无可重写诊断
    if !state.has_rewritable_findings {
        return Stop(StopReason::NoRewritableRules);
    }

    LoopDecision::Continue
}
```

### 7.3 默认配置

```rust
struct LoopConfig {
    max_iterations: usize,              // 默认 10
    min_improvement_pct: f64,           // 默认 0.05 (5%)
    max_plateau: usize,                 // 默认 3
    regression_threshold_pct: f64,      // 默认 0.10 (10%)
    require_equivalence_proof: bool,    // 默认 true
    auto_run_analyze: bool,             // 默认 true
}
```

---

## 8. 开发路线图

> **实施策略**：先做一条规则的深度样板，再推广。首个 pilot 选定 `SUBQ-001 → subquery-to-join`，详细计划见 [Pilot 实施计划](pilot-subquery-to-join.md)。

### Phase 0：Pilot — subquery-to-join 深度样板（3 周）

**目标**：以 SUBQ-001 为首个规则，打通完整闭环，建立可复制模式。

| 周 | 里程碑 | 核心交付 | 详细计划 |
|---|--------|---------|---------|
| Week 1 | 管道贯通 | Finding 结构化字段 + RewriteContext.diagnostic_hints + 收敛检测 + 编排器 + 端到端测试 | [Week 1 任务分解](pilot-subquery-to-join.md#4-week-1-详细任务分解) |
| Week 2 | 验证引擎补全 | VeriEQL encoder EXISTS/InSubquery arm + CEGIS 反馈循环 + rewrite→verify 链式测试 | [Week 2 规划](pilot-subquery-to-join.md#3-三周里程碑) |
| Week 3 | 打磨样板 | 修复 QED 假测试 + NOT EXISTS/NOT IN 变体 + "添加新规则" Checklist | [Week 3 规划](pilot-subquery-to-join.md#3-三周里程碑) |

**完整计划文档**：[`docs/pilot-subquery-to-join.md`](pilot-subquery-to-join.md)

### Phase 1：推广到第二批规则

**前提**：Pilot Week 3 完成，Checklist 模板就绪。

**目标**：按 Pilot 建立的模式，快速复制到 3-5 条高优先级诊断。

| 规则 | 对应诊断 | 优先级 | 预估工作量 | 说明 |
|------|---------|--------|-----------|------|
| `add-explicit-cast` | TYPE-001 | Critical | 3 天 | QED 可直接验证；Safe 级别 |
| `subquery-to-join` 扩展 | REW-001 | Warning | 2 天 | Pilot 规则的自然延伸（IN 列表→JOIN） |
| `suggest-trgm-index` | TYPE-004 | Warning | 3 天 | Manual 级别，输出 DDL 建议 |

### Phase 2：完整编排器

**前提**：Phase 1 完成，3+ 条规则已验证。

**目标**：`ogexplain optimize` 子命令实现多规则自动化闭环。

| 任务 | 组件 | 说明 |
|------|------|------|
| `optimize` 子命令框架 | ogexplain CLI | clap 子命令定义，参数解析 |
| Stats 检查 + 自动 ANALYZE | ogexplain | 查询 pg_stat_user_tables，自动执行 ANALYZE |
| 多规则并行/串行重写编排 | ogexplain | 按 finding 优先级排序，逐条重写+验证 |
| re-EXPLAIN + 指标对比 | ogexplain | 调用 rust-opengauss 重新执行 |
| 收敛检测 + 回滚 | ogexplain | 实现 StopReason + 退化回滚 |
| 优化报告生成 | ogexplain | 每轮 metrics 变化、provenance、证明证书 |

### Phase 3：增强能力

**目标**：提升覆盖率和用户体验。

| 任务 | 组件 | 说明 |
|------|------|------|
| `rewrite-group-agg` 规则 | metamorphosis | 对应 AGG-001 |
| 批量优化模式 | ogexplain | `--batch` 支持 CSV 多 SQL |
| DDL 自动执行（可选） | ogexplain | `--auto-ddl` 自动创建索引（需确认） |
| 配置自动调优（可选） | ogexplain | `--auto-config` 自动 SET work_mem |
| MCP 集成 | ogexplain | `optimize` 作为 MCP 工具暴露 |
| 迭代可视化 | ogexplain | TUI 模式展示每轮指标变化曲线 |

---

## 9. 关键设计决策

### 决策 1：Orchestrator 放在 ogexplain 还是独立组件？

**决策**：放在 ogexplain，作为 `optimize` 子命令。

**理由**：
- ogexplain 已有 DB 连接能力（`explain` 子命令）
- ogexplain 已有 rewriter 模块（可扩展为调用 metamorphosis）
- 循环的入口是"分析"，ogexplain 是自然的 owner
- metamorphosis 保持纯粹（AST 输入→SQL 输出），不引入 DB 依赖
- 避免新建独立 repo/crate 的维护成本

**代价**：ogexplain 需新增 metamorphosis + QED 作为可选依赖（feature gate）。

### 决策 2：diagnostic_hints 如何传递？

**决策**：ogexplain JSON 输出新增 `diagnostic_hints` 字段，metamorphosis `RewriteContext` 新增 `diagnostic_hints` option。

**传递路径**：
```
ogexplain analyze → JSON output → diagnostic_hints[]
                                      ↓
ogexplain optimize (orchestrator) → RewriteContext.diagnostic_hints
                                      ↓
metamorphosis RewriteEngine → rules consult hints in matches()
```

**理由**：
- 保持两个工具的松耦合（metamorphosis 仍可独立使用）
- hints 是 optional 的，不影响 metamorphosis 的独立运行
- 符合 metamorphosis 现有模式（`known_variables`、`schema` 都是 optional context）

### 决策 3：验证策略——每轮都验证还是最终验证？

**决策**：分层验证，不是每轮都跑完整 QED。

| 改写类型 | 验证策略 |
|---------|---------|
| Safe 规则（构造等价） | VeriEQL bound=1 smoke test |
| Conditional 规则 | VeriEQL bound=2-3 |
| 诊断驱动新规则 | VeriEQL + QED |
| 最终交付前 | QED 完整证明（无论规则级别） |

**理由**：QED 分钟级延迟会拖慢迭代。分层策略在保证安全性的前提下优化吞吐。

### 决策 4：收敛判据——cost 还是 time？

**决策**：以 cost 为主，time 为辅。

**理由**：
- cost 是确定性的（同一计划同一 cost），time 受系统负载影响
- EXPLAIN（不带 ANALYZE）也能获取 cost，不需要实际执行
- time 作为辅助验证：如果 cost 下降但 time 上升，说明优化器代价模型与实际不符

### 决策 5：是否自动执行 DDL（CREATE INDEX）？

**决策**：默认不自动执行，仅输出建议。`--auto-ddl` flag 可选启用。

**理由**：
- DDL 是破坏性操作（创建索引消耗存储、影响写入性能）
- 索引选择需要人工判断（哪些列组合、索引类型）
- 自动执行风险太高

---

## 附录 A：组件改动矩阵

| 改动 | ogexplain-analyzer | metamorphosis | QED/VeriEQL | Orchestrator |
|------|:---:|:---:|:---:|:---:|
| Phase 1 (MVP) | — | — | — | 脚本 |
| Finding.table/columns | ✅ | — | — | — |
| diagnostic_hints 字段 | — | ✅ | — | — |
| add-explicit-cast 规则 | — | ✅ | — | — |
| 映射引擎 | — | — | — | ✅ |
| optimize 子命令 | ✅ | — | — | ✅ |
| 收敛检测器 | — | — | — | ✅ |
| 报告生成器 | — | — | — | ✅ |
| CEGIS 反馈 | — | ✅ | — | ✅ |

## 附录 B：术语表

| 术语 | 含义 |
|------|------|
| **Finding** | ogexplain 的一条诊断结果 |
| **DiagnosticHint** | 从 Finding 提取的结构化重写触发器 |
| **RewriteAction** | metamorphosis 重写引擎的一个动作（Replace/Generate/Suggest） |
| **ProofReport** | QED/VeriEQL 的等价性验证结果 |
| **CEGIS** | Counterexample-Guided Inductive Synthesis，反例驱动的迭代修正 |
| **Convergence** | 闭环迭代的收敛——指标不再改善 |
| **Plateau** | 连续多轮改善低于阈值的平台期 |
| **Regression** | 重写后指标恶化（比改写前更差） |
| **Provenance** | 改写来源追溯——哪个诊断触发了哪次重写 |

---

> **相关文档**
>
> - 用户指南：[SQL 闭环优化流程指南](sql-optimization-guide.md)
> - ogexplain-analyzer 设计规格：`ogexplain-analyzer/.sisyphus/plans/ogexplain-analyzer-spec.md`
> - metamorphosis 设计文档：`metamorphosis/docs/metamorphosis_design_doc.md`
> - QED 理论文档：`metamorphosis/docs/QED.md`

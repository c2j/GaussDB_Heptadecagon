# Pilot 实施计划：subquery-to-join 闭环

> **状态**：规划中，逐步调整
>
> **目标**：以 `SUBQ-001 → subquery-to-join` 为首个规则，打通 ogexplain 诊断 → metamorphosis 重写 → QED/VeriEQL 验证 → re-EXPLAIN → 收敛检测的完整闭环，建立可复制的样板模式。

---

## 目录

- [1. 整体思路](#1-整体思路)
- [2. 选型理由](#2-选型理由)
- [3. 三周里程碑](#3-三周里程碑)
- [4. Week 1 详细任务分解](#4-week-1-详细任务分解)
- [5. Week 1 验收标准](#5-week-1-验收标准)
- [6. 风险与缓解](#6-风险与缓解)
- [附录 A：文件改动清单](#附录-a文件改动清单)
- [附录 B：后续规则 Checklist 模板](#附录-b后续规则-checklist-模板)

---

## 1. 整体思路

### 1.1 核心原则

**一条规则做到深，再推广到其他规则。** 不追求广度覆盖（28 条诊断 × 14 条规则的全映射），而是选一条规则，把从诊断到验证的每一个环节都打通、打磨、文档化，形成可复制的工程模式。

### 1.2 闭环全景

```
ogexplain analyze ──────────────────────────────────────────
    │
    ├─ Finding { rule_id: "SUBQ-001", table: "items", ... }
    │                                        ↓ 结构化字段
    ├─ DiagnosticHint { rule_id, table, columns, ... }
    │                                        ↓ 映射引擎
    ├─ RewriteContext { enabled_rules: ["subquery-to-join"], diagnostic_hints }
    │                                        ↓
    ├─ metamorphosis rewrite ─────────────── ─┐
    │   EXISTS(subquery) → INNER JOIN          │ Week 1: 管道贯通
    │                                        ─┘
    ├─ QED / VeriEQL verify ─────────────── ─┐
    │   VeriEQL: EQ / NEQ(counterexample)     │ Week 2: 验证补全
    │   CEGIS: 反例 → 修正改写 → 重验         │
    │                                        ─┘
    ├─ re-EXPLAIN ANALYZE ─────────────────── ─┐
    │   ogexplain analyze → Reportₙ₊₁          │ Week 1: 收敛检测
    │   metrics 对比                           │
    │                                        ─┘
    └─ 收敛? → 继续 / 停止 ────────────────── ─┐
                                               │ Week 3: 打磨样板
                                               │ 修复假测试、文档化
                                               ─┘
```

### 1.3 分层递进

每周解决一个抽象层的问题：

| 周 | 解决的层 | 不解决的问题 |
|---|---------|------------|
| Week 1 | **管道层**：数据怎么从 A 流到 B | 验证引擎的编码缺口 |
| Week 2 | **验证层**：怎么证明重写是对的 | 其他规则的验证 |
| Week 3 | **模式层**：怎么让后续规则可复制 | 扩展到其他诊断 |

---

## 2. 选型理由

### 2.1 为什么选 subquery-to-join

| 标准 | subquery-to-join | 其他候选 |
|------|:-:|:-:|
| 两端都已有代码 | ✅ ogexplain 检测 + metamorphosis 规则 | add-explicit-cast: 规则需新建 |
| 验证有意义（非平凡） | ✅ Conditional——真有 cardinality 风险 | between-to-eq: 太简单，验证像走形式 |
| 经典高频优化场景 | ✅ 子查询→JOIN 是最常见的 SQL 优化 | TYPE-001: 去引号太琐碎 |
| 暴露全部工程挑战 | ✅ hint 传递 + 验证缺口 + CEGIS + 多变体 | 其他规则只暴露部分 |

### 2.2 暴露的工程挑战（及对应解决周）

| 挑战 | 描述 | 解决周 |
|------|------|--------|
| Finding 缺结构化字段 | table/columns 嵌在 suggestion 文本里 | Week 1 |
| RewriteContext 无 hints | 无法将诊断上下文传递给重写引擎 | Week 1 |
| 诊断→规则映射 | 哪个 finding 触发哪个 rewrite rule | Week 1 |
| VeriEQL encoder 缺口 | EXISTS/InSubquery 落入 `_` fallback arm | Week 2 |
| Cardinality 正确性风险 | IN→JOIN 可能改变行数 | Week 2 (CEGIS) |
| 假测试 | QED 的 4 个 subquery 测试是 identity proof | Week 3 |
| 多变体（IN/NOT IN/EXISTS/NOT EXISTS） | 每种写法需要不同的重写策略 | Week 3 |

### 2.3 为什么不选更简单的规则先做

简单规则（如 between-to-eq）无法暴露验证引擎的缺口，也无法展示 CEGIS 反馈循环的价值。做完之后推广到复杂规则时，仍然要解决这些缺口。不如一步到位，用一个有足够深度的规则趟出完整路径。

---

## 3. 三周里程碑

### Week 1：管道贯通

**目标**：端到端闭环跑通，即使验证环节暂时用 Conditional 安全检查代替。

```
输入: SELECT * FROM orders WHERE EXISTS (SELECT 1 FROM items WHERE items.order_id = orders.id)
输出: 重写后的 SQL + 前后指标对比
验证: ⏳ Conditional 安全检查（is_safe_subquery）通过
```

**交付物**：
- `ogexplain optimize` 子命令（或编排脚本）
- Finding 结构化字段（table/columns）
- RewriteContext.diagnostic_hints
- 收敛检测逻辑
- 端到端测试用例

### Week 2：验证引擎补全 + CEGIS

**目标**：让 VeriEQL 真正能验证子查询重写，经历 CEGIS 循环。

```
重写候选: EXISTS → INNER JOIN
    ↓
VeriEQL 验证（补全 encoder）
    ↓
发现反例: orders 1 行 × items 5 行匹配 → cardinality 不一致
    ↓ CEGIS
修正改写: 加 DISTINCT 或改为 semi-join
    ↓
VeriEQL 重验 → Equivalent ✓
```

**交付物**：
- VeriEQL encoder 补全 EXISTS/InSubquery arm
- CEGIS 反馈机制
- rewrite→verify 链式测试
- 至少一个经历 CEGIS 修正的测试用例

### Week 3：打磨样板

**目标**：打磨到可以作为后续规则的复制模板。

**交付物**：
- 修复 QED 假测试（4 个 identity proof → 真实子查询 vs JOIN 对）
- 补全 NOT EXISTS / NOT IN 变体
- 完整文档："添加新规则" Checklist
- 代码审查 + 重构

---

## 4. Week 1 详细任务分解

### 任务依赖关系

```
T1 (Finding 字段) ─────────────┐
T2 (RewriteContext hints) ─────┤
T3 (收敛检测) ─────────────────┼──→ T5 (编排器) ──→ T6 (E2E 测试)
T4 (映射引擎) ─────────────────┘
```

T1/T2/T3/T4 可并行，T5 依赖 T1+T2+T4，T6 依赖 T5。

---

### T1：Finding 结构化字段（ogexplain）

**目标**：给 Finding 添加 `table` 和 `columns` 字段，让诊断结果携带结构化表名/列名。

**改动文件**：

#### T1.1: `crates/ogexplain-core/src/analyzer/report.rs`

在 Finding struct 中添加两个字段：

```rust
// 当前 (report.rs:55-68):
#[derive(Debug, Clone, Serialize, PartialEq)]
pub struct Finding {
    pub rule_id: String,
    pub severity: Severity,
    pub category: DiagnosticCategory,
    pub title: String,
    pub detail: String,
    pub node_line: Option<usize>,
    pub node_type: Option<String>,
    pub suggestion: Option<String>,
    pub sql_rewrite: Option<crate::rewriter::types::RewriteResult>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub evidence: Option<crate::analyzer::pattern::types::Evidence>,
}

// 改为:
#[derive(Debug, Clone, Serialize, PartialEq)]
pub struct Finding {
    pub rule_id: String,
    pub severity: Severity,
    pub category: DiagnosticCategory,
    pub title: String,
    pub detail: String,
    pub node_line: Option<usize>,
    pub node_type: Option<String>,
    pub suggestion: Option<String>,
    pub sql_rewrite: Option<crate::rewriter::types::RewriteResult>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub evidence: Option<crate::analyzer::pattern::types::Evidence>,
    /// 关联表名（从计划节点提取，用于下游工具定向重写）
    #[serde(skip_serializing_if = "Option::is_none")]
    pub table: Option<String>,
    /// 关联列名（从过滤条件/连接条件提取）
    #[serde(skip_serializing_if = "Vec::is_empty")]
    pub columns: Vec<String>,
}
```

#### T1.2: `crates/ogexplain-core/src/analyzer/rules/mod.rs`

扩展 `make_finding` 函数签名：

```rust
// 当前 (mod.rs:83-101):
fn make_finding(
    rule: &dyn DiagnosticRule,
    detail: String,
    node: &PlanNode,
    suggestion: Option<String>,
) -> Finding {
    Finding {
        rule_id: rule.id().to_string(),
        severity: rule.severity(),
        category: rule.category(),
        title: rule.name(),
        detail,
        node_line: Some(node.line_number),
        node_type: Some(node.node_type.to_string()),
        suggestion,
        sql_rewrite: None,
        evidence: None,
    }
}

// 改为:
fn make_finding(
    rule: &dyn DiagnosticRule,
    detail: String,
    node: &PlanNode,
    suggestion: Option<String>,
) -> Finding {
    make_finding_ext(rule, detail, node, suggestion, None, Vec::new())
}

/// Extended make_finding with structured table/column data.
fn make_finding_ext(
    rule: &dyn DiagnosticRule,
    detail: String,
    node: &PlanNode,
    suggestion: Option<String>,
    table: Option<String>,
    columns: Vec<String>,
) -> Finding {
    Finding {
        rule_id: rule.id().to_string(),
        severity: rule.severity(),
        category: rule.category(),
        title: rule.name(),
        detail,
        node_line: Some(node.line_number),
        node_type: Some(node.node_type.to_string()),
        suggestion,
        sql_rewrite: None,
        evidence: None,
        table,
        columns,
    }
}
```

> **设计决策**：保留 `make_finding` 签名不变（向后兼容），新增 `make_finding_ext`。需要填充 table/columns 的规则调用 ext 版本，其余规则不用改。

#### T1.3: `crates/ogexplain-core/src/analyzer/rules/subquery_rules.rs`

SUBQ-001 已经在计算 `child_table`（第 66-68 行），只需传给 `make_finding_ext`：

```rust
// 当前 (subquery_rules.rs:62-76):
fn check(&self, node: &PlanNode, _ctx: &PlanContext) -> Option<Finding> {
    if node.node_type == NodeType::SubqueryScan
        || node.node_type == NodeType::VectorSubqueryScan
    {
        let child_table = find_first_scan_descendant(node)
            .map(|r| first_identifier(&r))
            .unwrap_or_else(|| "unknown".to_string());

        return Some(make_finding(
            self,
            t!("finding.SUBQ-001.detail_subquery_scan", table = child_table).to_string(),
            node,
            Some(t!("finding.SUBQ-001.suggestion_subquery_scan").to_string()),
        ));
    }
    // ...
}

// 改为:
fn check(&self, node: &PlanNode, _ctx: &PlanContext) -> Option<Finding> {
    if node.node_type == NodeType::SubqueryScan
        || node.node_type == NodeType::VectorSubqueryScan
    {
        let child_table = find_first_scan_descendant(node)
            .map(|r| first_identifier(&r))
            .unwrap_or_else(|| "unknown".to_string());

        return Some(make_finding_ext(
            self,
            t!("finding.SUBQ-001.detail_subquery_scan", table = &child_table).to_string(),
            node,
            Some(t!("finding.SUBQ-001.suggestion_subquery_scan").to_string()),
            Some(child_table),       // ← 新增：结构化表名
            Vec::new(),              // ← 列名待后续从 plan 属性中提取
        ));
    }
    // ...
}
```

**预估工作量**：0.5 天

**验收**：`cargo test --workspace` 全通过；`ogexplain analyze` 的 JSON 输出中出现 `"table": "items"` 字段。

---

### T2：RewriteContext.diagnostic_hints（metamorphosis）

**目标**：让 metamorphosis 的重写引擎能接收 ogexplain 的诊断上下文。

**改动文件**：

#### T2.1: `crates/core/src/types.rs`

新增 DiagnosticHint 类型：

```rust
/// 诊断提示，由 ogexplain-analyzer 传入，用于驱动定向重写。
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct DiagnosticHint {
    /// 来源诊断规则 ID（如 "SUBQ-001"）
    pub rule_id: String,
    /// 关联表名
    pub table: Option<String>,
    /// 关联列名
    pub columns: Vec<String>,
    /// 严重级别
    pub severity: String,
    /// 诊断详情
    pub detail: String,
}
```

#### T2.2: `crates/core/src/context.rs`

在 RewriteContext 中添加字段：

```rust
// 当前 (context.rs:35-48):
pub struct RewriteContext<'a> {
    pub version: Option<&'a str>,
    pub schema: Option<&'a SchemaMap>,
    pub config: &'a RewriteConfig,
    pub source_file: Option<&'a str>,
    pub known_variables: Option<&'a HashSet<String>>,
}

// 改为:
pub struct RewriteContext<'a> {
    pub version: Option<&'a str>,
    pub schema: Option<&'a SchemaMap>,
    pub config: &'a RewriteConfig,
    pub source_file: Option<&'a str>,
    pub known_variables: Option<&'a HashSet<String>>,
    /// 来自 ogexplain-analyzer 的诊断提示。
    /// 存在时，规则可跳过自行 pattern 检测，直接信任诊断结果。
    pub diagnostic_hints: Option<&'a Vec<crate::types::DiagnosticHint>>,
}
```

#### T2.3: RewriteContext literal 站点更新

所有构造 `RewriteContext { ... }` 的地方加一行 `diagnostic_hints: None,`。

预计涉及的文件（根据 explore agent 报告，约 6 处）：

| 文件 | 行号（约） | 场景 |
|------|-----------|------|
| `crates/cli/src/main.rs` | ~620 | rewrite 命令 |
| `crates/cli/src/main.rs` | ~696 | suggest 命令 |
| `crates/cli/src/main.rs` | ~795 | rewrite CSV 模式 |
| `crates/cli/src/main.rs` | ~1005 | suggest CSV 模式 |
| `crates/cli/src/main.rs` | ~1248 | rewrite procedure 模式 |
| `crates/cli/src/main.rs` | ~1386 | suggest procedure 模式 |
| `crates/mcp-server/src/tools.rs` | ~91 | MCP 工具 |

> 用 `grep -rn "RewriteContext {" crates/` 找到所有站点。

#### T2.4: `crates/rules/src/subquery_to_join.rs`

在 `matches()` 开头加入 hint 检查：

```rust
fn matches(&self, ctx: &RewriteContext, stmt: &Statement) -> MatchResult {
    // 如果有诊断提示说存在 SUBQ-001，增强信心但仍需 pattern 检测定位子查询位置
    let has_diagnostic = ctx.diagnostic_hints
        .map(|hints| hints.iter().any(|h| h.rule_id == "SUBQ-001"))
        .unwrap_or(false);

    // 原有 pattern 检测逻辑不变——仍需定位子查询在 AST 中的位置
    match stmt {
        // ... existing pattern detection ...
    }
}
```

> **设计决策**：Week 1 不改 matches 逻辑——hint 只增强信心，不跳过 pattern 检测。Week 3 再考虑 hint-driven 快速路径。

**预估工作量**：0.5 天

**验收**：`cargo build --workspace` 成功；`cargo test --workspace` 全通过。

---

### T3：收敛检测（ogexplain）

**目标**：对比两轮 DiagnosticReport 的 summary 指标，判断是否继续迭代。

**改动文件**：

#### T3.1: 新建 `crates/ogexplain-core/src/convergence.rs`

```rust
use serde::Serialize;
use crate::summary::SummaryRow;

#[derive(Debug, Clone, Serialize)]
pub struct LoopConfig {
    pub max_iterations: usize,
    pub min_improvement_pct: f64,
    pub max_plateau_count: usize,
    pub regression_threshold_pct: f64,
}

impl Default for LoopConfig {
    fn default() -> Self {
        Self {
            max_iterations: 10,
            min_improvement_pct: 0.05,
            max_plateau_count: 3,
            regression_threshold_pct: 0.10,
        }
    }
}

#[derive(Debug, Clone, Serialize)]
pub enum LoopDecision {
    Continue,
    Stop(StopReason),
}

#[derive(Debug, Clone, Serialize)]
pub enum StopReason {
    Success,
    Plateau,
    Regression,
    MaxIterations,
    NoRewritableFindings,
}

pub fn should_continue(
    prev: &SummaryRow,
    curr: &SummaryRow,
    config: &LoopConfig,
    iteration: usize,
    plateau_count: usize,
    has_rewritable: bool,
) -> LoopDecision {
    // 1. 无 Critical → 成功
    if curr.critical_count == 0 {
        return LoopDecision::Stop(StopReason::Success);
    }

    // 2. 指标恶化 → 回滚
    if let (Some(prev_cost), Some(curr_cost)) = (prev.total_cost, curr.total_cost) {
        if curr_cost > prev_cost * (1.0 + config.regression_threshold_pct) {
            return LoopDecision::Stop(StopReason::Regression);
        }
    }

    // 3. 平台期
    if let (Some(prev_cost), Some(curr_cost)) = (prev.total_cost, curr.total_cost) {
        if prev_cost > 0.0 {
            let improvement = (prev_cost - curr_cost) / prev_cost;
            if improvement < config.min_improvement_pct
                && plateau_count >= config.max_plateau_count
            {
                return LoopDecision::Stop(StopReason::Plateau);
            }
        }
    }

    // 4. 迭代上限
    if iteration >= config.max_iterations {
        return LoopDecision::Stop(StopReason::MaxIterations);
    }

    // 5. 无可重写诊断
    if !has_rewritable {
        return LoopDecision::Stop(StopReason::NoRewritableFindings);
    }

    LoopDecision::Continue
}
```

#### T3.2: `crates/ogexplain-core/src/lib.rs`

添加模块声明：

```rust
pub mod convergence;
```

**预估工作量**：0.5 天

**验收**：单元测试覆盖 5 种 StopReason + Continue 场景。

---

### T4：诊断→规则映射引擎（ogexplain）

**目标**：将 ogexplain 的 finding.rule_id 映射到 metamorphosis 的规则 ID 和重写动作。

**改动文件**：

#### T4.1: 新建 `crates/ogexplain-core/src/mapper.rs`

```rust
use serde::Serialize;
use crate::analyzer::report::Finding;

/// 一条诊断对应的修复动作
#[derive(Debug, Clone, Serialize)]
pub enum RemediationAction {
    /// 用 metamorphosis 规则重写
    Rewrite { rules: Vec<&'static str> },
    /// 用 ogexplain 自带的 sql_rewrite（如 SUBQ-006）
    UseBuiltinRewrite,
    /// DDL 建议（如 CREATE INDEX）
    DdlAdvice,
    /// 配置建议（如 SET work_mem）
    ConfigAdvice,
    /// 执行 ANALYZE 后重试
    RunAnalyze,
    /// 架构层警告，需人工介入
    Log,
}

/// 诊断 rule_id → 修复动作 的静态映射
pub fn map_diagnostic(rule_id: &str) -> RemediationAction {
    match rule_id {
        "SUBQ-001" | "REW-001" => RemediationAction::Rewrite {
            rules: vec!["subquery-to-join"],
        },
        "SUBQ-006" => RemediationAction::UseBuiltinRewrite,
        "TYPE-001" => RemediationAction::Rewrite {
            rules: vec!["add-explicit-cast"], // Week 3+ 新建
        },
        "SCAN-001" | "SCAN-004" | "JOIN-001" => RemediationAction::DdlAdvice,
        "MEM-001" | "MEM-004" | "JOIN-002" | "AGG-002" => RemediationAction::ConfigAdvice,
        "STATS-001" | "EST-001" | "EST-004" => RemediationAction::RunAnalyze,
        _ => RemediationAction::Log,
    }
}

/// 从 findings 中筛选可重写的诊断
pub fn filter_rewritable(findings: &[Finding]) -> Vec<&Finding> {
    findings
        .iter()
        .filter(|f| matches!(map_diagnostic(&f.rule_id), RemediationAction::Rewrite { .. } | RemediationAction::UseBuiltinRewrite))
        .collect()
}
```

#### T4.2: `crates/ogexplain-core/src/lib.rs`

```rust
pub mod mapper;
```

**预估工作量**：0.5 天

**验收**：单元测试覆盖映射表所有分支。

---

### T5：编排器（ogexplain optimize）

**目标**：串联完整闭环：analyze → map → rewrite → re-EXPLAIN → converge。

**改动文件**：

#### T5.1: `crates/ogexplain-cli/src/optimize_cmd.rs`（新建）

```rust
use ogexplain_core::{
    analyzer::config::DiagnosticConfig,
    convergence::{should_continue, LoopConfig, LoopDecision},
    mapper::{map_diagnostic, filter_rewritable, RemediationAction},
    summary::SummaryRow,
};
// 注意：metamorphosis 作为可选依赖，通过 feature gate 引入

pub struct OptimizeArgs {
    pub sql: String,
    pub config_path: String,       // ~/.gaussdb-mcp.toml
    pub schema_path: Option<String>,
    pub max_iterations: usize,
    pub lang: String,
}

pub fn run_optimize(args: OptimizeArgs) -> anyhow::Result<()> {
    let loop_config = LoopConfig {
        max_iterations: args.max_iterations,
        ..Default::default()
    };

    let mut current_sql = args.sql.clone();
    let mut iteration = 0;
    let mut plateau_count = 0;
    let mut prev_metrics: Option<SummaryRow> = None;
    let mut history: Vec<IterationRecord> = Vec::new();

    loop {
        iteration += 1;

        // Step 1: EXPLAIN + analyze
        let report = run_explain_and_analyze(&current_sql, &args)?;
        let curr_metrics = report.summary.clone().unwrap_or_default();

        // Step 2: Convergence check (从第 2 轮起)
        if let Some(prev) = &prev_metrics {
            let has_rewritable = !filter_rewritable(&report.findings).is_empty();
            let decision = should_continue(
                prev,
                &curr_metrics,
                &loop_config,
                iteration,
                plateau_count,
                has_rewritable,
            );
            match decision {
                LoopDecision::Stop(reason) => {
                    print_final_report(&history, reason, &current_sql);
                    return Ok(());
                }
                LoopDecision::Continue => {
                    // 更新 plateau counter
                    if let (Some(p), Some(c)) = (prev.total_cost, curr_metrics.total_cost) {
                        if p > 0.0 && (p - c) / p < loop_config.min_improvement_pct {
                            plateau_count += 1;
                        } else {
                            plateau_count = 0;
                        }
                    }
                }
            }
        }

        // Step 3: Filter rewritable findings
        let rewritable = filter_rewritable(&report.findings);
        if rewritable.is_empty() {
            print_final_report(&history, StopReason::NoRewritableFindings, &current_sql);
            return Ok(());
        }

        // Step 4: Map findings → rewrite rules
        let finding = &rewritable[0]; // Week 1: 只处理第一条
        let action = map_diagnostic(&finding.rule_id);

        // Step 5: Rewrite
        let rewritten_sql = match &action {
            RemediationAction::UseBuiltinRewrite => {
                // SUBQ-006: 从 finding.sql_rewrite 提取
                finding.sql_rewrite.as_ref()
                    .map(|r| r.rewritten_sql.clone())
                    .unwrap_or_else(|| current_sql.clone())
            }
            RemediationAction::Rewrite { rules } => {
                // 调用 metamorphosis rewrite
                call_metamorphosis_rewrite(
                    &current_sql,
                    rules,
                    &args.schema_path,
                    &finding.table,
                    &finding.columns,
                    &finding.rule_id,
                )?
            }
            _ => {
                // 非重写类（DDL/Config/Log），Week 1 跳过
                current_sql.clone()
            }
        };

        if rewritten_sql == current_sql {
            // 重写未产生变化
            print_final_report(&history, StopReason::NoRewritableFindings, &current_sql);
            return Ok(());
        }

        // Record this iteration
        history.push(IterationRecord {
            iteration,
            rule_id: finding.rule_id.clone(),
            action: format!("{:?}", action),
            metrics_before: prev_metrics.clone(),
            metrics_after: Some(curr_metrics.clone()),
        });

        prev_metrics = Some(curr_metrics);
        current_sql = rewritten_sql;
    }
}

fn call_metamorphosis_rewrite(
    sql: &str,
    rules: &[&str],
    schema_path: &Option<String>,
    table: &Option<String>,
    columns: &[String],
    rule_id: &str,
) -> anyhow::Result<String> {
    // Week 1 实现：调用 metamorphosis CLI 作为子进程
    // Week 2+ 改为 library API 调用（避免子进程开销）

    use std::process::Command;
    use std::io::Write;
    use std::fs;

    // 写入临时 SQL 文件
    let input_path = "/tmp/ogexplain_optimize_input.sql";
    fs::write(input_path, sql)?;

    let mut cmd = Command::new("metamorphosis");
    cmd.arg("rewrite")
       .arg("--file").arg(input_path)
       .arg("--rules").arg(rules.join(","));

    if let Some(schema) = schema_path {
        cmd.arg("--schema").arg(schema);
    }
    cmd.arg("--input-format").arg("sql");

    let output = cmd.output()?;
    if !output.status.success() {
        anyhow::bail!("metamorphosis rewrite failed: {}", String::from_utf8_lossy(&output.stderr));
    }

    Ok(String::from_utf8_lossy(&output.stdout).trim().to_string())
}

// ... 迭代记录、报告输出等辅助结构
```

#### T5.2: `crates/ogexplain-cli/src/lib.rs`

注册 `Optimize` 子命令（clap）：

```rust
#[derive(clap::Subcommand)]
pub enum Command {
    Analyze { /* ... */ },
    Explain { /* ... */ },
    Optimize {
        #[arg(short = 's', long)]
        sql: Option<String>,
        #[arg(short = 'f', long)]
        sql_file: Option<String>,
        #[arg(long)]
        config: Option<String>,
        #[arg(long)]
        schema: Option<String>,
        #[arg(long, default_value = "10")]
        max_iterations: usize,
    },
    // ...
}
```

**预估工作量**：2 天

**验收**：能对一个含子查询的 SQL 完成一轮 analyze→rewrite→re-EXPLAIN→对比。

---

### T6：端到端测试

**目标**：验证 Week 1 闭环可运行。

#### T6.1: 测试 SQL

```sql
-- test_subquery_optimize.sql
SELECT o.order_id, o.customer_id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.order_id IN (
    SELECT i.order_id FROM items i WHERE i.amount > 100
);
```

#### T6.2: 预期流程

```
1. ogexplain explain → EXPLAIN ANALYZE → diagnose
   Finding: SUBQ-001 (Warning), table=items

2. map_diagnostic("SUBQ-001") → Rewrite { rules: ["subquery-to-join"] }

3. metamorphosis rewrite --rules subquery-to-join
   输出: SELECT ... JOIN items i ON o.order_id = i.order_id WHERE i.amount > 100

4. ogexplain explain → re-EXPLAIN ANALYZE on rewritten SQL
   metrics: cost 下降, critical 减少

5. should_continue → Continue / Stop
```

#### T6.3: 测试方式

Week 1 用手动脚本验证（不需要自动化测试框架）：

```bash
#!/bin/bash
# test_e2e.sh
SQL="SELECT o.order_id FROM orders o WHERE o.order_id IN (SELECT i.order_id FROM items i WHERE i.amount > 100)"

ogexplain optimize \
  --sql "$SQL" \
  --config ~/.gaussdb-mcp.toml \
  --schema schema.json \
  --max-iterations 3
```

**预估工作量**：0.5 天

---

## 5. Week 1 验收标准

| # | 验收项 | 验证方法 | 状态 |
|---|--------|---------|------|
| V1 | Finding JSON 包含 `table` 字段 | `ogexplain analyze fixture.txt --format json \| jq '.findings[0].table'` | ☐ |
| V2 | RewriteContext 编译通过 | `cargo build --workspace` (metamorphosis) | ☐ |
| V3 | 收敛检测逻辑正确 | `cargo test -p ogexplain-core convergence` | ☐ |
| V4 | 映射引擎覆盖 SUBQ-001 | `cargo test -p ogexplain-core mapper` | ☐ |
| V5 | `ogexplain optimize` 子命令可执行 | `ogexplain optimize --help` | ☐ |
| V6 | 端到端：子查询 SQL 能触发重写 | `ogexplain optimize --sql "..."` 输出重写后 SQL | ☐ |
| V7 | 端到端：re-EXPLAIN 指标有对比 | 输出包含 "cost: X→Y (-Z%)" | ☐ |
| V8 | 无回归 | `cargo test --workspace` (两个 repo) 全通过 | ☐ |

---

## 6. 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| metamorphosis rewrite CLI 输出格式不含纯 SQL | 中 | T5 受阻 | 提前验证 `metamorphosis rewrite --file x.sql` 的 stdout 格式；如需调整，修改 CLI 输出 |
| ogexplain 的 explain 子命令需要 DB 连接 | 高 | T6 需要 DB 环境 | 使用 ogagila 示例库（docker-compose up -d）；或用预存的 EXPLAIN 文本绕过 DB 连接 |
| RewriteContext literal 站点数量超预期 | 低 | T2 工作量增加 | 用 `grep -rn "RewriteContext {" crates/` 确认；每处加一行 `diagnostic_hints: None` |
| metamorphosis rewrite 对 IN(subquery) 的重写不生效 | 中 | T6 无法完成 | 确认 `subquery_to_join.rs` 的 `is_safe_subquery` 前置检查不会拒绝测试 SQL；必要时调整测试 SQL |
| cargo workspace 依赖冲突（ogexplain 引入 metamorphosis） | 中 | T5 编译失败 | Week 1 用子进程调用（不引入 library 依赖）；Week 2 再改为 library API |

---

## 附录 A：文件改动清单

### ogexplain-analyzer 仓库

| 文件 | 改动类型 | 任务 | 说明 |
|------|---------|------|------|
| `crates/ogexplain-core/src/analyzer/report.rs` | 修改 | T1 | Finding 新增 table/columns 字段 |
| `crates/ogexplain-core/src/analyzer/rules/mod.rs` | 修改 | T1 | 新增 make_finding_ext |
| `crates/ogexplain-core/src/analyzer/rules/subquery_rules.rs` | 修改 | T1 | SUBQ-001 调用 make_finding_ext |
| `crates/ogexplain-core/src/convergence.rs` | **新建** | T3 | 收敛检测逻辑 |
| `crates/ogexplain-core/src/mapper.rs` | **新建** | T4 | 诊断→规则映射引擎 |
| `crates/ogexplain-core/src/lib.rs` | 修改 | T3/T4 | 模块声明 |
| `crates/ogexplain-cli/src/optimize_cmd.rs` | **新建** | T5 | optimize 子命令 |
| `crates/ogexplain-cli/src/lib.rs` | 修改 | T5 | 注册 Optimize 子命令 |

### metamorphosis 仓库

| 文件 | 改动类型 | 任务 | 说明 |
|------|---------|------|------|
| `crates/core/src/types.rs` | 修改 | T2 | 新增 DiagnosticHint 类型 |
| `crates/core/src/context.rs` | 修改 | T2 | RewriteContext 新增 diagnostic_hints |
| `crates/cli/src/main.rs` | 修改 | T2 | ~6 处 literal 加 `diagnostic_hints: None` |
| `crates/mcp-server/src/tools.rs` | 修改 | T2 | 1 处 literal |
| `crates/rules/src/subquery_to_join.rs` | 修改（可选） | T2 | hint 检查（Week 1 可跳过） |

### 总计

- 新建文件：4 个
- 修改文件：~10 个
- 新增代码行（估计）：~400 行
- 修改代码行（估计）：~50 行

---

## 附录 B：后续规则 Checklist 模板

> Week 3 完成后，以下 checklist 将作为添加新规则的模板。

### 添加一条诊断驱动的重写规则，需要：

```
□ 1. ogexplain 侧
  □ 1a. 确认诊断规则已实现（rule_id, severity, category）
  □ 1b. Finding 填充 table/columns 结构化字段（调用 make_finding_ext）
  □ 1c. 确认 JSON 输出包含结构化字段

□ 2. mapper 侧
  □ 2a. 在 mapper.rs map_diagnostic() 中添加 rule_id → RemediationAction 映射
  □ 2b. 如果是新规则（非已有 metamorphosis 规则），在映射中指定规则名

□ 3. metamorphosis 侧
  □ 3a. 实现 RewriteRule trait（id, matches, apply, safety_level）
  □ 3b. 在 rules/src/lib.rs builtin_rules() 中注册
  □ 3c. 如果是 Conditional 规则，实现 is_safe_* 前置检查

□ 4. 验证
  □ 4a. QED 能翻译该规则产出的 SQL（无 UnsupportedExpr）
  □ 4b. QED 能 soundly 编码（检查 z3_solver.rs match arm）
  □ 4c. 如果 QED 不行，VeriEQL 能翻译 + 编码
  □ 4d. 编写 verify_rewrite 测试（原 SQL vs 重写 SQL → Equivalent）
  □ 4e. 如果 VeriEQL 发现反例（NEQ），走 CEGIS 修正流程

□ 5. 编排器
  □ 5a. 确认 optimize 子命令能端到端处理该规则
  □ 5b. 添加测试用例

□ 6. 文档
  □ 6a. 更新用户指南（sql-optimization-guide.md）
  □ 6b. 更新设计文档映射表（closed-loop-optimization-design.md §5.2）
  □ 6c. 更新本文档的覆盖状态
```

---

> **文档状态**：Week 1 任务待执行。后续每周结束时更新验收标准状态（☐ → ✅）。
>
> **相关文档**
>
> - 用户指南：[SQL 闭环优化流程指南](sql-optimization-guide.md)
> - 架构设计：[闭环优化架构设计](closed-loop-optimization-design.md)

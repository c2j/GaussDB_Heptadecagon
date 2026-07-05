# Design Doc: `sql-pattern-search` — codeweb × astgrep 进程内集成

## 1. 概述

为 codeweb 新增 `sql-pattern-search` 子命令，利用 astgrep 的 YAML 规则引擎，对 codeweb bincode 图库中已提取的 SQL（存储过程、函数、MyBatis Mapper、Java 内嵌 SQL）进行模式匹配，输出结构化的安全/质量发现。

### 核心价值

```
codeweb analyze  →  bincode 图库（含全部 SQL 文本）
                         ↓
codeweb sql-pattern-search --rule rules/security.yaml  →  结构化发现 + 调用链上下文
                         ↓
              发现可直接关联到 codeweb 节点 key + trace() 影响范围
```

### 与现有 `trace-sql` 的区别

| | `trace-sql` | `sql-pattern-search` |
|---|---|---|
| 查找方式 | SQL 文本子串匹配 | YAML 规则 AST 模式匹配 |
| 匹配精度 | 字符串级别 | 语义级别（元变量、条件、污点分析） |
| 规则复用 | 无 | 兼容 astgrep/semgrep YAML 规则 |
| 输出 | 人类可读文本 | JSON / 文本，含 Finding + codeweb 节点上下文 |

---

## 2. 已验证的依赖分析

### 2.1 版本冲突矩阵

```
                    codeweb         astgrep           冲突？
─────────────────────────────────────────────────────────────
ogsql-parser        v0.8.0          v0.8.4            ⚠️ 不同版本
                    (branch=main)   (tag=v0.8.4)
                    commit 376bed8   commit c02ceed

tree-sitter         0.24.7          0.25.10           🔴 HARD CONFLICT
                    (links="tree-    (links="tree-
                     sitter")        sitter")

tree-sitter-java    0.23            0.23.5            ⚠️ 次版本差异

thiserror           2.x             1.x (workspace)   ⚠️ 主版本差异
```

### 2.2 已验证的 HARD CONFLICT

`tree-sitter` 使用 `links = "tree-sitter"` 声明，链接到原生 C 库。Cargo 的解析器强制要求整个依赖图中只有一个版本使用给定的 `links` 值。编译验证**已确认失败**：

```
error: failed to select a version for `tree-sitter`.
  astgrep-matcher → astgrep-parser → tree-sitter ^0.25
  vs
  ogsql-parser(java feature) → tree-sitter ^0.24
  
  only one package may specify links = "tree-sitter"
```

**根本原因**：`astgrep-matcher` 依赖 `astgrep-parser`（用于 `PatternTreeParser` 解析规则模式字符串），而 `astgrep-parser` 硬依赖 `tree-sitter ^0.25`。`ogsql-parser` 的 `java` feature 依赖 `tree-sitter ^0.24`。两者无法共存。

### 2.3 依赖链追踪

```
astgrep-rules ──→ astgrep-matcher ──→ astgrep-parser ──→ tree-sitter ^0.25 (硬依赖)
     │                    │
     └─ astgrep-core      └─ astgrep-ast  (都无 tree-sitter)
          (无 tree-sitter)

codeweb ──→ ogsql-parser(java feature) ──→ tree-sitter ^0.24
     └─ tree-sitter-java = "0.23" (直接依赖)
```

### 2.4 为什么无法绕过

`astgrep-matcher` 依赖 `astgrep-parser` 的唯一用途是 `PatternTreeParser`：

```rust
// astgrep-matcher/src/tree_matcher.rs:19
use astgrep_parser::pattern_tree::{PatternTree, PatternTreeParser};
```

`PatternTreeParser` 解析 Semgrep 模式字符串（如 `SELECT * FROM $TABLE`）为结构化 `PatternTree`。它依赖 tree-sitter 解析模式字符串的语法。**无法从 astgrep-parser 中提取出来**（它使用 `tree_sitter::Parser` 解析模式，使用目标语言的 tree-sitter 语法）。

---

## 3. 前置条件：需要上游解决的跨模块问题

### Issue 1: `c2j/ogsql-parser` — 升级 tree-sitter 到 0.25

**问题**：`ogsql-parser` 的 `java` feature 使用 `tree-sitter = "0.24"`，与 astgrep 的 `tree-sitter = "0.25"` 冲突。

**影响范围**：
- `ogsql-parser/Cargo.toml`: `tree-sitter = { version = "0.24", optional = true }` → 升级到 `0.25`
- `ogsql-parser/Cargo.toml`: `tree-sitter-java = { version = "0.23", optional = true }` → 升级到 `0.23.5`
- 验证：tree-sitter 0.24→0.25 的 API 变更（主要涉及 `Language::load` 签名、`Parser::set_language` 返回值变化）
- 回归测试：`cargo test --features java` 确保 Java 源码 SQL 提取功能正常

**建议 issue 标题**：`deps: upgrade tree-sitter to 0.25 and tree-sitter-java to 0.23.5`

**预计工作量**：~1-2 小时（API 适配 + 测试验证）

### Issue 2: `c2j/astgrep` — astgrep-rules 暴露不依赖 parser 的入口 API

**问题**：当前 `astgrep-rules` → `astgrep-matcher` → `astgrep-parser` 的依赖链使所有规则消费者都继承了 tree-sitter 依赖。对于只需要对已解析 AST 进行规则匹配的消费者（如 codeweb），这是不必要的沉重依赖。

**建议方案**：在 `astgrep-rules` 中新增 `RuleEngine::analyze_with_parser()`（已有 `analyze`）和一个更轻量的入口，使消费者可以提供自己的 parser实现：

```rust
// 新增 feature gate: astgrep-rules 的 "standalone" feature
// 当启用时，不依赖 astgrep-matcher（及其传递的 astgrep-parser）
pub fn analyze_node(
    &mut self,
    node: &UniversalNode,          // 预解析的 AST
    context: &RuleContext,
) -> Result<Vec<Finding>>
```

**预计工作量**：~2-3 天（需要重构 astgrep-rules/astgrep-matcher 的依赖关系，将 `PatternTreeParser` 相关的类型抽象为 trait 或 feature-gate）

**备选方案**（如果 Issue 2 不可行）：
- codeweb 自行实现 `PatternTreeParser`（从 astgrep-parser 移植 ~900 行）——不推荐，维护负担重
- 接受 tree-sitter 0.25 依赖，等待 Issue 1 解决后统一升级

### Issue 3（可选）: `c2j/astgrep` — `OgsqlAdapter::convert_statement_for_test` 重命名为 `convert_statement`

**问题**：当前公有方法 `convert_statement_for_test(&Statement) -> Result<UniversalNode>` 带有 `_for_test` 后缀，但它是通用的、生产可用的 ogsql→UniversalNode 转换方法。如果 codeweb 需要使用它作为集成点，建议重命名。

**建议**：重命名为 `pub fn convert_statement(stmt: &Statement) -> Result<UniversalNode>`。保留原方法作为 `#[deprecated]` 别名。

---

## 4. 架构设计

### 4.1 数据流

```
codeweb sql-pattern-search --rule <yaml> --project <dir> [--dialect gaussdb] [--format json|text]

  1. 加载 YAML 规则
     │  RuleEngine::load_rules_from_file(rule_path)
     │
  2. 加载 bincode 图库
     │  Project::find(project) → load_store()
     │
  3. 遍历 SQL 节点（按类型批处理）
     │  store.nodes_by_type("mapper")  → MappedStatement
     │  store.nodes_by_type("sql")     → JavaSql
     │  store.nodes_by_type("proc")    → Procedure
     │  store.nodes_by_type("func")    → Function
     │
  4. 对每个 SQL 文本：
     │  a. 预规范化（MyBatis 占位符 → ?）
     │  b. GaussDBDialect::parse(&sql) → Box<dyn AstNode>
     │  c. RuleEngine::analyze(ast, &ctx) → Vec<Finding>
     │
  5. 组装输出
     │  每条 Finding + codeweb 节点上下文（key, file, line, callers）
     │  JSON 或 人类可读文本
```

### 4.2 新增文件

```
src/
├── commands/
│   └── sql_pattern_search.rs    # 新增：主逻辑 (~300 行)
├── main.rs                      # 修改：+Commands 变体 + 分发 (~30 行)
└── error.rs                     # 修改：+错误变体 (~5 行)
```

### 4.3 CLI 接口

```bash
# 基本用法
codeweb sql-pattern-search --rule rules/sql_injection.yaml

# 指定项目目录
codeweb sql-pattern-search --rule rules/select_star.yaml --project ./my-project

# 指定 SQL 方言（默认 gaussdb）
codeweb sql-pattern-search --rule rules/compatibility.yaml --dialect opengauss

# JSON 输出
codeweb sql-pattern-search --rule rules/weak_encryption.yaml --format json

# 输出到文件
codeweb sql-pattern-search --rule rules/privilege.yaml --output findings.json

# 仅匹配指定节点类型
codeweb sql-pattern-search --rule rules/sql_injection.yaml --node-type mapper

# 显示调用链影响
codeweb sql-pattern-search --rule rules/select_star.yaml --trace
```

### 4.4 输出 JSON Schema

```json
{
  "rule_file": "rules/sql_injection.yaml",
  "dialect": "gaussdb",
  "total_findings": 3,
  "findings": [
    {
      "rule_id": "sql-injection-001",
      "severity": "ERROR",
      "confidence": "HIGH",
      "message": "SQL injection vulnerability: String concatenation in WHERE clause",
      "fix": "Use parameterized queries with placeholders (?) or named parameters",
      "codeweb_node": {
        "key": "proc:pkg_product.search_products",
        "type": "proc",
        "file": "sql/pkg_product.sql",
        "line": 14,
        "sql_matched": "SELECT * FROM t_products WHERE name LIKE '%' || p_keyword || '%'"
      },
      "impact": {
        "callers": [
          {
            "key": "method:com.example.service.ProductService.search",
            "type": "method",
            "file": "java/com/example/service/ProductService.java",
            "line": 42
          }
        ],
        "callees": []
      }
    }
  ]
}
```

---

## 5. 详细实现步骤

### 步骤 1: Cargo.toml — 添加依赖（Issue 1 解决后）

```toml
# codeweb/Cargo.toml — 在 [dependencies] 下添加

# astgrep 集成（均为 git 依赖，非 crates.io）
astgrep-core = { git = "https://github.com/c2j/astgrep", tag = "v0.4.0" }
astgrep-ast  = { git = "https://github.com/c2j/astgrep", tag = "v0.4.0" }
astgrep-parser = { git = "https://github.com/c2j/astgrep", tag = "v0.4.0",
    default-features = false }  # 禁用 sql-tree-sitter，不引入 tree-sitter-sequel
astgrep-rules = { git = "https://github.com/c2j/astgrep", tag = "v0.4.0" }

# 注意: astgrep-rules → astgrep-matcher → astgrep-parser 形成传递链
# astgrep-parser 的 tree-sitter ^0.25 依赖将与升级后的 ogsql-parser 统一
```

### 步骤 2: src/error.rs — 新增错误类型

```rust
#[derive(Debug, thiserror::Error)]
pub enum CodeWebError {
    // ... 现有变体 ...

    #[error("sql pattern search error: {message}")]
    SqlPatternSearch { message: String },
}
```

### 步骤 3: src/main.rs — CLI 注册

在 `Commands` 枚举中添加：

```rust
/// Search SQL nodes via astgrep YAML rules against the bincode graph
SqlPatternSearch {
    /// Path to astgrep rule YAML file
    #[arg(short, long)]
    rule: PathBuf,

    /// Project directory (default: current directory)
    #[arg(short, long, default_value = ".")]
    project: PathBuf,

    /// SQL dialect for parsing (default: gaussdb)
    #[arg(long, default_value = "gaussdb",
           value_parser = ["gaussdb", "opengauss", "standard", "polardb-mysql"])]
    dialect: String,

    /// Output format
    #[arg(long, default_value = "text", value_parser = ["text", "json"])]
    format: String,

    /// Output file (stdout if omitted)
    #[arg(short, long)]
    output: Option<PathBuf>,

    /// Filter by node type (mapper, sql, proc, func)
    #[arg(long)]
    node_type: Option<String>,

    /// Also show call chain impact for each finding
    #[arg(long)]
    trace: bool,
},
```

在 `run()` 分发中添加：

```rust
Some(Commands::SqlPatternSearch { rule, project, dialect, format, output, node_type, trace }) => {
    commands::sql_pattern_search::run(&rule, &project, &dialect, &format,
                                       output.as_deref(), node_type.as_deref(), trace)
}
```

### 步骤 4: src/commands/sql_pattern_search.rs — 核心实现

```rust
use crate::error::{CodeWebError, Result};
use crate::graph::store::GraphStore;
use crate::graph::{Node, node_type_tag};
use crate::project::Project;
use astgrep_core::{AstNode, Finding, Language, SqlDialect};
use astgrep_parser::dialect;
use astgrep_rules::{RuleContext, RuleEngine};
use petgraph::Direction;
use std::path::Path;

pub fn run(
    rule_path: &Path,
    project: &Path,
    dialect_str: &str,
    format: &str,
    output: Option<&Path>,
    node_type_filter: Option<&str>,
    trace: bool,
) -> Result<()> {
    // 1. 加载规则
    let yaml = std::fs::read_to_string(rule_path).map_err(|e| {
        CodeWebError::SqlPatternSearch {
            message: format!("failed to read rule file {}: {}", rule_path.display(), e),
        }
    })?;
    let mut engine = RuleEngine::new();
    engine.load_rules_from_yaml(&yaml).map_err(|e| {
        CodeWebError::SqlPatternSearch {
            message: format!("failed to load rules: {}", e),
        }
    })?;

    // 2. 解析方言
    let sql_dialect = SqlDialect::from_str(dialect_str).ok_or_else(|| {
        CodeWebError::SqlPatternSearch {
            message: format!("unknown dialect: {}", dialect_str),
        }
    })?;
    let dialect_parser = dialect::dispatch(sql_dialect);

    // 3. 加载图库
    let mut proj = Project::find(project)?;
    let store = proj.load_store()?;
    let graph = store.graph();

    // 4. 遍历 SQL 节点
    let mut all_findings: Vec<EnrichedFinding> = Vec::new();
    let node_types: &[&str] = match node_type_filter {
        Some("mapper") => &["mapper"],
        Some("sql") => &["sql"],
        Some("proc") => &["proc"],
        Some("func") => &["func"],
        _ => &["mapper", "sql", "proc", "func"],
    };

    for nt in node_types {
        for idx in store.nodes_by_type(nt) {
            let node = &graph[*idx];
            let sql_texts = extract_sql_texts(node);

            for sql_text in sql_texts {
                // 预规范化 MyBatis 占位符
                let normalized = normalize_mybatis_placeholders(&sql_text);

                // 解析 SQL → AST
                let ast = match dialect_parser.parse(&normalized, Path::new("")) {
                    Ok(ast) => ast,
                    Err(e) => {
                        eprintln!("warning: parse failed for node {} ({:?}): {}",
                                  node_type_tag(node), idx.index(), e);
                        continue;
                    }
                };

                // 构建上下文
                let ctx = build_context(node, &sql_text, sql_dialect);

                // 执行规则匹配
                let findings = match engine.analyze(ast.as_ref(), &ctx) {
                    Ok(f) => f,
                    Err(e) => {
                        eprintln!("warning: analyze failed for node {} ({}): {}",
                                  node_type_tag(node), idx.index(), e);
                        continue;
                    }
                };

                // 5. 富化发现：关联 codeweb 节点上下文
                for finding in findings {
                    let impact = if trace {
                        Some(trace_impact(graph, *idx))
                    } else {
                        None
                    };

                    all_findings.push(EnrichedFinding {
                        finding,
                        codeweb_context: CodewebContext {
                            node_key: crate::graph::key::NodeKey::from_node(node).to_string(),
                            node_type: node_type_tag(node).to_string(),
                            file: node_source_file(node),
                            line: node_source_line(node),
                            sql_matched: sql_text.clone(),
                        },
                        impact,
                    });
                }
            }
        }
    }

    // 6. 格式化输出
    match format {
        "json" => output_json(&all_findings, rule_path, dialect_str, output)?,
        _ => output_text(&all_findings, rule_path, dialect_str)?,
    }

    Ok(())
}

// ── 辅助函数 ──

struct EnrichedFinding {
    finding: Finding,
    codeweb_context: CodewebContext,
    impact: Option<ImpactInfo>,
}

struct CodewebContext {
    node_key: String,
    node_type: String,
    file: Option<String>,
    line: Option<usize>,
    sql_matched: String,
}

struct ImpactInfo {
    callers: Vec<CallerInfo>,
    callees: Vec<CallerInfo>,
}

struct CallerInfo {
    key: String,
    node_type: String,
    file: Option<String>,
    line: Option<usize>,
}

fn extract_sql_texts(node: &Node) -> Vec<String> {
    match node {
        Node::MappedStatement { sql: Some(t), .. } => vec![t.clone()],
        Node::JavaSql { sql: Some(t), .. } => vec![t.clone()],
        Node::Procedure { body_sql, .. } | Node::Function { body_sql, .. } => {
            body_sql.iter().map(|s| s.sql_text.clone()).collect()
        }
        _ => vec![],
    }
}

fn normalize_mybatis_placeholders(sql: &str) -> String {
    // 将 __XML_PARAM_*__ 和 __XML_RAW_*__ 替换为 ?
    // 参考 store.rs:1043-1077 的 replace_xml_placeholders 实现
    let re_param = regex::Regex::new(r"__XML_PARAM_\d+__").unwrap();
    let re_raw = regex::Regex::new(r"__XML_RAW_\d+__").unwrap();
    let result = re_param.replace_all(sql, "?");
    re_raw.replace_all(&result, "?").to_string()
}

fn build_context(node: &Node, sql_text: &str, dialect: SqlDialect) -> RuleContext {
    let file_path = node_source_file(node).unwrap_or_else(|| "unknown".to_string());
    let mut ctx = RuleContext::new(file_path, Language::Sql, sql_text.to_string());
    ctx.sql_dialect = Some(dialect);
    ctx
}

fn node_source_file(node: &Node) -> Option<String> {
    match node {
        Node::MappedStatement { xml_file, .. } => Some(xml_file.to_string_lossy().to_string()),
        Node::JavaSql { java_file, .. } => Some(java_file.to_string_lossy().to_string()),
        Node::Procedure { location, .. } | Node::Function { location, .. } => {
            Some(location.file.to_string_lossy().to_string())
        }
        _ => None,
    }
}

fn node_source_line(node: &Node) -> Option<usize> {
    match node {
        Node::MappedStatement { line, .. } | Node::JavaSql { line, .. } => Some(*line),
        Node::Procedure { location, .. } | Node::Function { location, .. } => Some(location.line),
        _ => None,
    }
}

fn trace_impact(graph: &crate::graph::CodeGraph, idx: petgraph::graph::NodeIndex) -> ImpactInfo {
    let callers: Vec<_> = graph
        .neighbors_directed(idx, Direction::Incoming)
        .take(20)
        .map(|n| caller_info_from_node(&graph[n]))
        .collect();
    let callees: Vec<_> = graph
        .neighbors_directed(idx, Direction::Outgoing)
        .take(20)
        .map(|n| caller_info_from_node(&graph[n]))
        .collect();
    ImpactInfo { callers, callees }
}

fn caller_info_from_node(node: &Node) -> CallerInfo {
    CallerInfo {
        key: crate::graph::key::NodeKey::from_node(node).to_string(),
        node_type: node_type_tag(node).to_string(),
        file: node_source_file(node),
        line: node_source_line(node),
    }
}
```

### 步骤 5: 输出格式化

```rust
fn output_json(
    findings: &[EnrichedFinding],
    rule_path: &Path,
    dialect: &str,
    output: Option<&Path>,
) -> Result<()> {
    let result = serde_json::json!({
        "rule_file": rule_path.to_string_lossy(),
        "dialect": dialect,
        "total_findings": findings.len(),
        "findings": findings.iter().map(|ef| {
            let f = &ef.finding;
            let ctx = &ef.codeweb_context;
            let mut obj = serde_json::json!({
                "rule_id": f.rule_id,
                "severity": f.severity.as_str(),
                "confidence": f.confidence.as_str(),
                "message": f.message,
                "fix": f.fix_suggestion,
                "codeweb_node": {
                    "key": ctx.node_key,
                    "type": ctx.node_type,
                    "file": ctx.file,
                    "line": ctx.line,
                    "sql_matched": ctx.sql_matched,
                },
            });
            if let Some(ref impact) = ef.impact {
                obj["impact"] = serde_json::json!({
                    "callers": impact.callers.iter().map(|c| serde_json::json!({
                        "key": c.key, "type": c.node_type,
                        "file": c.file, "line": c.line,
                    })).collect::<Vec<_>>(),
                    "callees": impact.callees.iter().map(|c| serde_json::json!({
                        "key": c.key, "type": c.node_type,
                        "file": c.file, "line": c.line,
                    })).collect::<Vec<_>>(),
                });
            }
            obj
        }).collect::<Vec<_>>(),
    });

    let json_str = serde_json::to_string_pretty(&result).unwrap();

    match output {
        Some(path) => std::fs::write(path, json_str).map_err(|e| {
            CodeWebError::SqlPatternSearch {
                message: format!("failed to write output: {}", e),
            }
        })?,
        None => println!("{}", json_str),
    }
    Ok(())
}
```

---

## 6. 测试计划

### 6.1 单元测试（`tests/sql_pattern_search_test.rs`）

使用现有 e2e-demo 和 complex-demo 夹具：

```rust
#[test]
fn test_select_star_detection() {
    // 规则: select_star.yaml  → 预期在 pkg_product.sql:14 发现 SELECT *
    // 预期 Finding.rule_id = "select-star-001"
}

#[test]
fn test_missing_where_safe() {
    // 规则: missing_where.yaml → codeweb 中所有 DELETE/UPDATE 都有 WHERE
    // 预期 0 发现
}

#[test]
fn test_mybatis_dynamic_table_injection() {
    // 规则: sql_injection.yaml(009) → OrderMapper.xml:10 ${tableName}
    // 预期 1 发现
}

#[test]
fn test_no_findings_on_clean_sql() {
    // 确保合法 SQL 不产生误报
}

#[test]
fn test_json_output_format() {
    // 验证 JSON 输出 schema
}

#[test]
fn test_node_type_filter() {
    // --node-type mapper 仅匹配 MappedStatement 节点
}

#[test]
fn test_trace_impact() {
    // --trace 模式显示上游调用方
}
```

### 6.2 集成测试

```bash
# 手动验证
cargo run -- sql-pattern-search \
  --rule ../astgrep/tests/categories/sql/rules/select_star.yaml \
  --project lib/codeweb-e2e-demo

# 预期输出: 发现若干 SELECT * 用法，包含文件位置
```

---

## 7. 跨模块 Issue 清单

| # | 仓库 | 标题 | 优先级 | 阻塞？ |
|---|------|------|--------|--------|
| 1 | `c2j/ogsql-parser` | deps: upgrade tree-sitter to 0.25 and tree-sitter-java to 0.23.5 | P0 | **是** |
| 2 | `c2j/astgrep` | feat: expose analyze API that accepts pre-parsed UniversalNode (no parser dependency) | P1 | 否（可用变通方案） |
| 3 | `c2j/astgrep` | refactor: rename OgsqlAdapter::convert_statement_for_test to convert_statement | P2 | 否 |
| 4 | `c2j/codeweb` | feat: add sql-pattern-search command with astgrep rule integration | P0 | —（本设计文档） |

**Issue 1 是唯一硬阻塞**。Issue 2 可以变通（直接使用 astgrep-parser 的 GaussDBDialect），Issue 3 是锦上添花。

---

## 8. 替代方案

如果 Issue 1（ogsql-parser tree-sitter 升级）短期内无法解决：

**方案 B：独立进程模式**（零依赖冲突）
```bash
# codeweb 导出 SQL 临时文件
codeweb sql-pattern-search --rule rules/*.yaml --backend=external
  → 内部：遍历图节点，写 SQL 到 /tmp/codeweb-sql-XXXX/
  → 调用：astgrep analyze --dialect gaussdb --rules rules/ /tmp/codeweb-sql-XXXX/
  → 解析 astgrep JSON 输出，关联回 codeweb 节点
```
缺点：丢失调用链上下文，需要文件 I/O 开销，跨进程序列化。

**方案 C：MCP 编排**（零代码修改，适合 LLM 场景）
```
LLM 调用 codeweb_search_sql → 获取 SQL 节点
  → astgrep validate/analyze → 匹配规则
  → codeweb_node_detail → 获取文件位置
  → codeweb_trace → 影响分析
```
缺点：手动编排，不适合批量 CI/CD。

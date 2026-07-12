# 组件依赖关系与升级传播指南

> 本文档梳理 GaussDB Heptadecagon 生态中 11 个工具之间的依赖关系，目的是当某个基础组件或中间组件升级后，能根据依赖链逐层升级、传播变更，避免下游工具因版本不匹配而构建失败或行为异常。
>
> **最后更新**：2026-07-13（v0.8.31 基线对齐实战验证）

---

## 目录

- [1. 依赖层级总览](#1-依赖层级总览)
- [2. 依赖关系详图](#2-依赖关系详图)
- [3. 各组件依赖详情](#3-各组件依赖详情)
  - [3.1 ogsql-parser（基石层）](#31-ogsql-parser基石层)
  - [3.2 rust-opengauss（基石层）](#32-rust-opengauss基石层)
  - [3.3 ogexplain-analyzer（中间层）](#33-ogexplain-analyzer中间层)
  - [3.4 metamorphosis（中间层）](#34-metamorphosis中间层)
  - [3.5 codeweb（中间层）](#35-codeweb中间层)
  - [3.6 astgrep（中间层）](#36-astgrep中间层)
  - [3.7 flux-gauss（中间层）](#37-flux-gauss中间层)
  - [3.8 独立工具](#38-独立工具)
  - [3.9 CodeRoughcollie（顶层集成）](#39-coderoughcollie顶层集成)
- [4. 版本锁定策略现状](#4-版本锁定策略现状)
- [5. 已知不兼容问题](#5-已知不兼容问题)
- [6. 升级传播剧本](#6-升级传播剧本)
- [7. 升级操作检查清单](#7-升级操作检查清单)
- [8. v0.8.31 对齐实战记录](#8-v0831-对齐实战记录) ← **NEW**
  - [8.1 操作环境与策略](#81-操作环境与策略)
  - [8.2 遇到的问题与解决方案](#82-遇到的问题与解决方案)
  - [8.3 踩坑清单](#83-踩坑清单)
  - [8.4 关键发现](#84-关键发现)
  - [8.5 推荐操作脚本模板](#85-推荐操作脚本模板)
- [9. 常用维护命令](#9-常用维护命令)

---

<a id="1-依赖层级总览"></a>
## 1. 依赖层级总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        顶层集成层                                 │
│                     CodeRoughcollie                              │
│    消费 ogsql-parser + ogexplain + rust-opengauss +              │
│         metamorphosis + astgrep（子进程）                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
    ┌──────────┬───────────┼───────────┬──────────┬───────────────┐
    │          │           │           │          │               │
┌───▼───┐ ┌───▼────┐ ┌────▼────┐ ┌─────▼────┐ ┌───▼────┐ ┌───────▼──────┐
│ogexpl │ │meta-   │ │codeweb  │ │astgrep   │ │flux-   │ │   独立工具     │
│ain-   │ │morph-  │ │         │ │          │ │gauss   │ │grep-excel   │
│analyz │ │osis    │ │         │ │          │ │        │ │WDRProbe     │
│er     │ │        │ │         │ │          │ │        │ │SP-Complex   │
│       │ │        │ │         │ │          │ │        │ │ity-Evaluator│
└──┬──┬─┘ └───┬────┘ └────┬────┘ └────┬─────┘ └───┬────┘ └──────────────┘
   │    │     │           │           │           │       (无内部依赖)
   │    └─────┼───────────┼───────────┼───────────┘
   │          │           │           │
┌──▼──────────▼───────────▼───────────▼───────────────────────────┐
│                         基石层                                    │
│              ogsql-parser        rust-opengauss                  │
└──────────────────────────────────────────────────────────────────┘
```

| 层级 | 组件 | 内部依赖数量 | 被依赖数量 |
|------|------|------------|-----------|
| **基石层** | ogsql-parser | 0 | 6（ogexplain、metamorphosis、codeweb、astgrep、flux-gauss、CodeRoughcollie） |
| **基石层** | rust-opengauss | 0 | 2（ogexplain-analyzer[可选]、CodeRoughcollie） |
| **中间层** | ogexplain-analyzer | ogsql-parser + rust-opengauss(可选) | 1（CodeRoughcollie） |
| **中间层** | metamorphosis | ogsql-parser | 2（CodeRoughcollie、ogexplain-optimizer） ← **注意：ogexplain-optimizer 也依赖 metamorphosis** |
| **中间层** | codeweb | ogsql-parser | 1（CodeRoughcollie，计划中） |
| **中间层** | astgrep | ogsql-parser | 1（CodeRoughcollie，子进程调用） |
| **中间层** | flux-gauss | ogsql-parser | 0 |
| **独立** | grep-excel | 0 | 0 |
| **独立** | WDRProbe | 0 | 0 |
| **独立** | SP-Complexity-Evaluator | 0 | 0 |
| **顶层** | CodeRoughcollie | 5（ogsql-parser、ogexplain、rust-opengauss、metamorphosis、astgrep[子进程]） | 0 |

---

<a id="2-依赖关系详图"></a>
## 2. 依赖关系详图

以下数据均从各仓库 `Cargo.toml` 实际依赖声明中提取（截至 2026-07-13, ogsql-parser v0.8.31 基线）。

```
ogsql-parser ─────────────────────────────────────────────────────────
    ▲           ▲          ▲           ▲            ▲            ▲
    │           │          │           │            │            │
    │     ogexplain    metamorphosis  codeweb    astgrep     flux-gauss
    │     -analyzer                              (v0.8.31)
    │        │           │                       │            │
    │        │     ┌─────┘                       │            │
    │        │     │  (ogexplain-optimizer       │            │
    │        │     │   依赖 metamorphosis)        │            │
    │        │     │                             │            │
    └────────┴─────┴─────────────┴───────────────┴────────────┘
                     (全部直接依赖 ogsql-parser)

rust-opengauss ──────────────────────────────────────────────────────
    ▲                  ▲
    │                  │
    │           CodeRoughcollie
    │           (cr-db: tokio-opengauss)
    │
ogexplain-analyzer
(cli: db feature, 可选)


CodeRoughcollie ─────────────────────────────────────────────────────
    │
    ├── cr-audit-static      → ogsql-parser + metamorphosis-core + metamorphosis-rules
    ├── cr-audit-complexity  → ogsql-complexity (来自 ogexplain-analyzer)
    ├── cr-audit-explain     → ogexplain-core
    ├── cr-db                → tokio-opengauss (来自 rust-opengauss)
    ├── cr-audit-impact      → (计划集成 codeweb，三期)
    └── 封装 astgrep 安全规则  → 子进程调用，非 Cargo 依赖
```

**关键发现**：`ogsql-parser` 是整个生态中影响面最大的组件——**6 个工具直接依赖它**。其 AST 结构的任何变更都会产生最大范围的连锁升级。

**隐蔽依赖**：`ogexplain-optimizer`（ogexplain-analyzer 的子 crate）通过 git rev 依赖 `metamorphosis-*` crate。升级 ogsql-parser 时，如果只更新 ogexplain-analyzer 自身的 pin 而忘记更新 metamorphosis 的 rev，会导致**双版本 ogsql-parser 冲突**（编译器报告 `multiple different versions of crate ogsql_parser in the dependency graph`）。

---

<a id="3-各组件依赖详情"></a>
## 3. 各组件依赖详情

<a id="31-ogsql-parser基石层"></a>
### 3.1 ogsql-parser（基石层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/ogsql-parser`](https://github.com/c2j/ogsql-parser) |
| **语言** | Rust |
| **当前版本** | v0.8.31 |
| **内部依赖** | 无 |
| **构建命令** | `cargo build --release` |
| **接口形式** | 库 (lib) + MCP Server + HTTP API + TUI + CLI |
| **发布方式** | **纯 git tag**（未发布到 crates.io，404） |
| **标签体系** | 69 个 tag（v0.1.0 → v0.8.31），发布流程：手动 bump `Cargo.toml` 版本号 → `git tag vX.Y.Z` → push → GitHub Release |

**暴露给下游的关键接口**：
- AST 类型定义（180+ 语句类型）
- `parse_sql()` / `Parser::new(tokens).parse()` 两种解析入口
- SQL ↔ JSON 序列化
- iBatis/MyBatis XML 解析（feature: `ibatis`）
- Java 源码 SQL 提取（feature: `java`）
- MCP Server：7 个工具
- `Visitor` trait + `walk_statement` / `walk_pl_block`（多个下游实现此 trait）
- `Spanned<T>` 泛型包装（AST 节点 + span）
- `StatementInfo` 结构（`Parser::parse_sql` 的返回类型）

**直接影响面**（6 个工具 — 当前版本 v0.8.31 对齐后）：

| 下游组件 | 依赖方式 | 锁定策略 | 当前版本 |
|----------|---------|---------|---------|
| ogexplain-analyzer | 3 个 crate（ogexplain-core、ogexplain-optimizer、ogsql-complexity） | `tag = "v0.8.31"` | ✅ v0.8.31 |
| metamorphosis | workspace.dependencies 继承 | `tag = "v0.8.31"` | ✅ v0.8.31 |
| codeweb | 直接依赖 | `tag = "v0.8.31"`, features=`["ibatis","java"]` | ✅ v0.8.31 |
| astgrep | astgrep-parser crate | `tag = "v0.8.31"` | ✅ v0.8.31 |
| flux-gauss | fluxgauss + fluxgauss-mcp crate | `branch = "main"` | ⚠️ 浮动（未参与本次对齐） |
| CodeRoughcollie | cr-audit-static 通过 workspace.dependencies | `tag = "v0.8.31"` | ✅ v0.8.31 |

**升级影响分析**：
- **AST 类型结构变更**（破坏性）→ 全部 6 个下游必须同步升级
- **新增语法支持**（兼容性）→ 下游无需修改，但可选择利用新特性
- **MCP 工具接口变更**→ 仅影响通过 MCP 调用的场景，不影响库级集成
- **`Visitor` trait 变更**（破坏性）→ 影响 4 个实现了 Visitor 的下游（ogexplain-analyzer、metamorphosis、codeweb、CodeRoughcollie）
- **`Spanned<T>` / `StatementInfo` 变更**（破坏性）→ 影响深度 AST 遍历的下游
- **⚠️ 无 CHANGELOG / 迁移文档**：ogsql-parser 目前没有 CHANGELOG 或 BREAKING CHANGES 记录。升级前必须手动 diff 两个 tag 之间的差异，重点关注 `src/ast/`、`src/lib.rs` 的 `pub use` 导出变化。

---

<a id="32-rust-opengauss基石层"></a>
### 3.2 rust-opengauss（基石层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/rust-opengauss`](https://github.com/c2j/rust-opengauss) |
| **语言** | Rust |
| **当前版本** | v0.13.4 |
| **内部依赖** | 无 |
| **与 ogsql-parser 的关系** | **无任何依赖**（不存在 Cargo.toml、Cargo.lock、代码引用中） |
| **构建命令** | `cargo build -p gaussdb-mcp` |

rust-opengauss 是唯一**完全不受 ogsql-parser 升级影响的基石组件**。它在 ogsql-parser 升级时**不需要任何操作**。

---

<a id="33-ogexplain-analyzer中间层"></a>
### 3.3 ogexplain-analyzer（中间层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/ogexplain-analyzer`](https://github.com/c2j/ogexplain-analyzer) |
| **语言** | Rust |
| **当前版本** | v0.4.9 |
| **构建命令** | `cargo build --workspace` |
| **MCP Server** | ogexplain-mcp：5 个工具 |

**内部 crate 结构**：
```
ogexplain-analyzer (workspace)
  ├── ogexplain-core        → ogsql-parser (tag)
  ├── ogsql-complexity      → ogsql-parser (tag)
  ├── ogexplain-optimizer   → ogsql-parser (tag) + metamorphosis (git rev) ← ⚠️ 隐蔽依赖
  ├── ogexplain-cli         → ogexplain-core, ogsql-complexity, opengauss (可选 db feature)
  ├── ogexplain-tui         → ogexplain-core, ogsql-complexity
  └── ogexplain-mcp         → ogexplain-core, ogsql-complexity
```

**⚠️ 隐蔽依赖链**：`ogexplain-optimizer` 的 `Cargo.toml` 中有 4 行对 `metamorphosis` 的 git rev 依赖：
```toml
metamorphosis-core = { git = "...", rev = "a50c83e...", package = "metamorphosis-core" }
metamorphosis-rules = { git = "...", rev = "a50c83e...", package = "metamorphosis-rules" }
metamorphosis-qed = { git = "...", rev = "a50c83e...", package = "metamorphosis-qed", optional = true }
metamorphosis-verieql = { git = "...", rev = "a50c83e...", package = "metamorphosis-verieql", optional = true }
```

**升级陷阱**：如果只更新 ogexplain-analyzer 自身的 ogsql-parser pin，而忘记同步更新 metamorphosis 的 rev 指向已升级 ogsql-parser 的 metamorphosis 版本，将导致双版本 ogsql-parser 冲突。**编译器会明确报告** `multiple different versions of crate ogsql_parser in the dependency graph`。

**涉及 ogsql-parser 的源文件**（9 个，修改时优先级排序）：
1. `crates/ogsql-complexity/src/visitor.rs`（最深 AST 遍历）
2. `crates/ogexplain-core/src/rewriter/transform.rs`（构造 AST 节点）
3. `crates/ogexplain-core/src/rewriter/detector.rs`（匹配 Statement 枚举）
4. `crates/ogexplain-optimizer/src/rewrite.rs`（Parser + SqlFormatter）
5. `crates/ogexplain-optimizer/src/verify.rs`（metamorphosis QED 接口）
6. `crates/ogexplain-optimizer/src/orchestrator.rs`（注释引用）
7. `crates/ogsql-complexity/src/engine.rs`（PL/pgSQL 类型）
8. `crates/ogsql-complexity/src/pl_visitor.rs`
9. `crates/ogexplain-core/src/lib.rs`

---

<a id="34-metamorphosis中间层"></a>
### 3.4 metamorphosis（中间层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/metamorphosis`](https://github.com/c2j/metamorphosis) |
| **语言** | Rust |
| **当前版本** | v0.2.3 |
| **构建命令** | `cargo build --workspace` |
| **MCP Server** | metamorphosis-mcp：5 个工具 |

**内部 crate 结构**（7 crates）：
```
metamorphosis (workspace) — ogsql-parser pin 在 [workspace.dependencies] 中统一管理
  ├── core     → ogsql-parser
  ├── rules    → core, ogsql-parser
  ├── qed      → core, rules, ogsql-parser
  ├── verieql  → ogsql-parser
  ├── cli      → core, qed, rules, verieql, ogsql-parser
  ├── mcp-server → core, rules, qed, verieql, ogsql-parser
  └── regress (tests/) → ogsql-parser
```

**涉及 ogsql-parser 的源文件**（25 个，最深消费者）：
- `crates/core/src/extractor/mod.rs`（最深 API 使用：SchemaMap、AlterTableAction、DataType、token::decode_sql_file）
- `crates/cli/src/provenance.rs`（PL/pgSQL 深度依赖：8 个 plpgsql 类型）
- `crates/core/src/inline/inline_walker.rs`（12 个 AST 类型）
- `crates/rules/src/*.rs`（14 个规则文件，每个匹配多个 AST 类型）

---

<a id="35-codeweb中间层"></a>
### 3.5 codeweb（中间层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/codeweb`](https://github.com/c2j/codeweb) |
| **语言** | Rust |
| **当前版本** | v0.7.37 |
| **构建命令** | `cargo build --features full` |
| **MCP Server** | 6 个工具 |

**依赖关系**：
| 依赖项 | 引用方式 | 当前版本 |
|--------|---------|---------|
| ogsql-parser | `tag = "v0.8.31"`, features = `["ibatis", "java"]` | ✅ v0.8.31 |

**⚠️ 历史风险**：codeweb 曾使用 `branch = "main"`（浮动），这意味着 ogsql-parser 的任何 push 都会导致 codeweb 的 Cargo.lock 过期。v0.7.38 起已改为 tag 固定。

**涉及 ogsql-parser 的源文件**（12 个）：
- `src/parser/extractor.rs`（3108 行，实现 Visitor trait）
- `src/graph/builder.rs`（3966 行，AST 构造 + 6 个 enum 穷举匹配）
- `src/parser/ibatis_loader.rs`（ibatis feature API）
- `src/parser/java_loader.rs`（java feature API，注意 tree-sitter 版本兼容）

**测试代码中的 ogsql-parser 耦合**：`src/graph/builder.rs` 的 `#[cfg(test)]` 模块中直接构造 `ParsedStatement` 等 struct。**如果 ogsql-parser 给 struct 新增了非 Optional 字段，即使生产代码不报错，测试代码也会编译失败**（如 v0.8.31 新增的 `body_start_line: usize`）。

---

<a id="36-astgrep中间层"></a>
### 3.6 astgrep（中间层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/astgrep`](https://github.com/c2j/astgrep) |
| **语言** | Rust |
| **当前版本** | v0.4.9 |
| **构建命令** | `cargo build --release` |

**依赖关系**：
| 依赖项 | 引用方式 | 当前版本 |
|--------|---------|---------|
| ogsql-parser | `tag = "v0.8.31"`（在 astgrep-parser crate 中）| ✅ v0.8.31 |

**涉及 ogsql-parser 的源文件**（6 个 adapter 文件 + 内联测试）：
- `crates/astgrep-parser/src/adapter/ogsql/` 目录下的全部 6 个文件
- `ddl.rs`（596 行，~40 enum 变体穷举匹配）
- `expr.rs`（571 行，~30 Expr 变体）
- `mod.rs`（Statement 26 变体 dispatch）
- 内联 `#[cfg(test)]` 模块也直接调用 `Parser::new(tokens).parse()`

**⚠️ 历史问题**：astgrep 曾存在 Cargo.toml pin（`tag = "v0.8.0"`）与 Cargo.lock 解析版本（v0.8.4）不一致的问题，导致 `cargo build` 时可能重新解析到不同于预期的版本。v0.4.8+ 已解决。

---

<a id="37-flux-gauss中间层"></a>
### 3.7 flux-gauss（中间层）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/flux-gauss`](https://github.com/c2j/flux-gauss) |
| **语言** | Python（核心引擎）+ Rust（MCP Server 封装） |
| **当前版本** | v0.6.14 |
| **依赖 ogsql-parser 方式** | `git = "...", branch = "main"`（浮动） ← ⚠️ 尚未改为 tag 固定 |
| **被依赖情况** | 无 |

---

<a id="38-独立工具"></a>
### 3.8 独立工具

以下 3 个工具**无任何内部依赖**，升级它们不会影响生态中其他组件：

| 工具 | 仓库 | 语言 | 构建命令 |
|------|------|------|---------|
| **grep-excel** | [`c2j/grep-excel`](https://github.com/c2j/grep-excel) | Rust | `cargo build --release` |
| **WDRProbe** | [`c2j/WDRProbe`](https://github.com/c2j/WDRProbe) | Rust + TypeScript | Tauri 构建链 |
| **SP-Complexity-Evaluator** | [`c2j/SP-Complexity-Evaluator`](https://github.com/c2j/SP-Complexity-Evaluator) | Java / Spring Boot | `./mvnw clean package` |

---

<a id="39-coderoughcollie顶层集成"></a>
### 3.9 CodeRoughcollie（顶层集成）

| 属性 | 值 |
|------|-----|
| **仓库** | [`c2j/CodeRoughcollie`](https://github.com/c2j/CodeRoughcollie) |
| **语言** | Rust |
| **当前版本** | v0.2.7 |
| **构建命令** | `cargo build --workspace` |
| **CLI 命令** | `coderc` |

**完整依赖清单**（当前全部分别统一到 ogsql-parser v0.8.31）：

| 依赖项 | 消费 crate | 引用方式 | 状态 |
|--------|-----------|---------|------|
| **ogsql-parser** | cr-audit-static | `git = "...", tag = "v0.8.31"`（workspace.dependencies） | ✅ |
| **ogexplain-analyzer** → ogsql-complexity | cr-audit-complexity | `path = "lib/ogexplain-analyzer/crates/ogsql-complexity"` | ✅ |
| **ogexplain-analyzer** → ogexplain-core | cr-audit-explain | `path = "lib/ogexplain-analyzer/crates/ogexplain-core"` | ✅（已恢复） |
| **rust-opengauss** → tokio-opengauss | cr-db | `path = "lib/rust-opengauss/crates/tokio-opengauss"` | ✅ |
| **metamorphosis** → core + rules | cr-audit-static | `path = "lib/metamorphosis/crates/core"` + `"lib/metamorphosis/crates/rules"` | ✅ |
| **astgrep** | cr-audit-static | 子进程调用 `astgrep` CLI | ✅ 运行时依赖 |

**CodeRoughcollie 的 lib/ 嵌套子模块**（7 个）：
```
lib/
├── ogsql-parser/         → submodule
├── ogexplain-analyzer/   → submodule（含自己的 lib/metamorphosis 嵌套）
├── metamorphosis/        → submodule
├── rust-opengauss/       → submodule
├── codeweb/              → submodule
├── astgrep/              → submodule
└── cr-rules/             → submodule（无 ogsql-parser 依赖）
```

**⚠️ 嵌套子模块升级陷阱**：CodeRoughcollie 的子模块嵌套了更深层的子模块。`git submodule update --recursive` 可能因 SSH 密钥或 clone 超时而失败。升级时只需关注 **直接嵌套子模块**（`lib/*` 的第一层），不需要递归到 `lib/ogexplain-analyzer/lib/*` 深度。

---

<a id="4-版本锁定策略现状"></a>
## 4. 版本锁定策略现状

> ✅ v0.8.31 对齐后，除 flux-gauss 外，所有 ogsql-parser 消费者均已改为 **`tag = "v0.8.31"`** 固定。不再存在浮动 git 依赖。

| 消费者 | 引用方式 | 对齐后状态 |
|--------|---------|-----------|
| ogexplain-analyzer | `tag = "v0.8.31"` | ✅ 固定 |
| metamorphosis | `tag = "v0.8.31"` (workspace.dependencies) | ✅ 固定 |
| codeweb | `tag = "v0.8.31"`, features=["ibatis","java"] | ✅ 固定 |
| astgrep | `tag = "v0.8.31"` | ✅ 固定 |
| CodeRoughcollie | `tag = "v0.8.31"` (workspace.dependencies) | ✅ 固定 |
| flux-gauss | `branch = "main"` | ⚠️ 浮动（下次行动项） |

---

<a id="5-已知不兼容问题"></a>
## 5. 已知不兼容问题

> ✅ v0.8.31 对齐后，之前记录的所有不兼容问题均已解决。

| 问题 | 状态 |
|------|------|
| ~~ogexplain-analyzer #12：ogsql-parser v0.8.1 不兼容~~ | ✅ 已解决（v0.8.31 全量对齐） |
| ~~CodeRoughcollie cr-audit-explain 被禁用~~ | ✅ 已恢复 |
| ~~metamorphosis QED Verification clippy 报错~~ | ✅ 已修复（`needless_borrow`） |
| ~~codeweb 测试 `body_start_line` 缺失~~ | ✅ 已修复 |
| ~~CodeRoughcollie arm64 交叉编译失败~~ | ✅ 已修复（`gcc-aarch64-linux-gnu` + CC env） |
| flux-gauss ogsql-parser pin 浮动 | ⚠️ 待处理 |

---

<a id="6-升级传播剧本"></a>
## 6. 升级传播剧本

### 场景 A：ogsql-parser 升级

**影响范围：最大（6 个下游工具）**

```
ogsql-parser 发版 v0.X.0 → 全部 6 个消费者同步升级
```

**升级步骤**（已验证流程，基于 submodule 方案）：

**Phase 0 — 预检**：
```bash
# 1. 确认各组件当前 ogsql-parser pin
for d in ogexplain-analyzer metamorphosis codeweb astgrep CodeRoughcollie; do
  grep -rn 'ogsql-parser.*tag' lib/$d/Cargo.toml lib/$d/crates/*/Cargo.toml 2>/dev/null | sort -u
done

# 2. 确认 metamorphosis rev（ogexplain-optimizer 中的隐蔽依赖）
grep 'metamorphosis.*rev' lib/ogexplain-analyzer/crates/ogexplain-optimizer/Cargo.toml
```

**Phase 1 — 上游消费者并行升级**（与 Phase 0 组合：checkout 到最新 tag 同时改 pin）：
```bash
# 所有消费者可并行执行。每个消费者的操作模板：
cd lib/<component>
git fetch --tags origin
git checkout <latest-tag>
# 修改 Cargo.toml 中的 ogsql-parser tag → 新版本
cargo update -p ogsql-parser
cargo build --workspace  # 或对应的 build 命令
# 修复编译错误（如有）
git add -A && git commit -m "chore: bump ogsql-parser to vX.Y.Z"
git push origin HEAD:main
```

**Phase 2 — CodeRoughcollie 嵌套子模块同步**：
```bash
cd lib/CodeRoughcollie
# 同步每个嵌套子模块到 Phase 1 推送到远程的 commit
cd lib/ogexplain-analyzer && git fetch origin && git checkout <pushed-commit> && cd ../..
cd lib/metamorphosis     && git fetch origin && git checkout <pushed-commit> && cd ../..
cd lib/codeweb           && git fetch origin && git checkout <pushed-commit> && cd ../..
cd lib/astgrep           && git fetch origin && git checkout <pushed-commit> && cd ../..
cd lib/ogsql-parser      && git fetch --tags origin && git checkout vX.Y.Z && cd ../..
cargo update -p ogsql-parser
cargo build --workspace
git add lib/* Cargo.lock && git commit -m "chore: sync nested submodules to ogsql-parser vX.Y.Z"
git push origin HEAD:main
```

**Phase 3 — 版本 bump + CI 验证**：
```bash
# 每个组件 bump 版本号（+0.0.1），打 tag，等 CI 通过
cd lib/<component>
# 修改 Cargo.toml version
git add Cargo.toml && git commit -m "chore: bump version to X.Y.Z"
git tag vX.Y.Z && git push origin HEAD:main && git push origin vX.Y.Z
# 等待 CI，修复失败
```

---

## 场景 B~G（简化）

其他升级场景的影响范围较小，详见下表汇总：

| 场景 | 升级组件 | 影响下游 | 步骤要点 |
|------|---------|---------|---------|
| B | rust-opengauss | 2 个 | ogexplain-analyzer（可选 db feature）+ CodeRoughcollie（cr-db） |
| C | ogexplain-analyzer | 1 个 | CodeRoughcollie 的 ogsql-complexity + ogexplain-core 路径依赖 |
| D | metamorphosis | 2 个 | CodeRoughcollie + ogexplain-optimizer（隐蔽依赖！） |
| E | codeweb | 0（计划中） | CodeRoughcollie 三期 |
| F | astgrep | 1 个（子进程） | CodeRoughcollie 子进程调用，验证 CLI 兼容性 |
| G | 独立工具 | 0 | 无需协调 |

---

<a id="7-升级操作检查清单"></a>
## 7. 升级操作检查清单

### 通用检查项

- [ ] 确认变更类型：**破坏性**（AST 结构、API 签名变更）还是**兼容性**（新增功能、Bug 修复）
- [ ] 查阅本文档 §2 依赖详图，确认受影响的下游组件列表
- [ ] 按依赖层级**从底层到顶层**顺序升级（基石 → 中间 → 顶层）
- [ ] 每一层升级后执行 `cargo build` + 修复编译错误
- [ ] **⚠️ 检查隐蔽依赖**：ogexplain-optimizer 的 metamorphosis rev 是否需同步更新
- [ ] 每个消费者升级后 **push 到远程 main**
- [ ] CodeRoughcollie 的嵌套子模块全部同步到升级后的 commit
- [ ] 全量 `cargo build --workspace` 通过
- [ ] 每个组件 bump 版本号 +0.0.1，打 tag，等 CI 通过

### ogsql-parser 专项检查

- [ ] 6 个下游工具是否全部适配
- [ ] codeweb 的 `ibatis` 和 `java` feature 是否正常工作
- [ ] CodeRoughcollie `cargo build --workspace` 全量编译通过

### 提交后检查（CI 常见失败）

- [ ] `cargo fmt --all -- --check` — 格式检查
- [ ] `cargo clippy --workspace -- -D warnings` — 新 Rust 版本可能触发新的 clippy lint
- [ ] 测试代码中直接构造的 struct（如 `ParsedStatement`）是否有新增字段缺失
- [ ] 交叉编译 arm64：需要 `gcc-aarch64-linux-gnu` + `CC_aarch64_unknown_linux_gnu` 环境变量

---

<a id="8-v0831-对齐实战记录"></a>
## 8. v0.8.31 对齐实战记录

> **本次升级**（2026-07-13）：将全部消费者从 ogsql-parser v0.8.0~v0.8.29 拉齐到 v0.8.31。以下是完整的操作实录、踩坑记录和经验总结。

<a id="81-操作环境与策略"></a>
### 8.1 操作环境与策略

- **工作目录**：`GaussDB_Heptadecagon/lib/`（umbrella 仓库的 submodule 目录）
- **策略**：**直接在 submodule 上修改**，利用 `lib/` 的集中布局实现并行操作和即时集成验证
- **不依赖 canonical clone**：所有操作在 submodule 内完成，事后再同步 canonical clone
- **并行度**：Phase 1 的 6 个消费者**完全并行**（互不依赖），Phase 2 串行依赖 Phase 1

<a id="82-遇到的问题与解决方案"></a>
### 8.2 遇到的问题与解决方案

#### 问题 1：metamorphosis 双版本冲突 🔴

**现象**：
```
error[E0308]: mismatched types
note: there are multiple different versions of crate `ogsql_parser` in the dependency graph
```

**根因**：`ogexplain-optimizer/Cargo.toml` 中有 4 行对 `metamorphosis` 的 `rev = "a50c83e..."` 依赖。这个旧的 rev 指向一个仍使用 ogsql-parser v0.8.29 的 metamorphosis 版本。当 ogexplain-analyzer 自身升级到 v0.8.31 后，两个 ogsql-parser 版本共存导致了类型不匹配。

**解决**：将 4 个 `rev = "a50c83e..."` 全部替换为升级后的 metamorphosis commit `rev = "0a34a74..."`，然后 `cargo update -p metamorphosis-core -p metamorphosis-rules`。

**教训**：升级 ogexplain-analyzer 的 ogsql-parser pin 时，**必须同步检查 and 升级 ogexplain-optimizer 中的 metamorphosis rev**。

#### 问题 2：codeweb 测试代码 struct 字段缺失 🔴

**现象**：`cargo test --features full` 编译失败：
```
error[E0063]: missing field `body_start_line` in initializer of `ParsedStatement`
```

**根因**：v0.8.31 的 `ParsedStatement` 新增了 `body_start_line: usize` 字段。4 处测试代码（`src/graph/builder.rs` 的 `#[cfg(test)]` 模块）直接用 struct literal 初始化 `ParsedStatement`，缺少新字段。

**解决**：在 4 处 `ParsedStatement { .. }` 初始化中添加 `body_start_line: 5,`。

**教训**：
- `cargo build` 通过不代表 `cargo test` 能通过
- 测试代码中直接构造依赖项的 struct 是脆弱的——任何新增的非 Optional 字段都会导致编译失败
- 升级后应执行 `cargo test`（不仅是 `cargo build`）

#### 问题 3：CodeRoughcollie arm64 交叉编译失败 🔴

**现象**：
```
error: failed to run custom build command for `tree-sitter-java v0.23.5`
zig: error: version '.2.17' in target triple is invalid
```

**根因**：`cargo zigbuild --target aarch64-unknown-linux-gnu.2.17` 使用 zig 编译器处理 C 依赖（tree-sitter-java），但 zig 0.14.0 不支持 `.2.17` glibc 版本后缀。原生 Rust 代码可以用 zigbuild，但碰到 C crate（`cc` build script）时 zig 的 cc wrapper 会失败。

**解决**：
1. 安装 `gcc-aarch64-linux-gnu`（不仅是 `binutils-aarch64-linux-gnu`）
2. 设置环境变量 `CC_aarch64_unknown_linux_gnu=aarch64-linux-gnu-gcc` 让 C crate 使用系统交叉编译器而非 zig

**教训**：Rust 交叉编译中，C 依赖（特别是 tree-sitter-* crate）需要真正的 GCC 交叉编译器，不能完全依赖 zig。

#### 问题 4：clippy lint 因 Rust 版本升级新增 🔴

**现象**：QED Verification workflow 失败：
```
error: this expression creates a reference which is immediately dereferenced by the compiler
  --> crates/core/src/extractor/mod.rs:103:43
```

**根因**：Rust 1.97 新增了 `clippy::needless_borrow` lint。旧版本中 `read_sql_file(&path)` 的 `&path` 能通过，但新版本中 `path` 已经是引用，`&` 产生了双重引用。

**解决**：`read_sql_file(&path)` → `read_sql_file(path)`。

**教训**：即使代码未改动，Rust 工具链更新可能导致新的 clippy lint 失败。升级后应 `rustup update stable` 然后 `cargo clippy --workspace -- -D warnings`。

#### 问题 5：cargo fmt 检查失败 🟡

**现象**：`cargo fmt --all -- --check` 以非零退出码退出。

**根因**：在 submodule 内只执行了 `cargo build`，未执行 `cargo fmt`。部分文件有历史遗留的格式不一致。

**解决**：每个子模块执行 `cargo fmt --all`，提交。

**教训**：升级完成后应统一执行 `cargo fmt --all`（在各自模块内）、`cargo clippy --workspace -- -D warnings`。

<a id="83-踩坑清单"></a>
### 8.3 踩坑清单

| # | 坑 | 严重 | 避免方法 |
|---|-----|------|---------|
| 1 | 仅编译 2 个 crate 而漏掉第 3 个（ogexplain-optimizer） | 🔴 | 始终 `cargo build --workspace`，不用 `-p <单个crate>` |
| 2 | 忘记 ogexplain-optimizer 的 metamorphosis rev 依赖 | 🔴 | 升级前 grep 所有 `metamorphosis.*rev` |
| 3 | `cargo build` 通过但 `cargo test` 失败（struct 字段缺失） | 🔴 | 每次升级后运行 `cargo test --workspace` |
| 4 | Rust 工具链更新引入新 clippy lint | 🟡 | `rustup update stable` + `cargo clippy --workspace -- -D warnings` |
| 5 | cargo fmt 历史债务 | 🟡 | 每个模块 `cargo fmt --all` |
| 6 | Arm64 CI 交叉编译缺少 gcc-aarch64 | 🔴 | 检查 CI yml，确认安装了 gcc-aarch64-linux-gnu |
| 7 | `git submodule update --recursive` 因深层嵌套超时 | 🟡 | 只更新直接子模块，不用 `--recursive` |
| 8 | `git add -A` 在 detached HEAD 下带上 untracked 文件 | 🟡 | 只 `git add` 需要提交的文件 |
| 9 | Push 到 main 被拒绝（detached HEAD + remote 有更新） | 🟡 | 先 `git fetch origin main && git rebase origin/main` 再 push |
| 10 | 浮动 git dep 的历史残留（`{ git = "..." }` 无 tag/branch） | 🔴 | 所有依赖改为 `tag = "vX.Y.Z"` 固定 |

<a id="84-关键发现"></a>
### 8.4 关键发现

1. **所有消费者均从浮动 git dep 改为 tag 固定**是本轮升级的最大收益——消除了 6 个依赖"下次 `cargo update` 即可能断裂"的风险。

2. **ogsql-parser 的 31 个 patch 版本之间实际上非常稳定**。所有 4 个消费者在 31-version 跨度上的源代码均**零修改**通过编译。唯一的代码变更来自测试中的 struct 初始化（`body_start_line`）和 clippy 新 lint。

3. **`cargo build` 的并行能力被低估**。6 个消费者的 Phase 1 升级（fetch → checkout → edit Cargo.toml → cargo update → cargo build → commit → push）完全可以并行执行，互不等待。

4. **CI 是最后的守门员**。本地 `cargo build` 通过了 3 个 crate 的 ogexplain-analyzer，但 CI 的 `cargo build --workspace` 暴露了被遗漏的 ogexplain-optimizer。

<a id="85-推荐操作脚本模板"></a>
### 8.5 推荐操作脚本模板

以下脚本模板可用于下次 ogsql-parser 升级时快速启动：

```bash
#!/bin/bash
# 升级前：检查所有消费者当前 ogsql-parser pin
NEW_TAG="v0.8.31"  # 替换为目标版本
UMBRELLA="/Users/c2j/Projects/Desktop_Projects/DB/GaussDB_Heptadecagon"

echo "=== 当前 pin 状态 ==="
for d in ogexplain-analyzer metamorphosis codeweb astgrep CodeRoughcollie; do
  echo "--- $d ---"
  grep -rn 'ogsql-parser.*tag' $UMBRELLA/lib/$d/Cargo.toml $UMBRELLA/lib/$d/crates/*/Cargo.toml 2>/dev/null | sort -u
done

echo ""
echo "=== 隐蔽依赖检查 ==="
grep 'metamorphosis.*rev' $UMBRELLA/lib/ogexplain-analyzer/crates/ogexplain-optimizer/Cargo.toml

echo ""
echo "=== 各组件最新 tag ==="
for d in ogexplain-analyzer metamorphosis codeweb astgrep CodeRoughcollie; do
  echo -n "$d: "
  git -C $UMBRELLA/lib/$d tag --sort=-v:refname | head -3 | tr '\n' ' '
  echo ""
done

echo ""
echo "=== 操作步骤 ==="
echo "1. 对每个消费者: git checkout <latest-tag> → 改 pin → cargo build --workspace → commit → push"
echo "2. 同步 CodeRoughcollie 嵌套子模块 → cargo build --workspace"
echo "3. 每个消费者 bump 版本 +0.0.1 → tag → 等 CI"
echo "4. 本地验证: cargo fmt --all、cargo clippy --workspace -- -D warnings、cargo test --workspace"
```

---

<a id="9-常用维护命令"></a>
## 9. 常用维护命令

```bash
# 查看所有 submodule 当前 pin
git -C lib/ submodule foreach 'echo "$sm_path: $(git log -1 --format=%h)"'

# 查看所有消费者的 ogsql-parser tag pin
for d in ogexplain-analyzer metamorphosis codeweb astgrep CodeRoughcollie; do
  echo "=== $d ==="
  grep -rn 'ogsql-parser.*tag' lib/$d/Cargo.toml lib/$d/crates/*/Cargo.toml 2>/dev/null | grep -o 'tag = "[^"]*"' | sort -u
done

# 查看隐蔽依赖（ogexplain-optimizer 的 metamorphosis rev）
grep -A2 'metamorphosis.*rev' lib/ogexplain-analyzer/crates/ogexplain-optimizer/Cargo.toml

# 检查是否有浮动 git dep（无 tag/branch/rev）
grep -rn 'ogsql-parser.*git.*"' lib/*/Cargo.toml lib/*/crates/*/Cargo.toml 2>/dev/null | grep -v 'tag\|branch\|rev'

# 查看各组件远程最新 tag
for d in ogexplain-analyzer metamorphosis codeweb astgrep CodeRoughcollie; do
  echo -n "$d: "
  git -C lib/$d ls-remote --tags origin 2>/dev/null | tail -1 | awk '{print $2}'
done

# 单个消费者升级模板
cd lib/<component>
git fetch --tags origin
git checkout <latest-tag>
# 编辑 Cargo.toml 改 ogsql-parser tag
cargo update -p ogsql-parser
cargo build --workspace       # 始终用 --workspace 而非 -p
cargo test --workspace        # 验证测试代码
cargo clippy --workspace -- -D warnings
cargo fmt --all
git add -A && git commit -m "chore: bump ogsql-parser to vX.Y.Z"
git push origin HEAD:main
```

---

> **维护提示**：本文档的依赖关系数据提取自各仓库 `Cargo.toml` 的实际依赖声明。当任何仓库的依赖关系发生变化时，应同步更新本文档。每次 ogsql-parser 升级后，更新 §3 各组件详情中的"当前版本"和 §8 的实战记录。

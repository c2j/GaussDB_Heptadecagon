<div align="center">

# GaussDB Heptadecagon

**GaussDB / openGauss 开发者工具集**

[English](#english) | [中文](#中文)

</div>

---

<a id="中文"></a>

## 为什么叫"十七边形"？

1796 年，19 岁的高斯证明了**正十七边形**可以用尺规作图——这是两千年来该领域的第一个突破。他本人对此引以为傲，甚至要求在自己的墓碑上刻上正十七边形。

> 17 是一个费马素数：$2^{2^2} + 1 = 17$

本项目以"十七边形"为名，致敬高斯的精神：**用精确的工具攻克看起来不可能的问题**。GaussDB 生态的开发者工具长期碎片化、迁移方案陈旧、诊断能力薄弱——我们相信，一套精心构建的开源工具集，可以像高斯的证明一样，打开新的可能性。

---

## 这是什么？

GaussDB Heptadecagon 是围绕 GaussDB / openGauss 的 **一组开源开发者工具**（当前已发布8个，其余9个陆续开发中），覆盖从 SQL 解析、性能诊断、代码图谱到数据迁移的完整链路。

所有工具独立可用、各自迭代，同时通过共享的 SQL 解析器（`ogsql-parser`）形成有机整体。

---

## 工具总览

| 工具 | 定位 | 语言 | 一句话说明 |
|------|------|------|-----------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | 基石 | Rust | 手写递归下降 SQL 解析器，717 关键字、180+ 语句类型、1646 测试用例，openGauss/GaussDB 全方言覆盖 |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | 诊断 | Rust | EXPLAIN 执行计划解析 + 15+ 诊断规则，自动输出优化建议 |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | 重写 | Rust | SQL 语义重写引擎，基于 AST 的可插拔规则链，安全/条件/人工三级执行策略 |
| [**codeweb**](https://github.com/c2j/codeweb) | 图谱 | Rust | 跨 SQL + Java + MyBatis 的语义调用图，`Java 方法 → Mapper → SQL → 存储过程` 全链路追踪 |
| [**grep-excel**](https://github.com/c2j/grep-excel) | 数据 | Rust | 基于 DuckDB 的 Excel/CSV 搜索引擎，TUI 交互 + MCP Server 模式 |
| [**WDRProbe**](https://github.com/c2j/WDRProbe) | 运维 | Rust+TS | GaussDB WDR 报告桌面分析工具 |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | 迁移 | Python | GaussDB 存储过程 → Java + iBatis 服务自动转换 |
| [**SP-Complexity-Evaluator**](https://github.com/c2j/SP-Complexity-Evaluator) | 评估 | Java | SQL 和存储过程复杂度评估服务，支持 Oracle/Gauss/Hive 多方言 |

---

## 架构

```
                        ┌─────────────────────────────────────────┐
                        │          GaussDB Heptadecagon           │
                        └─────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
    ┌────▼─────┐              ┌────────▼────────┐          ┌────────▼────────┐
    │  迁移层   │              │    分析诊断层     │          │   发现与可视化层  │
    │          │              │                  │          │                 │
    │flux-gauss│              │ogexplain-analyzer│          │    codeweb      │
    │ SP-Eval  │              │  metamorphosis   │          │  grep-excel     │
    └──────────┘              └────────┬─────────┘          │   WDRProbe      │
                                      │                    └────────┬────────┘
                                      │                             │
                            ┌─────────▼─────────────────────────────▼──┐
                            │              基石层                        │
                            │          ogsql-parser                     │
                            │  手写递归下降 · 全方言 · AST · PL/pgSQL    │
                            └──────────────────────────────────────────┘
```

**ogsql-parser** 是整个工具集的基石——ogexplain-analyzer、metamorphosis、codeweb 均直接消费其 AST 输出。其他工具独立运行，但共享相同的设计哲学：精确、高性能、对 GaussDB 方言的一等支持。

---

## 各工具亮点

### ogsql-parser — SQL 解析器

整个工具集的地基。不依赖解析器生成器，纯手写递归下降。

- **717 个关键字**、180+ AST 语句类型
- 完整 PL/pgSQL 支持（DO、匿名块、游标、异常处理、GOTO、FORALL...）
- 完整 DDL/DDL 覆盖（CREATE TABLE、ALTER TABLE、PACKAGE、TRIGGER...）
- iBatis/MyBatis XML 解析 + Java 源码 SQL 提取
- MCP Server / HTTP API / TUI / CLI 四种接口
- SQL ↔ JSON 无损往返
- 1646 测试用例，openGauss 全量回归测试通过

### ogexplain-analyzer — 执行计划诊断

把 EXPLAIN 输出变成可操作的优化建议。

- 解析 TEXT 格式的 EXPLAIN / EXPLAIN ANALYZE 输出
- 15+ 条诊断规则：大表全扫、嵌套循环、排序溢出、下推失败、隐式类型转换、行列混合引擎...
- **openGauss 特有支持**：Vector 节点、CStore 列存扫描、Streaming 算子、FQS 下推检测
- 诊断 → 建议（cross-rule synthesis）：多条诊断自动合成优化建议
- CLI / TUI / 库三种使用方式，支持直接连接数据库执行 EXPLAIN
- 中英双语输出

### metamorphosis — SQL 重写引擎

基于 AST 的语义级 SQL 重写，不是字符串替换。

- 消费 ogsql-parser 的 AST，从不直接解析 SQL
- 可插拔规则链：Safe（自动执行）、Conditional（前置检查）、Manual（仅建议）
- 内置规则：`SELECT *` 展开为显式列、WHERE 等值条件唯一性探测...
- CLI + MCP Tool 接口
- 规则扩展只需实现 `DiagnosticRule` trait

### codeweb — 语义代码图谱

打通 `Java 方法 → MyBatis Mapper → SQL → 存储过程` 的完整调用链。

- **三层解析**：SQL 存储过程 CALL 关系 + MyBatis XML Mapper + Java 方法调用
- 双向查询：`callers()` / `callees()` / `trace()` / `impact()`
- 增量分析——只重新解析变更文件
- 导出 DOT / JSON / Mermaid / 浏览器交互可视化（Cytoscape.js）
- CGEF 格式导入/合并，支持跨团队图谱聚合

### grep-excel — Excel 搜索引擎

用 DuckDB 的力量搜索 Excel/CSV 文件。

- 全文搜索 / 精确匹配 / 通配符 / 正则，四种模式
- **直接执行 SQL**：`SELECT * FROM sheet_1_0 WHERE ...`
- MCP Server 模式：让 AI 助手直接操作你的 Excel
- 跨平台：Windows / macOS (Intel + Apple Silicon) / Linux (x64 + ARM64)

### WDRProbe — WDR 报告分析

GaussDB WDR（Workload Diagnosis Report）桌面分析工具。

- 解析 WDR 报告，提取关键性能指标
- 桌面应用（Tauri），本地运行，无需服务端

### flux-gauss — 存储过程迁移

GaussDB 存储过程 → Java + iBatis 服务代码自动转换。

- 自动将 PL/pgSQL 存储过程转换为 Java 服务层代码
- 生成 iBatis/MyBatis Mapper XML

### SP-Complexity-Evaluator — 复杂度评估

SQL 和存储过程的复杂度量化评分服务。

- 多维度评分：表数量、JOIN、子查询、循环嵌套、聚合函数...
- 支持 Oracle / Gauss / Hive 三种方言，可扩展
- REST API + Web 界面 + 批量 ZIP 处理 + Excel 导出
- 自定义权重表和高权重表/过程标记

---

## 技术栈

| 技术 | 使用场景 |
|------|---------|
| **Rust** | 7/8 工具的核心语言——ogsql-parser、ogexplain-analyzer、metamorphosis、codeweb、grep-excel、WDRProbe |
| **Java / Spring Boot** | SP-Complexity-Evaluator |
| **Python** | flux-gauss |
| **TypeScript** | WDRProbe 前端 |
| **DuckDB** | grep-excel 查询引擎 |
| **tree-sitter** | codeweb 的 Java 解析 |
| **Tauri** | WDRProbe 桌面框架 |
| **ratatui** | TUI 工具 |

---

## 我们在解决什么问题？

GaussDB / openGauss 生态目前面临的现实困境：

| 痛点 | 现状 | Heptadecagon 的回答 |
|------|------|-------------------|
| **迁移工具陈旧** | Oracle → GaussDB 的开源工具 4-5 年未更新 | flux-gauss 提供现代化的存储过程转换方案 |
| **性能诊断粗糙** | EXPLAIN 输出需要人工逐行分析 | ogexplain-analyzer 自动化 15+ 条诊断规则 |
| **代码关系黑盒** | 大型项目的 SQL-存储过程-Java 调用链不可见 | codeweb 构建完整的语义调用图 |
| **SQL 质量不可控** | 缺乏客观的 SQL 复杂度量化和重写工具 | metamorphosis + SP-Complexity-Evaluator |
| **方言差异大** | openGauss 特有语法（Streaming、CStore、Vector...）无专用解析器 | ogsql-parser 全方言覆盖，1646 测试 |

---

## 快速开始

每个工具都是独立仓库，可以直接克隆使用：

```bash
# 基石——SQL 解析器
git clone https://github.com/c2j/ogsql-parser.git
cd ogsql-parser && cargo build --release

# 执行计划诊断
git clone https://github.com/c2j/ogexplain-analyzer.git
cd ogexplain-analyzer && cargo build --release

# SQL 重写
git clone https://github.com/c2j/metamorphosis.git
cd metamorphosis && cargo build --release

# 代码图谱
git clone https://github.com/c2j/codeweb.git
cd codeweb && cargo build --release

# Excel 搜索
git clone https://github.com/c2j/grep-excel.git
cd grep-excel && cargo build --release

# 复杂度评估（Java）
git clone https://github.com/c2j/SP-Complexity-Evaluator.git
cd SP-Complexity-Evaluator && ./mvnw clean package
```

---

## 路线图

- [ ] **ogsql-parser**: 更多 GaussDB 特有语法覆盖、性能优化
- [ ] **metamorphosis**: 扩充重写规则库、添加 SQL 等价性验证
- [ ] **codeweb**: 支持更多语言（Kotlin、Go）、增强可视化
- [ ] **ogexplain-analyzer**: 新增分布式查询诊断规则
- [ ] **grep-excel**: 增量导入、更大文件支持
- [ ] **生态整合**: 统一 CLI 入口、跨工具工作流

---

## 参与建设

我们欢迎所有形式的贡献——代码、测试、文档、Bug 报告、功能建议。

### 如何参与

1. **试用**：Clone 任意工具，用你的真实场景测试
2. **反馈**：在对应仓库提 Issue，描述你遇到的场景和问题
3. **贡献代码**：Fork → Branch → PR，每个仓库都有 CONTRIBUTING 指引
4. **推荐新工具**：如果你发现 GaussDB 生态中缺失的工具，提 Issue 讨论

### 特别需要

- **试点用户**：在实际项目中使用这些工具，反馈真实体验
- **GaussDB 方言专家**：帮助完善 ogsql-parser 的语法覆盖
- **迁移场景**：提供 Oracle / MySQL → GaussDB 的真实迁移案例
- **Rust 开发者**：参与核心工具的开发和优化

---

## 许可证

所有工具均采用 **MIT License** 开源。

---

<a id="english"></a>

## Why "Heptadecagon"?

In 1796, at the age of 19, Carl Friedrich Gauss proved that a **regular heptadecagon** (17-sided polygon) could be constructed with compass and straightedge — the first advance in polygon construction in over 2,000 years. He considered it one of his proudest achievements.

> 17 is a Fermat prime: $2^{2^2} + 1 = 17$

We name this project "Heptadecagon" to honor that spirit: **using precise tools to solve seemingly impossible problems**. The GaussDB ecosystem has long suffered from fragmented tooling, outdated migration solutions, and weak diagnostics. We believe a well-crafted open-source toolchain can open new possibilities — just like Gauss's proof did.

---

## What Is This?

GaussDB Heptadecagon is a collection of **8 open-source developer tools** centered around GaussDB / openGauss, covering the full spectrum from SQL parsing, performance diagnostics, code graphing to data migration.

Each tool works independently while forming an organic whole through the shared SQL parser (`ogsql-parser`).

---

## Tool Overview

| Tool | Role | Lang | Summary |
|------|------|------|---------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | Foundation | Rust | Hand-written recursive descent SQL parser — 717 keywords, 180+ statement types, 1646 tests, full openGauss/GaussDB dialect coverage |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | Diagnostics | Rust | EXPLAIN plan parser + 15+ diagnostic rules with optimization suggestions |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | Rewriting | Rust | SQL semantic rewriting engine with pluggable rule chain on AST |
| [**codeweb**](https://github.com/c2j/codeweb) | Graph | Rust | Cross SQL + Java + MyBatis semantic call graph — full `Java Method → Mapper → SQL → Stored Procedure` chain tracing |
| [**grep-excel**](https://github.com/c2j/grep-excel) | Data | Rust | DuckDB-powered Excel/CSV search engine with TUI + MCP Server mode |
| [**WDRProbe**](https://github.com/c2j/WDRProbe) | Operations | Rust+TS | GaussDB WDR report desktop analysis tool |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | Migration | Python | GaussDB stored procedure → Java + iBatis service auto-conversion |
| [**SP-Complexity-Evaluator**](https://github.com/c2j/SP-Complexity-Evaluator) | Assessment | Java | SQL and stored procedure complexity scoring service with multi-dialect support |

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │          GaussDB Heptadecagon           │
                        └─────────────────────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
    ┌────▼─────┐              ┌────────▼────────┐          ┌────────▼────────┐
    │Migration │              │  Analysis &     │          │Discovery &      │
    │  Layer   │              │  Diagnostics    │          │  Visualization  │
    │          │              │                 │          │                 │
    │flux-gauss│              │ogexplain-analyzer│         │    codeweb      │
    │ SP-Eval  │              │  metamorphosis   │         │  grep-excel     │
    └──────────┘              └────────┬─────────┘         │   WDRProbe      │
                                      │                   └────────┬────────┘
                                      │                            │
                            ┌─────────▼────────────────────────────▼──┐
                            │              Foundation                  │
                            │          ogsql-parser                    │
                            │   Hand-written · Full dialect · AST     │
                            └─────────────────────────────────────────┘
```

**ogsql-parser** is the foundation — ogexplain-analyzer, metamorphosis, and codeweb all consume its AST output directly. The other tools operate independently but share the same design philosophy: precision, high performance, and first-class GaussDB dialect support.

---

## What Problems Are We Solving?

| Pain Point | Current State | Heptadecagon's Answer |
|------------|--------------|----------------------|
| **Outdated migration tools** | Oracle → GaussDB open-source tools are 4-5 years unmaintained | flux-gauss provides modern stored procedure conversion |
| **Crude performance diagnostics** | EXPLAIN output requires manual line-by-line analysis | ogexplain-analyzer automates 15+ diagnostic rules |
| **Invisible code relationships** | SQL-procedure-Java call chains are opaque in large projects | codeweb builds complete semantic call graphs |
| **Uncontrolled SQL quality** | No objective complexity quantification or rewriting tools | metamorphosis + SP-Complexity-Evaluator |
| **Dialect fragmentation** | No dedicated parser for openGauss-specific syntax | ogsql-parser with full dialect coverage, 1646 tests |

---

## Contributing

We welcome all forms of contribution — code, tests, documentation, bug reports, feature suggestions.

1. **Try it out**: Clone any tool and test with your real-world scenarios
2. **Give feedback**: Open issues in the respective repositories
3. **Contribute code**: Fork → Branch → PR
4. **Suggest new tools**: If you see a gap in the GaussDB ecosystem, open an issue to discuss

### We Especially Need

- **Pilot users**: Use these tools in real projects and share your experience
- **GaussDB dialect experts**: Help improve ogsql-parser syntax coverage
- **Migration scenarios**: Provide real Oracle / MySQL → GaussDB migration cases
- **Rust developers**: Join the development of core tools

---

## License

All tools are open-sourced under the **MIT License**.

---

<div align="center">

**用精确的工具，解决看起来不可能的问题。**

*Solve seemingly impossible problems with precise tools.*

</div>

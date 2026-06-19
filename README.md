<div align="center">

# GaussDB Heptadecagon

**GaussDB / openGauss 开发者工具集**

[English](#english) | [中文](#中文)

</div>

---

<a id="中文"></a>

## 为什么叫"Heptadecagon(十七边形)"？

1796 年，19 岁的高斯证明了**正十七边形**可以用尺规作图——这是两千年来该领域的第一个突破。他本人对此引以为傲，甚至要求在自己的墓碑上刻上正十七边形。

> 17 是一个费马素数：$2^{2^2} + 1 = 17$，同时，还有$2^3 + 3^2 = 17$

大模型擅长编程。然而，数据库领域编程并未像其他编程领域一样突飞猛进。本项目以"十七边形"为名，致敬高斯的精神：**用精确的工具攻克看起来不可能的问题**。GaussDB 生态的开发者工具长期碎片化、迁移方案陈旧、诊断能力薄弱——我们相信，一套精心构建的开源工具集，可以像高斯的证明一样，打开新的可能性。

---

## 这是什么？

GaussDB Heptadecagon 是围绕 GaussDB / openGauss 的 **一组开源开发者工具**（当前已发布11个，其余6个陆续开发中），覆盖从 SQL 解析、性能诊断、代码图谱到数据迁移的完整链路。

所有工具独立可用、各自迭代，同时通过共享的 SQL 解析器（`ogsql-parser`）形成有机整体。

---

## 工具总览

| 工具 | 定位 | 语言 | 一句话说明 |
|------|------|------|-----------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | 基石 | Rust | 手写递归下降 SQL 解析器，717 关键字、180+ 语句类型、1646 测试用例，openGauss/GaussDB 全方言覆盖 |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | 诊断 | Rust | EXPLAIN 执行计划解析 + 25 条诊断规则，自动输出优化建议，MCP Server 模式 |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | 重写 | Rust | SQL 语义重写引擎，基于 AST 的可插拔规则链，安全/条件/人工三级执行策略 |
| [**codeweb**](https://github.com/c2j/codeweb) | 图谱 | Rust | 跨 SQL + Java + MyBatis 的语义调用图，`Java 方法 → Mapper → SQL → 存储过程` 全链路追踪，MCP Server 模式 |
| [**grep-excel**](https://github.com/c2j/grep-excel) | 数据 | Rust | 基于 DuckDB 的 Excel/CSV 搜索引擎，TUI 交互 + MCP Server 模式 |
| [**WDRProbe**](https://github.com/c2j/WDRProbe) | 运维 | Rust+TS | GaussDB WDR 报告桌面分析工具 |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | 迁移 | Python | GaussDB 存储过程 → Java + iBatis 服务自动转换，MCP Server 模式 |
| [**SP-Complexity-Evaluator**](https://github.com/c2j/SP-Complexity-Evaluator) | 评估 | Java | SQL 和存储过程复杂度评估服务，支持 Oracle/Gauss/Hive 多方言 |
| [**rust-opengauss**](https://github.com/c2j/rust-opengauss) | 驱动 | Rust | GaussDB / openGauss 原生 Rust 驱动 + MCP Server，零 FFI 依赖，支持 SHA256/SM3 等国密认证 |
| [**astgrep**](https://github.com/c2j/astgrep) | 安全 | Rust | 多语言静态代码安全分析(SAST),Semgrep 风格规则 + 污点分析,GaussDB 方言走 ogsql-parser |
| [**CodeRoughcollie**](https://github.com/c2j/CodeRoughcollie) | 评审 | Rust | 静态 + 动态混合代码审核，消费 ogexplain-analyzer 28 条规则 + 真实 EXPLAIN + 复杂度评估 + astgrep 安全规则 |

---

## 架构

```
                          ┌─────────────────────────────────────────┐
                          │          GaussDB Heptadecagon           │
                          └─────────────────────────────────────────┘
                                         │
                      ┌──────────────────▼──────────────────────────┐
                      │               代码审核层                      │
                      │               CodeRoughcollie                │
                      │      静态 + 动态混合审核 · 集成全生态规则        │
                      └──────────────────┬──────────────────────────┘
                                         │
           ┌────────────────────┬────────┴───────────┬────────────────────┬────────────────────┐
           │                    │                    │                    │                    │
           │       迁移层        │     分析诊断层       │   发现与可视化层     │     安全扫描层       │
           │                    │                    │                    │                    │
           │     flux-gauss     │ ogexplain-analyzer │      codeweb       │      astgrep       │
           │      SP-Eval       │   metamorphosis    │     grep-excel     │                    │
           │                    │                    │      WDRProbe      │                    │
           │                    │                    │                    │                    │
           └────────────────────┴────────────────────┴────────────────────┴────────────────────┘
                                                     │
                      ┌──────────────────────────────▼─────────────────────────────┐
                      │                           基石层                            │
                      │               ogsql-parser · rust-opengauss                │
                      │           手写递归下降 · 全方言 · 原生驱动 · MCP               │
                      └────────────────────────────────────────────────────────────┘
```

**ogsql-parser** 是整个工具集的基石——ogexplain-analyzer、metamorphosis、codeweb、astgrep 均直接消费其 AST 输出。**rust-opengauss** 提供原生 Rust 驱动和 MCP Server，让 AI 助手直连 GaussDB/openGauss。**CodeRoughcollie** 是顶层集大成者——它消费 ogexplain-analyzer 的 28 条诊断规则、ogsql-complexity 复杂度评分、rust-opengauss 的真实 EXPLAIN 能力、astgrep 的安全规则，将全生态能力汇聚为一套代码审核流程。全部 11 个工具中有 **7 个已内置 MCP Server**（共 45 个 MCP 工具），AI 助手可覆盖从解析、诊断、重写、图谱、迁移到安全扫描的完整链路。其他工具独立运行，但共享相同的设计哲学：精确、高性能、对 GaussDB 方言的一等支持。

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
- 25 条诊断规则：大表全扫、嵌套循环、排序溢出、下推失败、隐式类型转换、行列混合引擎...
- **openGauss 特有支持**：Vector 节点、CStore 列存扫描、Streaming 算子、FQS 下推检测
- 诊断 → 建议（cross-rule synthesis）：多条诊断自动合成优化建议
- CLI / TUI / 库三种使用方式，支持直接连接数据库执行 EXPLAIN
- MCP Server 模式：5 个工具（诊断分析、计划解析、规则列表、优化建议、复杂度评分）
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
- MCP Server 模式：6 个工具（统计、节点列表、节点详情、调用链追踪、SQL 搜索、声明式查询）

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
- MCP Server 模式：`validate_sql` + `convert_sql`，AI 助手可直接调用存储过程转换

### SP-Complexity-Evaluator — 复杂度评估

SQL 和存储过程的复杂度量化评分服务。

- 多维度评分：表数量、JOIN、子查询、循环嵌套、聚合函数...
- 支持 Oracle / Gauss / Hive 三种方言，可扩展
- REST API + Web 界面 + 批量 ZIP 处理 + Excel 导出
- 自定义权重表和高权重表/过程标记

### rust-opengauss — 原生 Rust 驱动 + MCP Server

GaussDB / openGauss 的原生 Rust 数据库驱动，零 FFI 依赖（无 libpq、无 C 库）。

- 同步 + 异步双 API（`opengauss` / `tokio-opengauss`）
- 完整认证支持：SHA256、MD5+SHA256、SM3 国密、SCRAM-SHA-256、MD5
- MCP Server 模式：6 个工具（连接管理、表结构查询、SQL 执行、执行计划分析）
- 多连接配置 + OS 密钥链安全存储密码
- TLS 支持（disable / skip-verify / verify-full）

### astgrep — 静态代码安全分析

多语言静态代码安全分析工具（SAST），受 Semgrep 启发，基于 tree-sitter AST 与 YAML 声明式规则做模式匹配和污点分析。

- 10-crate workspace：core / ast / parser / matcher / dataflow / rules / cli / web / gui / test-utils
- 模式匹配 + 元变量捕获 + 污点分析（Source → Sink，支持 sanitizer）
- 多方言 SQL：GaussDB / OpenGauss → ogsql-parser；PolarDB-MySQL → sqlparser-rs
- 6 种输出格式：JSON / YAML / SARIF / Text / XML / HTML
- 三种接口：CLI（主）、Web（Axum REST API）、GUI（egui 桌面）
- 检测场景：SQL 注入、XSS、认证漏洞、敏感数据流
- Semgrep 兼容性努力进行中

### CodeRoughcollie — 代码评审

静态 + 动态混合代码审核工具，消费全生态能力，把零散的诊断、复杂度、安全扫描汇聚为一套统一审核流程。CLI 命令 `coderc`。

- **静态审核**：消费 ogexplain-analyzer 的 28 条诊断规则（25 经典 + 3 反模式），基于 AST 精确匹配 SQL 反模式；封装 astgrep 的 Java + SQL 安全规则
- **动态审核（核心竞争力）**：通过 rust-opengauss 连接 GaussDB 执行真实 `EXPLAIN (ANALYZE, COSTS, BUFFERS)`，获取实际代价与算子——这是任何纯静态 SAST 工具无法做到的
- **复杂度评估**：直接复用 ogsql-complexity 评分（`analyze()` / `gauss_analyze()`），与生态口径统一，支持基线增量对比
- **降级策略**：无数据库连接时自动回退静态分析，报告标注 `[EXPLAIN 降级：静态分析]`
- **13-crate workspace**：cr-core / cr-db / cr-audit-static / cr-audit-explain / cr-audit-complexity / cr-audit-impact / cr-plugin / cr-git / cr-config / cr-report / cr-mcp-server / cr-cli
- **三种报告格式**：Markdown / JSON / SARIF（适配 CI/CD 流水线）
- **退出码门禁**：Critical → exit 1（阻断 CI），Warning → exit 0（仅报告）
- **安全加固**：连接用户仅 `EXPLAIN` 权限、密码走 OS 密钥链、SQL 注入防护（ogsql-parser 预校验）
- **三期规划**：插件化规则体系（C ABI）、MCP Server（基于 rmcp）、语义影响分析（集成 codeweb）

---

## MCP 生态

11 个工具中已内置 **7 个 MCP Server**，共 **45 个 MCP 工具**，AI 助手（Claude Desktop、Cursor、VS Code Copilot Chat 等）可通过 stdio JSON-RPC 直接调用。

| 工具 | MCP 工具数 | 能力 |
|------|-----------|------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | 7 | SQL 解析、格式化、校验、JSON 往返、XML/Java SQL 提取 |
| [**rust-opengauss**](https://github.com/c2j/rust-opengauss) | 6 | 数据库连接、表结构查询、只读 SQL 执行、执行计划 |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | 5 | SQL 重写、探针建议、规则列表、等价性验证、Schema 提取 |
| [**grep-excel**](https://github.com/c2j/grep-excel) | 14 | Excel/CSV 导入、搜索、SQL 执行、数据读写、保存导出 |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | 5 | 执行计划诊断、计划解析、规则列表、优化建议、复杂度评分 |
| [**codeweb**](https://github.com/c2j/codeweb) | 6 | 图谱统计、节点列表、节点详情、调用链追踪、SQL 搜索、声明式查询 |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | 2 | 存储过程校验、存储过程转 Java 项目 |

所有 MCP Server 均基于 [rmcp](https://crates.io/crates/rmcp) Rust SDK 实现（flux-gauss 同时提供 Python 引擎），统一 stdio 传输，配置方式一致：

```json
{
  "mcpServers": {
    "gaussdb":  { "command": "/path/to/gaussdb-mcp" },
    "ogsql":    { "command": "/path/to/ogsql-mcp" },
    "ogexplain":{ "command": "/path/to/ogexplain-mcp" },
    "codeweb":  { "command": "/path/to/codeweb", "args": ["mcp", "--project", "/path/to/project"] },
    "metamorphosis": { "command": "/path/to/metamorphosis", "args": ["mcp"] }
  }
}
```

> **注**：astgrep 走 Axum REST API 路线，未内置 MCP Server，与上述 7 个 MCP 工具形成互补——适合 IDE 插件与 CI/CD 流水线集成场景。CodeRoughcollie（`cr-mcp-server`）MCP Server 正在三期开发中，届时将集成全生态审核能力。

### MCP 工具协作链路

```
gaussdb-mcp（执行 EXPLAIN）
      ↓
ogexplain-mcp（分析诊断 → 优化建议）
      ↓
metamorphosis-mcp（重写 SQL → 等价性验证）
      ↓
ogsql-parser-mcp（格式化 → 校验语法）
```

### 典型协作场景

下面三个场景展示大模型如何编排 7 个 MCP Server 完成真实开发任务——每一步都有工具兜底，大模型负责理解意图、编排流程、产出最终代码。

#### 场景一：编写 Java 服务 + iBatis Mapper SQL

**用户指令**：「帮我写一个订单查询的 Java 服务，带 iBatis Mapper XML」

```
1. rust-opengauss → 查询订单表结构（列名、类型、主键）
       ↓  大模型据此建立字段映射，不会写错列名
2. ogsql-parser → 解析项目中已有 Mapper XML 里的 SQL
       ↓  大模型学习团队既有风格（命名约定、动态 SQL 写法）
3. ogsql-parser → 校验自己生成的 SQL（GaussDB 方言语法）
       ↓  语法错误当场暴露，不带入代码库
4. metamorphosis → 探针检查（SELECT *、缺失 WHERE…）
       ↓  在落笔前消灭常见质量问题
5. ogexplain-analyzer → 获取优化建议（索引、连接方式…）
       ↓  生成的 SQL 天然规避全表扫描
6. 大模型输出 → Java Service + Mapper XML（已通过以上全部检查）
7. codeweb → 追踪调用链，确认 Java 方法 → Mapper → SQL 完整可达
```

**价值**：大模型不是"凭空生成"——它先看表结构、查既有风格、做语法校验和质量检查，最后才落笔。输出的代码天然符合 GaussDB 方言和团队规范。

#### 场景二：慢 SQL 优化闭环

**用户指令**：「这条查询太慢了，帮我看看怎么优化」

```
1. rust-opengauss → 对目标 SQL 执行 EXPLAIN，拿到执行计划
       ↓
2. ogexplain-analyzer → 解析执行计划，输出诊断（全表扫描、嵌套循环…）
       ↓  25 条规则自动分析，无需人工逐行读 EXPLAIN
3. ogexplain-analyzer → 合成优化建议（加索引、改连接、消除隐式转换…）
       ↓
4. metamorphosis → 按建议重写 SQL（Safe 规则自动执行）
       ↓  AST 级语义重写，不是字符串替换
5. metamorphosis → 等价性验证，确保重写前后结果一致
       ↓  只有效果不变的 SQL 才会交付
6. ogsql-parser → 格式化最终 SQL，交付给用户
```

**价值**：从 EXPLAIN → 诊断 → 建议 → 重写 → 验证 → 格式化，六个 MCP 工具自动接力，形成无人工干预的优化闭环。大模型全程编排，每一步都有工具兜底。

#### 场景三：存储过程 → Java + iBatis 迁移

**用户指令**：「把这个 GaussDB 存储过程迁移成 Java 服务」

```
1. flux-gauss → validate_sql  校验存储过程语法合法性
       ↓  语法问题提前发现，避免转换中途失败
2. flux-gauss → convert_sql  将 PL/pgSQL 转换为 Java + iBatis 项目
       ↓  自动生成 Service、Mapper XML、参数对象
3. ogsql-parser → 校验生成的 Mapper XML 中所有 SQL 语法
       ↓  确保转换后的 SQL 在 GaussDB 下可执行
4. codeweb → 构建调用图谱，验证 Java → Mapper → SQL 链路
       ↓  确认没有断裂的调用、没有遗漏的存储过程
5. astgrep → 扫描生成的 Java 代码安全风险（SQL 注入…）
       ↓  迁移产出的代码直接通过安全门禁
```

**价值**：flux-gauss 完成主体转换后，其余工具自动接管校验、图谱验证、安全扫描——迁移不再是"转完就完"，而是"转完即验证"。

---

## 技术栈

| 技术 | 使用场景 |
|------|---------|
| **Rust** | 10/11 工具的核心语言——ogsql-parser、ogexplain-analyzer、metamorphosis、codeweb、grep-excel、WDRProbe、rust-opengauss、astgrep、CodeRoughcollie |
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
| **性能诊断粗糙** | EXPLAIN 输出需要人工逐行分析 | ogexplain-analyzer 自动化 25 条诊断规则 |
| **代码关系黑盒** | 大型项目的 SQL-存储过程-Java 调用链不可见 | codeweb 构建完整的语义调用图 |
| **SQL 质量不可控** | 缺乏客观的 SQL 复杂度量化和重写工具 | metamorphosis + SP-Complexity-Evaluator |
| **方言差异大** | openGauss 特有语法（Streaming、CStore、Vector...）无专用解析器 | ogsql-parser 全方言覆盖，1646 测试 |
| **应用代码安全扫描缺失** | GaussDB 项目缺少针对 Java/Python + SQL 的统一安全扫描 | astgrep 提供多语言 SAST，GaussDB 方言一等支持 |
| **AI 工具无法直连数据库** | 缺少让 AI 助手安全访问 GaussDB 的标准接口 | 11 个工具中已内置 7 个 MCP Server（共 45 个工具），覆盖直连数据库、SQL 解析、诊断、重写、图谱、迁移、数据操作、安全扫描全链路 |
| **代码审核碎片化** | SQL 反模式、执行计划风险、复杂度、安全漏洞散落在不同工具，缺乏统一审核 | CodeRoughcollie 提供静态 + 动态混合审核，一键聚合全生态 28 条诊断规则 + 真实 EXPLAIN + 复杂度评估 + 安全扫描 |

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

# 原生 Rust 驱动 + MCP Server
git clone https://github.com/c2j/rust-opengauss.git
cd rust-opengauss && cargo build -p gaussdb-mcp

# 静态代码安全分析
git clone https://github.com/c2j/astgrep.git
cd astgrep && cargo build --release

# 代码评审（静态 + 动态混合审核）
git clone https://github.com/c2j/CodeRoughcollie.git
cd CodeRoughcollie && cargo build --workspace
```

---

## 路线图

- [ ] **ogsql-parser**: 更多 GaussDB 特有语法覆盖、性能优化
- [ ] **metamorphosis**: 扩充重写规则库、添加 SQL 等价性验证
- [ ] **codeweb**: 支持更多语言（Kotlin、Go）、增强可视化
- [ ] **ogexplain-analyzer**: 新增分布式查询诊断规则
- [ ] **grep-excel**: 增量导入、更大文件支持
- [ ] **astgrep**: 完成 Semgrep 兼容性、扩充 GaussDB 安全规则库
- [ ] **CodeRoughcollie**: 完成插件化规则体系（C ABI）、MCP Server、语义影响分析（集成 codeweb）
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

GaussDB Heptadecagon is a collection of **11 open-source developer tools** centered around GaussDB / openGauss, covering the full spectrum from SQL parsing, performance diagnostics, code graphing, data migration, to code review.

Each tool works independently while forming an organic whole through the shared SQL parser (`ogsql-parser`).

---

## Tool Overview

| Tool | Role | Lang | Summary |
|------|------|------|---------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | Foundation | Rust | Hand-written recursive descent SQL parser — 717 keywords, 180+ statement types, 1646 tests, full openGauss/GaussDB dialect coverage |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | Diagnostics | Rust | EXPLAIN plan parser + 25 diagnostic rules with optimization suggestions, MCP Server mode |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | Rewriting | Rust | SQL semantic rewriting engine with pluggable rule chain on AST |
| [**codeweb**](https://github.com/c2j/codeweb) | Graph | Rust | Cross SQL + Java + MyBatis semantic call graph — full `Java Method → Mapper → SQL → Stored Procedure` chain tracing, MCP Server mode |
| [**grep-excel**](https://github.com/c2j/grep-excel) | Data | Rust | DuckDB-powered Excel/CSV search engine with TUI + MCP Server mode |
| [**WDRProbe**](https://github.com/c2j/WDRProbe) | Operations | Rust+TS | GaussDB WDR report desktop analysis tool |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | Migration | Python | GaussDB stored procedure → Java + iBatis service auto-conversion, MCP Server mode |
| [**SP-Complexity-Evaluator**](https://github.com/c2j/SP-Complexity-Evaluator) | Assessment | Java | SQL and stored procedure complexity scoring service with multi-dialect support |
| [**rust-opengauss**](https://github.com/c2j/rust-opengauss) | Driver | Rust | Native GaussDB / openGauss Rust driver + MCP Server, zero FFI, SHA256/SM3 auth support |
| [**astgrep**](https://github.com/c2j/astgrep) | Security | Rust | Multi-language static code security analysis (SAST), Semgrep-style rules + taint analysis, GaussDB dialect via ogsql-parser |
| [**CodeRoughcollie**](https://github.com/c2j/CodeRoughcollie) | Review | Rust | Static + dynamic hybrid code review — consumes ogexplain-analyzer's 28 diagnostic rules + real EXPLAIN + complexity scoring + astgrep security rules |

---

## Architecture

```
                          ┌─────────────────────────────────────────┐
                          │          GaussDB Heptadecagon           │
                          └─────────────────────────────────────────┘
                                         │
                      ┌──────────────────▼──────────────────────────┐
                      │                    Review                    │
                      │                CodeRoughcollie               │
                      │      Static + dynamic hybrid · integrates    │
                      │              all ecosystem rules             │
                      └──────────────────┬──────────────────────────┘
                                         │
           ┌────────────────────┬────────┴───────────┬────────────────────┬────────────────────┐
           │                    │                    │                    │                    │
           │     Migration      │     Analysis &     │    Discovery &     │   Security Layer   │
           │                    │    Diagnostics     │   Visualization    │                    │
           │                    │                    │                    │                    │
           │     flux-gauss     │ ogexplain-analyzer │      codeweb       │      astgrep       │
           │      SP-Eval       │   metamorphosis    │     grep-excel     │                    │
           │                    │                    │      WDRProbe      │                    │
           │                    │                    │                    │                    │
           └────────────────────┴────────────────────┴────────────────────┴────────────────────┘
                                                     │
                      ┌──────────────────────────────▼─────────────────────────────┐
                      │                         Foundation                         │
                      │               ogsql-parser · rust-opengauss                │
                      │             Hand-written · Full dialect · MCP              │
                      └────────────────────────────────────────────────────────────┘
```

**ogsql-parser** is the foundation — ogexplain-analyzer, metamorphosis, codeweb, and astgrep all consume its AST output directly. **rust-opengauss** provides a native Rust driver and MCP Server, enabling AI assistants to connect directly to GaussDB/openGauss. **CodeRoughcollie** is the top integration layer — it consumes ogexplain-analyzer's 28 diagnostic rules, ogsql-complexity scoring, rust-opengauss's real EXPLAIN capability, and astgrep's security rules, unifying the entire ecosystem into a single code review workflow. **7 out of 11 tools now ship with built-in MCP Servers** (45 MCP tools total), giving AI assistants end-to-end coverage from parsing, diagnostics, rewriting, code graphing, migration, to security scanning. The other tools operate independently but share the same design philosophy: precision, high performance, and first-class GaussDB dialect support.

---

## Tool Highlights

### astgrep — Static Code Security Analysis

Multi-language static code security analysis tool (SAST), Semgrep-inspired, with tree-sitter AST pattern matching and taint analysis via YAML declarative rules.

- 10-crate workspace: core / ast / parser / matcher / dataflow / rules / cli / web / gui / test-utils
- Pattern matching + metavariable capture + taint analysis (Source → Sink with sanitizer support)
- Multi-dialect SQL: GaussDB / OpenGauss → ogsql-parser; PolarDB-MySQL → sqlparser-rs
- 6 output formats: JSON / YAML / SARIF / Text / XML / HTML
- Three interfaces: CLI (primary), Web (Axum REST API), GUI (egui desktop)
- Detection scenarios: SQL injection, XSS, authentication flaws, sensitive data flow
- Semgrep compatibility work in progress

### CodeRoughcollie — Code Review

Static + dynamic hybrid code review tool that aggregates the entire ecosystem's capabilities — diagnostics, complexity, security scanning — into one unified review workflow. CLI command `coderc`.

- **Static review**: consumes ogexplain-analyzer's 28 diagnostic rules (25 classic + 3 anti-pattern) for AST-based SQL antipattern detection; wraps astgrep's Java + SQL security rules
- **Dynamic review (core differentiator)**: connects to GaussDB via rust-opengauss to run real `EXPLAIN (ANALYZE, COSTS, BUFFERS)`, capturing actual cost and operators — something no pure static SAST tool can do
- **Complexity scoring**: directly reuses ogsql-complexity (`analyze()` / `gauss_analyze()`), keeping evaluation criteria consistent across the ecosystem, with baseline delta comparison
- **Graceful degradation**: automatically falls back to static analysis when no DB connection is available, annotating reports with `[EXPLAIN degraded: static analysis]`
- **13-crate workspace**: cr-core / cr-db / cr-audit-static / cr-audit-explain / cr-audit-complexity / cr-audit-impact / cr-plugin / cr-git / cr-config / cr-report / cr-mcp-server / cr-cli
- **Three report formats**: Markdown / JSON / SARIF (CI/CD pipeline ready)
- **Exit-code gating**: Critical → exit 1 (blocks CI), Warning → exit 0 (report only)
- **Security hardening**: DB user restricted to `EXPLAIN`-only, passwords via OS keychain, SQL injection prevention (ogsql-parser pre-validation)
- **Phase 3 roadmap**: plugin rule system (C ABI), MCP Server (rmcp-based), semantic impact analysis (codeweb integration)

---

## What Problems Are We Solving?

| Pain Point | Current State | Heptadecagon's Answer |
|------------|--------------|----------------------|
| **Outdated migration tools** | Oracle → GaussDB open-source tools are 4-5 years unmaintained | flux-gauss provides modern stored procedure conversion |
| **Crude performance diagnostics** | EXPLAIN output requires manual line-by-line analysis | ogexplain-analyzer automates 25 diagnostic rules |
| **Invisible code relationships** | SQL-procedure-Java call chains are opaque in large projects | codeweb builds complete semantic call graphs |
| **Uncontrolled SQL quality** | No objective complexity quantification or rewriting tools | metamorphosis + SP-Complexity-Evaluator |
| **Dialect fragmentation** | No dedicated parser for openGauss-specific syntax | ogsql-parser with full dialect coverage, 1646 tests |
| **Uncontrolled application code security** | GaussDB projects lack a unified security scanner for Java/Python + SQL | astgrep provides multi-language SAST with first-class GaussDB dialect support |
| **No AI-to-database bridge** | No standard interface for AI assistants to safely access GaussDB | 7 out of 11 tools ship with built-in MCP Servers (45 tools total), covering database connectivity, SQL parsing, diagnostics, rewriting, code graphing, migration, data operations, and security scanning |
| **Fragmented code review** | SQL antipatterns, execution plan risks, complexity, and security flaws are scattered across different tools with no unified review | CodeRoughcollie provides static + dynamic hybrid review, one-click aggregating all 28 ecosystem diagnostic rules + real EXPLAIN + complexity scoring + security scanning |

---

## MCP Ecosystem

**7 out of 11 tools ship with built-in MCP Servers**, exposing **45 MCP tools** in total. AI assistants (Claude Desktop, Cursor, VS Code Copilot Chat, etc.) can invoke them directly via stdio JSON-RPC.

| Tool | MCP Tools | Capabilities |
|------|-----------|-------------|
| [**ogsql-parser**](https://github.com/c2j/ogsql-parser) | 7 | SQL parsing, formatting, validation, JSON round-trip, XML/Java SQL extraction |
| [**rust-opengauss**](https://github.com/c2j/rust-opengauss) | 6 | Database connectivity, table metadata, read-only SQL execution, execution plans |
| [**metamorphosis**](https://github.com/c2j/metamorphosis) | 5 | SQL rewriting, probe suggestions, rule listing, equivalence verification, schema extraction |
| [**grep-excel**](https://github.com/c2j/grep-excel) | 14 | Excel/CSV import, search, SQL execution, data read/write, save/export |
| [**ogexplain-analyzer**](https://github.com/c2j/ogexplain-analyzer) | 5 | EXPLAIN diagnostics, plan parsing, rule listing, optimization suggestions, complexity scoring |
| [**codeweb**](https://github.com/c2j/codeweb) | 6 | Graph stats, node listing, node detail, call-chain tracing, SQL search, declarative queries |
| [**flux-gauss**](https://github.com/c2j/flux-gauss) | 2 | Stored procedure validation, stored procedure → Java project conversion |

All MCP Servers are built on the [rmcp](https://crates.io/crates/rmcp) Rust SDK (flux-gauss also provides a Python engine), using stdio transport with a consistent configuration pattern:

```json
{
  "mcpServers": {
    "gaussdb":   { "command": "/path/to/gaussdb-mcp" },
    "ogsql":     { "command": "/path/to/ogsql-mcp" },
    "ogexplain": { "command": "/path/to/ogexplain-mcp" },
    "codeweb":   { "command": "/path/to/codeweb", "args": ["mcp", "--project", "/path/to/project"] },
    "metamorphosis": { "command": "/path/to/metamorphosis", "args": ["mcp"] }
  }
}
```

> **Note**: astgrep uses an Axum REST API instead of an MCP Server, complementing the 7 MCP-enabled tools above. It is a good fit for IDE plugins and CI/CD pipeline integration. CodeRoughcollie (`cr-mcp-server`) MCP Server is in phase 3 development and will integrate all ecosystem review capabilities when ready.

### MCP Tool Collaboration

```
gaussdb-mcp (run EXPLAIN)
      ↓
ogexplain-mcp (diagnose → optimization suggestions)
      ↓
metamorphosis-mcp (rewrite SQL → verify equivalence)
      ↓
ogsql-parser-mcp (format → validate syntax)
```

### Typical Collaboration Scenarios

The three scenarios below show how an LLM orchestrates the 7 MCP Servers to accomplish real development tasks — every step is backed by a tool, while the LLM focuses on understanding intent, orchestrating the flow, and producing the final code.

#### Scenario 1: Writing a Java Service + iBatis Mapper SQL

**User**: "Write me an order query Java service with an iBatis Mapper XML"

```
1. rust-opengauss → query order table structure (columns, types, primary key)
       ↓  LLM builds accurate field mappings — no wrong column names
2. ogsql-parser → parse SQL from existing Mapper XMLs in the project
       ↓  LLM learns the team's existing style (naming, dynamic SQL patterns)
3. ogsql-parser → validate its generated SQL (GaussDB dialect syntax)
       ↓  syntax errors caught immediately, never reach the codebase
4. metamorphosis → probe check (SELECT *, missing WHERE…)
       ↓  common quality issues eliminated before writing
5. ogexplain-analyzer → get optimization suggestions (indexes, join strategies…)
       ↓  generated SQL avoids full-table scans from the start
6. LLM outputs → Java Service + Mapper XML (passed all checks above)
7. codeweb → trace call chain, confirm Java method → Mapper → SQL is complete
```

**Value**: The LLM doesn't "hallucinate" SQL — it reads the table schema, studies existing conventions, runs syntax validation and quality checks, then writes. The output naturally conforms to GaussDB dialect and team standards.

#### Scenario 2: Slow SQL Optimization Loop

**User**: "This query is too slow, help me optimize it"

```
1. rust-opengauss → run EXPLAIN on the target SQL, get the execution plan
       ↓
2. ogexplain-analyzer → parse the plan, output diagnostics (full scan, nested loop…)
       ↓  25 rules analyze automatically — no manual EXPLAIN reading
3. ogexplain-analyzer → synthesize optimization suggestions (indexes, joins, implicit casts…)
       ↓
4. metamorphosis → rewrite SQL per suggestions (Safe rules auto-applied)
       ↓  AST-level semantic rewrite, not string manipulation
5. metamorphosis → verify equivalence — rewritten SQL must produce same results
       ↓  only behavior-preserving SQL gets delivered
6. ogsql-parser → format final SQL and deliver to the user
```

**Value**: EXPLAIN → diagnose → suggest → rewrite → verify → format — six MCP tools chain automatically into a closed optimization loop with zero manual intervention. The LLM orchestrates throughout, with tools backing every step.

#### Scenario 3: Stored Procedure → Java + iBatis Migration

**User**: "Migrate this GaussDB stored procedure into a Java service"

```
1. flux-gauss → validate_sql  validate stored procedure syntax
       ↓  syntax issues caught early, avoiding mid-conversion failures
2. flux-gauss → convert_sql  convert PL/pgSQL to Java + iBatis project
       ↓  auto-generates Service, Mapper XML, parameter objects
3. ogsql-parser → validate all SQL in generated Mapper XMLs
       ↓  ensures converted SQL is executable under GaussDB
4. codeweb → build call graph, verify Java → Mapper → SQL chain
       ↓  no broken calls, no missing stored procedures
5. astgrep → scan generated Java code for security risks (SQL injection…)
       ↓  migration output passes security gates out of the box
```

**Value**: After flux-gauss completes the main conversion, other tools automatically take over validation, graph verification, and security scanning — migration becomes "convert and verify" rather than "convert and hope."

---

## Quick Start

Each tool is a standalone repository. Clone and build any of them:

```bash
# Foundation — SQL parser
git clone https://github.com/c2j/ogsql-parser.git
cd ogsql-parser && cargo build --release

# EXPLAIN diagnostics
git clone https://github.com/c2j/ogexplain-analyzer.git
cd ogexplain-analyzer && cargo build --release

# SQL rewriting
git clone https://github.com/c2j/metamorphosis.git
cd metamorphosis && cargo build --release

# Code graph
git clone https://github.com/c2j/codeweb.git
cd codeweb && cargo build --release

# Excel search
git clone https://github.com/c2j/grep-excel.git
cd grep-excel && cargo build --release

# Complexity evaluation (Java)
git clone https://github.com/c2j/SP-Complexity-Evaluator.git
cd SP-Complexity-Evaluator && ./mvnw clean package

# Native Rust driver + MCP Server
git clone https://github.com/c2j/rust-opengauss.git
cd rust-opengauss && cargo build -p gaussdb-mcp

# Static code security analysis
git clone https://github.com/c2j/astgrep.git
cd astgrep && cargo build --release

# Code review (static + dynamic hybrid)
git clone https://github.com/c2j/CodeRoughcollie.git
cd CodeRoughcollie && cargo build --workspace
```

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

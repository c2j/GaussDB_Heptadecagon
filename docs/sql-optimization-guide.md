# SQL 闭环优化流程指南

> **面向对象**：DBA、后端开发者、数据库性能工程师
>
> **目标**：使用 Heptadecagon 工具集（ogexplain-analyzer + metamorphosis + QED/VeriEQL）完成从诊断到重写、从验证到再评估的完整 SQL 优化闭环。

---

## 目录

- [1. 概述](#1-概述)
- [2. 前置准备](#2-前置准备)
- [3. 优化流程总览](#3-优化流程总览)
- [4. Phase 0 — 预检：建立可信基线](#4-phase-0--预检建立可信基线)
- [5. Phase 1 — 诊断：定位性能瓶颈](#5-phase-1--诊断定位性能瓶颈)
- [6. Phase 2 — 重写：应用优化建议](#6-phase-2--重写应用优化建议)
- [7. Phase 3 — 验证：保证语义等价](#7-phase-3--验证保证语义等价)
- [8. Phase 4 — 再评估：衡量优化效果](#8-phase-4--再评估衡量优化效果)
- [9. Phase 5 — 迭代：收敛或停止](#9-phase-5--迭代收敛或停止)
- [10. 实战示例](#10-实战示例)
- [11. 自动化循环（规划中）](#11-自动化循环规划中)
- [12. FAQ](#12-faq)

---

## 1. 概述

### 什么是闭环 SQL 优化

传统 SQL 优化是单向的：分析 EXPLAIN → 人工改写 → 期望变快。但这存在三个问题：

1. **改写可能改错**——SQL 语义变了，结果不一致
2. **改写可能无效**——改了但计划没变，或者变差了
3. **改写可能引入新问题**——解决了一个瓶颈，暴露了另一个

闭环优化的核心思想：**诊断 → 重写 → 验证 → 再评估 → 迭代**，每一步都有工具兜底，每一次改写都经过形式化验证，直到指标收敛。

### 工具角色分工

```
ogexplain-analyzer   诊断层：EXPLAIN → 28 条诊断规则 → findings + suggestions
metamorphosis        重写层：SQL AST → 规则引擎 → 重写后的 SQL
QED / VeriEQL        验证层：原 SQL vs 重写 SQL → 等价性证明
rust-opengauss       执行层：连接 DB → EXPLAIN ANALYZE → 真实执行计划
```

### 当前能力 vs 规划能力

| 能力 | 当前状态 | 规划中 |
|------|---------|--------|
| EXPLAIN 诊断（28 条规则） | ✅ 已实现 | 扩充至 45+ 条 |
| SQL 重写（14 条规则） | ✅ 已实现 | 新增诊断驱动规则 |
| QED 等价性证明 | ✅ 已实现 | 支持更复杂 SQL |
| VeriEQL 有界检查 | ✅ 已实现 | 增大 bound 上限 |
| 自动闭环迭代 | ❌ 手动 | `ogexplain optimize` 子命令 |
| 诊断 → 重写自动映射 | ❌ 手动映射 | 结构化 hint 传递 |

**当前阶段**：工具已就位，闭环需手动串联。本文档指导你完成手动闭环，同时描述自动化后的理想流程。

---

## 2. 前置准备

### 2.1 安装工具

```bash
# 诊断工具
git clone https://github.com/c2j/ogexplain-analyzer.git
cd ogexplain-analyzer && cargo build --release
# 二进制：target/release/ogexplain

# 重写工具
git clone https://github.com/c2j/metamorphosis.git
cd metamorphosis && cargo build --release
# 二进制：target/release/metamorphosis

# 数据库驱动（可选，用于直连 DB 执行 EXPLAIN）
git clone https://github.com/c2j/rust-opengauss.git
cd rust-opengauss && cargo build -p gaussdb-mcp --release
```

### 2.2 数据库连接配置

ogexplain-analyzer 的 `explain` 子命令可直接连接数据库：

```bash
# 创建配置文件 ~/.gaussdb-mcp.toml
cat > ~/.gaussdb-mcp.toml << 'EOF'
host = "192.168.1.100"
port = 5432
user = "query_user"
password = "keyring"        # "keyring" = 从 OS 密钥链读取
dbname = "production"
sslmode = "disable"
EOF

# 存储密码到 OS 密钥链（推荐）
gaussdb-mcp store-password
```

> **安全建议**：创建一个只读权限的数据库用户，仅授予 `EXPLAIN` 权限，避免优化过程中误执行写操作。

### 2.3 Schema 文件准备

metamorphosis 的部分规则（如 `eliminate-select-star`）需要 Schema 信息：

```bash
# 从数据库导出 Schema JSON
# 格式：{ "表名": { "列名": "类型" } }
cat > schema.json << 'EOF'
{
  "orders": {
    "order_id": "integer",
    "customer_id": "integer",
    "status": "numeric",
    "amount": "numeric",
    "created_at": "timestamp"
  },
  "customers": {
    "customer_id": "integer",
    "name": "varchar",
    "email": "varchar"
  }
}
EOF
```

QED 验证也需要 Schema（越详细越好，包含 PK/FK/NOT NULL 约束可提升证明效率）：

```bash
# 从 DDL 文件提取 Rich Schema（metamorphosis mcp 的 extract_schema 工具）
metamorphosis mcp
# 然后调用 extract_schema 工具，传入 DDL SQL 文件
```

---

## 3. 优化流程总览

```
┌──────────────────────────────────────────────────────────────────┐
│                    SQL 闭环优化流程                                │
│                                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐   │
│  │ Phase 0  │───→│ Phase 1  │───→│ Phase 2  │───→│ Phase 3  │   │
│  │ 预检     │    │ 诊断     │    │ 重写     │    │ 验证     │   │
│  │          │    │          │    │          │    │          │   │
│  │ ANALYZE  │    │ ogexplain│    │ metamor- │    │ QED /    │   │
│  │ EXPLAIN  │    │ analyze  │    │ phosis   │    │ VeriEQL  │   │
│  └──────────┘    └──────────┘    └──────────┘    └────┬─────┘   │
│                                                        │         │
│  ┌──────────┐    ┌──────────┐                         │         │
│  │ Phase 5  │←───│ Phase 4  │←────────────────────────┘         │
│  │ 迭代决策  │    │ 再评估   │                                   │
│  │          │    │          │                                   │
│  │ 收敛?    │    │ re-      │                                   │
│  │ 停止/继续 │    │ EXPLAIN  │                                   │
│  └──────────┘    └──────────┘                                   │
│         │                                                        │
│         ▼                                                        │
│  ┌──────────┐                                                   │
│  │  OUTPUT  │                                                   │
│  │ 优化SQL  │                                                   │
│  │ + 报告   │                                                   │
│  └──────────┘                                                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 4. Phase 0 — 预检：建立可信基线

### 为什么需要预检

统计信息过期 → 优化器选错计划 → 诊断结果不可信。**在诊断之前必须确保统计信息新鲜**。

### 4.1 检查统计信息

```bash
# 连接数据库，检查关键表的统计信息最后收集时间
psql -h 192.168.1.100 -U query_user -d production -c "
SELECT
    schemaname,
    relname   AS table_name,
    n_live_tup,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
WHERE relname IN ('orders', 'customers', 'order_items')
ORDER BY relname;
"
```

**判断标准**：
- `last_analyze` 超过 7 天，或 `n_live_tup` 与上次收集时差异 > 10% → 需要重新收集

### 4.2 收集统计信息（如需要）

```bash
# 对涉及的表执行 ANALYZE
psql -h 192.168.1.100 -U query_user -d production -c "
ANALYZE orders;
ANALYZE customers;
ANALYZE order_items;
"
```

### 4.3 获取基线 EXPLAIN

```bash
# 方式一：ogexplain 直连 DB 执行 EXPLAIN（推荐）
ogexplain explain \
  -s "SELECT * FROM orders o JOIN customers c ON o.customer_id = c.customer_id WHERE o.status = 1" \
  --analyze \
  --format json \
  -o baseline.json

# 方式二：手动执行 EXPLAIN，保存输出
psql ... -c "EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ..." > explain_output.txt

# 用 ogexplain 分析保存的文件
ogexplain analyze explain_output.txt --format json -o baseline.json
```

### 4.4 记录基线指标

从 `baseline.json` 的 `summary` 字段提取关键指标：

```bash
# 提取基线指标
python3 -c "
import json
with open('baseline.json') as f:
    data = json.load(f)
s = data.get('summary', {})
print('=== 基线指标 (Report₀) ===')
print(f'total_cost:       {s.get(\"total_cost\")}')
print(f'total_time_ms:    {s.get(\"total_time_ms\")}')
print(f'actual_rows:      {s.get(\"actual_rows\")}')
print(f'critical_count:   {s.get(\"critical_count\")}')
print(f'warning_count:    {s.get(\"warning_count\")}')
print(f'spill_kb:         {s.get(\"spill_kb\")}')
print(f'peak_memory_kb:   {s.get(\"peak_memory_kb\")}')
"
```

将这些数字记为 **Report₀**，后续每轮迭代都与它对比。

---

## 5. Phase 1 — 诊断：定位性能瓶颈

### 5.1 运行诊断

```bash
# ogexplain analyze 已在 Phase 0 执行
# 此步骤是理解诊断结果

# 文本报告（人类阅读）
ogexplain analyze explain_output.txt --format text

# JSON 报告（程序处理）
ogexplain analyze explain_output.txt --format json -o report.json

# 热力图（估算偏差可视化，需要 EXPLAIN ANALYZE）
ogexplain analyze explain_output.txt --format heatmap
```

### 5.2 理解诊断报告

诊断报告包含三个层次：

```
┌─────────────────────────────────────────────────────┐
│ 层次 1: Findings（逐条诊断）                          │
│                                                     │
│  每条 finding 包含:                                  │
│  · rule_id:     规则 ID（如 SCAN-001）               │
│  · severity:    严重级别（Critical/Warning/Info）    │
│  · category:    分类（ScanEfficiency/JoinStrategy…） │
│  · detail:      具体问题描述                         │
│  · suggestion:  优化建议文本                         │
│  · sql_rewrite: 结构化重写（仅 SUBQ-006）            │
├─────────────────────────────────────────────────────┤
│ 层次 2: Suggestions（跨规则综合建议）                  │
│                                                     │
│  多条 findings 自动合成为更高层建议:                    │
│  · IndexOptimization    → 建索引                     │
│  · StatisticsUpdate     → 更新统计                    │
│  · ConfigurationTuning  → 调参数                     │
│  · QueryRewrite         → 改写 SQL                   │
│  · DistributionOptim... → 分布式优化                  │
├─────────────────────────────────────────────────────┤
│ 层次 3: Summary（批量指标）                           │
│                                                     │
│  54 个指标字段，用于收敛检测                           │
└─────────────────────────────────────────────────────┘
```

### 5.3 按修复路径分类

拿到 findings 后，按以下表格分类，决定每条诊断走哪条修复路径：

| 诊断规则 | 修复路径 | 说明 |
|---------|---------|------|
| **SUBQ-001** 子查询未提升 | → metamorphosis 重写 | `subquery-to-join` 规则 |
| **SUBQ-006** 关联子查询自更新 | → ogexplain 自带重写 | `sql_rewrite` 字段已有可执行 SQL |
| **REW-001** 大 IN 列表 | → metamorphosis 重写 | `subquery-to-join` 规则 |
| **TYPE-001** 隐式类型转换 | → 手动修复（规划中） | 去掉引号或加 CAST |
| **TYPE-004** LIKE 前导通配符 | → 手动修复（规划中） | 考虑 pg_trgm + GIN |
| **SCAN-001** 大表全扫 | → DDL 建议 | `CREATE INDEX` |
| **SCAN-004** 无索引过滤 | → DDL 建议 | `CREATE INDEX` |
| **PART-001** 分区剪枝失效 | → DDL / SQL 调整 | 检查分区键函数 |
| **MEM-001** 排序溢出 | → 配置建议 | `SET work_mem` |
| **MEM-004** 峰值内存高 | → 配置建议 | `SET work_mem` |
| **JOIN-002** Hash 连接溢出 | → 配置建议 | `SET work_mem` |
| **AGG-002** HashAgg 溢出 | → 配置建议 | `SET work_mem` |
| **STATS-001** 统计未收集 | → 回到 Phase 0 | 执行 `ANALYZE` |
| **EST-001** 估算严重偏差 | → 回到 Phase 0 | 统计信息问题 |
| **PUSH-001** 查询未下推 | → 架构层（人工） | 需改写 SQL 消除下推阻碍 |
| **PUSH-002** 多层流式 | → 架构层（人工） | 分布式查询重构 |
| **NET-001** 广播大表 | → 架构层（人工） | 调整分布列 |
| **VEC-001** 行列混用 | → 架构层（人工） | 存储引擎统一 |
| **DIST-001** 分布列不匹配 | → 架构层（人工） | 分布列设计 |
| **SKEW-001** 数据倾斜 | → 数据探针 | `probe-data-skew` 探查分布 |

### 5.4 诊断→重写映射表

**SQL 可重写的诊断**（进入 Phase 2）：

| ogexplain 诊断 | 触发的 metamorphosis 规则 | Safety 级别 |
|----------------|--------------------------|-------------|
| SUBQ-001, REW-001 | `subquery-to-join` | Conditional |
| SUBQ-006 | ogexplain 自带 `sql_rewrite` | — |
| TYPE-001 | `add-explicit-cast` *(规划中)* | Safe |
| TYPE-004 | `suggest-trgm-index` *(规划中)* | Manual |

---

## 6. Phase 2 — 重写：应用优化建议

### 6.1 使用 ogexplain 自带重写（SUBQ-006）

```bash
# ogexplain 已在 analyze 时自动检测 SUBQ-006 并生成重写 SQL
# 从 JSON 报告中提取
python3 -c "
import json
with open('report.json') as f:
    data = json.load(f)
for finding in data.get('findings', []):
    if finding.get('rule_id') == 'SUBQ-006' and finding.get('sql_rewrite'):
        rw = finding['sql_rewrite']
        print('=== SUBQ-006 重写 ===')
        print(f'策略: {rw[\"strategy\"]}')
        print(f'重写SQL:')
        print(rw['rewritten_sql'])
        print(f'说明: {rw[\"explanation\"]}')
        # 保存
        with open('rewritten.sql', 'w') as out:
            out.write(rw['rewritten_sql'])
        print('已保存到 rewritten.sql')
        break
"
```

### 6.2 使用 metamorphosis 重写

```bash
# 方式一：按诊断映射选择规则
metamorphosis rewrite \
  --file slow_query.sql \
  --schema schema.json \
  --rules subquery-to-join \
  --input-format sql

# 方式二：应用所有 Safe 规则（自动执行语义等价重写）
metamorphosis rewrite \
  --file slow_query.sql \
  --schema schema.json \
  --input-format sql

# 方式三：生成建议而非直接重写（Manual 级规则）
metamorphosis suggest \
  --file slow_query.sql \
  --schema schema.json \
  --input-format sql
```

### 6.3 理解安全级别

| 安全级别 | 行为 | 使用场景 |
|---------|------|---------|
| **Safe** | 自动执行，数学上保证语义等价 | `UNION→UNION ALL`、`BETWEEN v AND v→=v` |
| **Conditional** | 检查前置条件后执行 | `子查询→JOIN`（需验证右表列 NOT NULL） |
| **Manual** | 仅生成建议 SQL，不自动替换 | 数据质量探针、安全告警 |

> **关键原则**：在闭环优化中，只接受经过 Phase 3 验证的重写。即使 metamorphosis 标记为 Safe 的规则，在第一次使用时也建议验证。

---

## 7. Phase 3 — 验证：保证语义等价

### 7.1 为什么需要验证

metamorphosis 的 Safe 规则理论上保证等价，但：
- 规则实现可能有 bug
- GaussDB 方言的特殊行为可能导致意外
- 复杂 SQL 组合可能暴露边界情况

**QED/VeriEQL 是最后一道防线**——用数学证明，而非信任。

### 7.2 两层验证策略

```
原 SQL + 重写 SQL
        │
        ▼
┌───────────────┐
│   VeriEQL     │     快速检查（秒级）
│  有界模型检查  │     对 ≤B 条元组验证等价性
│  OOPSLA'24    │     能发现反例 → 拒绝重写
└───┬───────┬───┘
    │       │
 EQ │       │ NEQ / UNKNOWN
    │       │
    │       ▼
    │   ┌───────────┐
    │   │    QED    │     完整证明（分钟级）
    │   │  SMT 求解  │     对所有可能数据证明等价
    │   │  (Z3)     │     数学保证
    │   └──┬────┬───┘
    │      │    │
    │   EQ │    │ NEQ / 超时
    │      │    │
    ▼      ▼    ▼
  接受    接受   拒绝
```

### 7.3 执行验证

```bash
# 方式一：metamorphosis verify 命令（推荐）
metamorphosis verify \
  --original slow_query.sql \
  --rewritten rewritten.sql \
  --schema schema.json \
  --engine qed \
  --timeout 60

# 方式二：通过 MCP 调用
# metamorphosis mcp → 调用 verify_equivalence 工具

# 方式三：先 VeriEQL 快速检查，再 QED 正式证明
# Step 1: VeriEQL 快速检查
metamorphosis verify \
  --original slow_query.sql \
  --rewritten rewritten.sql \
  --schema schema.json \
  --engine verieql \
  --bound 3

# Step 2: 通过后，QED 正式证明
metamorphosis verify \
  --original slow_query.sql \
  --rewritten rewritten.sql \
  --schema schema.json \
  --engine qed
```

### 7.4 理解验证结果

| 验证结果 | 含义 | 行动 |
|---------|------|------|
| **Equivalent** (EQ) | 数学证明两 SQL 等价 | ✅ 接受重写，进入 Phase 4 |
| **NotEquivalent** (NEQ) | 找到反例（具体数据导致不同结果） | ❌ 拒绝重写，检查反例修正 |
| **Unknown** | 超时或 SQL 超出支持范围 | ⚠️ 手动审查，或拆分 SQL 验证 |

**遇到 NotEquivalent 时**：

VeriEQL 会输出反例（哪些表的哪些行产生不同结果）。这是宝贵信息：

```bash
# 反例输出示例
# NotEquivalent:
#   counterexample:
#     orders: [{ order_id: 1, customer_id: NULL, status: 1 }]
#     customers: [{ customer_id: 1, name: 'Alice' }]
#
# 含义：当 orders.customer_id 为 NULL 时，
#       IN 子查询和 LEFT JOIN ... IS NULL 产出不同结果
#
# 行动：加 NULL 过滤条件后重新验证
```

---

## 8. Phase 4 — 再评估：衡量优化效果

### 8.1 对重写后的 SQL 重新执行 EXPLAIN

```bash
# 用重写后的 SQL 执行 EXPLAIN ANALYZE
ogexplain explain \
  -f rewritten.sql \
  --analyze \
  --format json \
  -o report_iter1.json
```

### 8.2 对比指标

```bash
python3 -c "
import json

def load_metrics(path):
    with open(path) as f:
        data = json.load(f)
    return data.get('summary', {})

baseline = load_metrics('baseline.json')
current  = load_metrics('report_iter1.json')

metrics = [
    ('total_cost',     '代价',       'lower is better'),
    ('total_time_ms',  '执行时间(ms)', 'lower is better'),
    ('critical_count', 'Critical数',  'lower is better'),
    ('warning_count',  'Warning数',   'lower is better'),
    ('spill_kb',       '溢出(KB)',    'lower is better'),
    ('peak_memory_kb', '峰值内存(KB)', 'lower is better'),
]

print(f'{\"指标\":<20} {\"基线\":>12} {\"当前\":>12} {\"变化\":>12} {\"方向\":<8}')
print('-' * 70)
for key, label, direction in metrics:
    old = baseline.get(key)
    new = current.get(key)
    if old is None or new is None:
        print(f'{label:<20} {str(old):>12} {str(new):>12} {\"N/A\":>12}')
        continue
    if old == 0:
        pct = 'N/A'
    else:
        pct = f'{((new - old) / old) * 100:+.1f}%'
    good = (new <= old) if 'lower' in direction else (new >= old)
    status = '✅' if good else '❌'
    print(f'{label:<20} {old:>12.1f} {new:>12.1f} {pct:>12} {status}')
"
```

### 8.3 指标对比解读

| 场景 | 解读 | 行动 |
|------|------|------|
| cost 下降、critical 减少 | ✅ 改善有效 | 继续 Phase 5，寻找下一个优化点 |
| cost 下降、但出现新的 critical | ⚠️ 改写引入新问题 | 分析新 finding，可能是下游瓶颈暴露 |
| cost 几乎不变 | ⚠️ 重写未生效 | 可能是诊断错误，或优化器未选择新计划 |
| cost 上升 | ❌ 退化 | 回滚到上一版本 SQL，停止迭代 |

---

## 9. Phase 5 — 迭代：收敛或停止

### 9.1 收敛决策表

| 条件 | 决策 | 说明 |
|------|------|------|
| `critical_count == 0` | ✅ **成功，停止** | 所有严重问题已解决 |
| 改善 < 5%，连续 3 轮 | ⏸ **平台期，停止** | 继续优化收益递减 |
| cost 上升 > 10% | ⏪ **回滚，停止** | 重写导致退化 |
| 已达第 10 轮 | ⏹ **预算用尽，停止** | 安全上限 |
| 剩余 findings 均为架构层 | ⏹ **超出 SQL 层面，停止** | 需人工介入 |

### 9.2 回滚策略

如果某轮重写导致退化：

```bash
# 保留每轮迭代的 SQL 版本
cp slow_query.sql      iteration_0_original.sql
cp rewritten_v1.sql    iteration_1.sql
cp rewritten_v2.sql    iteration_2.sql

# 如果 iteration_2 退化，回滚到 iteration_1
cp iteration_1.sql final_optimized.sql
```

### 9.3 迭代记录模板

每轮迭代记录以下信息：

```
迭代 #1
├── 输入: iteration_0_original.sql
├── 诊断: SUBQ-001 (子查询未提升)
├── 重写规则: subquery-to-join
├── 验证: QED Equivalent ✓
├── 指标变化: cost 25000→12000 (-52%), critical 3→1
├── 决策: 继续
└── 输出: iteration_1.sql
```

---

## 10. 实战示例

### 场景：慢查询优化

**原始 SQL**（订单查询，执行 234ms）：

```sql
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = '42'
  AND o.order_id IN (SELECT order_id FROM order_items WHERE amount > 100);
```

#### Phase 0：预检

```bash
# 检查统计信息 → 新鲜，无需 ANALYZE
# 获取基线 EXPLAIN
ogexplain explain -s "SELECT ..." --analyze --format json -o baseline.json
# 基线: cost=45000, time=234ms, critical=2, warning=3
```

#### Phase 1：诊断

```bash
ogexplain analyze explain_output.txt --format text
```

输出（关键部分）：

```
Findings:
  [TYPE-001] [Critical] 疑似隐式类型转换
    Seq Scan on orders 含过滤条件: (status = '42')
    过滤掉 500000 行 (共 500500 行)
    建议: WHERE status = 42 — 去掉引号或添加显式类型转换

  [SUBQ-001] [Warning] 子查询未提升
    SubqueryScan on order_items
    建议: 考虑将 IN 子查询改写为 JOIN

  [SCAN-001] [Warning] 大表全扫描
    Seq Scan on orders 扫描 500500 行
    建议: CREATE INDEX ON orders(status)
```

#### Phase 2：重写

```bash
# 手动修复 TYPE-001（status 列是 numeric，去掉引号）
# 同时用 metamorphosis 重写子查询

cat > fixed.sql << 'EOF'
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 42
  AND o.order_id IN (SELECT order_id FROM order_items WHERE amount > 100);
EOF

# 用 metamorphosis 重写 IN 子查询为 JOIN
metamorphosis rewrite \
  --file fixed.sql \
  --schema schema.json \
  --rules subquery-to-join \
  --input-format sql > rewritten.sql

# 结果：
# SELECT o.*, c.name
# FROM orders o
# JOIN customers c ON o.customer_id = c.customer_id
# JOIN order_items oi ON o.order_id = oi.order_id
# WHERE o.status = 42 AND oi.amount > 100;
```

#### Phase 3：验证

```bash
# VeriEQL 快速检查
metamorphosis verify \
  --original original.sql \
  --rewritten rewritten.sql \
  --schema schema.json \
  --engine verieql \
  --bound 3

# 输出: Equivalent ✓
```

#### Phase 4：再评估

```bash
ogexplain explain -f rewritten.sql --analyze --format json -o report_iter1.json
# 指标: cost=45000→8200 (-82%), time=234ms→45ms (-81%), critical=2→0
```

#### Phase 5：迭代决策

- critical_count == 0 → ✅ 成功
- 额外建议：创建索引 `CREATE INDEX ON orders(status)` 可进一步优化
- 但 SQL 层面已无 Critical 问题 → 停止迭代

#### 最终输出

```sql
-- 优化后 SQL（QED 验证等价）
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.status = 42 AND oi.amount > 100;
```

```
优化报告:
  迭代次数: 1
  改善: cost -82%, time -81%
  改写: 子查询→JOIN (subquery-to-join), 隐式类型转换修复
  验证: VeriEQL Equivalent ✓
  遗留建议: CREATE INDEX ON orders(status)
```

---

## 11. 自动化循环（规划中）

当前流程需要手动串联多个工具。规划中的 `ogexplain optimize` 子命令将实现一键闭环：

```bash
# 规划中的自动化命令
ogexplain optimize \
  --sql slow_query.sql \
  --config ~/.gaussdb-mcp.toml \
  --schema schema.json \
  --max-iterations 10 \
  --min-improvement 0.05 \
  --auto-analyze \
  --output-dir ./optimization_result/

# 自动执行:
# 1. 检查统计信息，必要时执行 ANALYZE
# 2. EXPLAIN ANALYZE → ogexplain 诊断
# 3. 按 diagnostic→rule 映射触发 metamorphosis 重写
# 4. QED/VeriEQL 验证等价性
# 5. re-EXPLAIN → 指标对比
# 6. 收敛检测 → 继续或停止
# 7. 输出: 优化SQL + 优化报告 + 证明证书
```

### 规划中的自动化能力

| 能力 | 描述 | 所需开发 |
|------|------|---------|
| `ogexplain optimize` 子命令 | 一键执行完整闭环 | ogexplain CLI |
| diagnostic→rule 映射引擎 | 自动将诊断结果映射到重写规则 | orchestrator |
| `RewriteContext.diagnostic_hints` | 将诊断上下文传递给重写引擎 | metamorphosis core |
| Finding 结构化字段 | `table`/`columns` 字段 | ogexplain core |
| 自动 ANALYZE 执行 | 统计过期时自动收集 | orchestrator |
| 收敛检测器 | 多维度指标对比 | orchestrator |
| 迭代报告生成器 | 每轮变化的可视化 | orchestrator |

---

## 12. FAQ

### Q: 每轮迭代都必须执行 EXPLAIN ANALYZE 吗？会不会很慢？

A: EXPLAIN ANALYZE 会实际执行查询。对于慢查询（秒级以上），每轮迭代确实需要等待。建议：
- 开发/测试环境执行完整 EXPLAIN ANALYZE
- 生产环境使用不带 ANALYZE 的 EXPLAIN（只看代价估算，不看实际执行）
- 或在只读副本上执行

### Q: QED 验证很慢（分钟级），有必要每轮都跑吗？

A: 不必要。建议分层策略：
- metamorphosis Safe 规则 → VeriEQL 快速检查即可（秒级）
- Conditional 规则 → VeriEQL + QED
- 诊断驱动的新型重写 → QED 完整证明
- 最终交付前 → QED 完整证明（无论规则级别）

### Q: 如果 VeriEQL 和 QED 都超时了怎么办？

A: 按以下顺序处理：
1. 简化 SQL（拆分子查询分别验证）
2. 减小 VeriEQL bound（从 3 降到 2）
3. 提供 Rich Schema（PK/FK 约束可加速证明）
4. 手动审查改写逻辑
5. 标记为"未经形式化验证"，需人工确认

### Q: 优化后 SQL 更快了，但执行计划完全变了，正常吗？

A: 正常。重写的目的就是改变执行计划。例如：
- 子查询 → JOIN：NestedLoop 可能变成 HashJoin
- 消除隐式转换：Seq Scan 可能变成 Index Scan
- 只要 QED 证明等价，计划变化是预期行为

### Q: 如果优化后反而变慢了？

A: 回滚到上一版本。可能原因：
- 统计信息不准确导致优化器选错计划（回到 Phase 0）
- 重写后的 SQL 形态触发了优化器 bug
- 分布式场景下，重写改变了数据分布策略

### Q: 能同时优化多条 SQL 吗？

A: 当前需逐条处理。规划中的批量模式（`ogexplain optimize --batch`）将支持：
- 多 SQL 文件批量优化
- CSV 模式输入输出
- 逐条生成优化报告

---

> **相关文档**
>
> - 架构设计：[闭环优化架构设计](closed-loop-optimization-design.md)
> - ogexplain-analyzer 用户手册：`ogexplain-analyzer/docs/UserGuide.md`
> - metamorphosis 用户手册：`metamorphosis/docs/UserGuide.md`
> - QED 理论文档：`metamorphosis/docs/QED.md`

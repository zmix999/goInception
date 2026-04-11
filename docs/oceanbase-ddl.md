# OceanBase DDL 原理与 goInception 适配详解

## 一、OceanBase DDL 基础概念

### Online DDL vs Offline DDL

OceanBase 基于 **LSM-Tree** 存储引擎，与 MySQL 的 B-Tree 更新就地写入不同，增量数据先写 MemTable，后台 Compaction 时合并到 SSTable。这个架构决定了 DDL 的行为：

```
MySQL DDL 三种算法：
┌──────────┐  ┌──────────┐  ┌──────────┐
│ INSTANT   │  │ INPLACE  │  │  COPY    │
│ 只改元数据 │  │ 引擎内重建 │  │ 建新表拷贝│
│ 不阻塞DML │  │ 短暂阻塞  │  │ 全程阻塞  │
└──────────┘  └──────────┘  └──────────┘

OceanBase DDL 两种类型：
┌──────────────────────────┐  ┌──────────────────────────┐
│  Online DDL               │  │  Offline DDL              │
│ 只改 schema 元数据         │  │ 需要重建表（table_id 变）  │
│ 数据补全推迟到 Compaction   │  │ 创建影子表 → 加表锁       │
│ DML 不受影响               │  │ → 数据重组 → 表交换       │
│                           │  │ DML 全程阻塞（TPS=0）     │
└──────────────────────────┘  └──────────────────────────┘
```

**Online DDL 的巧妙之处**：比如 `ADD COLUMN`，OceanBase 只修改 schema 元数据，不立即改已有数据。新写入的行用新 schema（带默认值），旧行在下次 Compaction 时自然补全。这是 LSM-Tree 架构的天然优势。

**Offline DDL 的代价**：相当于 MySQL 的 COPY 算法。创建影子表 → 加**表锁**（阻塞所有 DML）→ 数据重组写入影子表 → 替换原表。整个过程中 TPS/QPS 降到 0。

### 操作分类表

| 操作 | 类型 | 原因 |
|------|------|------|
| ADD COLUMN（末尾） | **Online** | 只改元数据，Compaction 补全 |
| ADD COLUMN（FIRST/AFTER）| OB ≥4.2.5 **Online**，<4.2.5 **Offline** | 4.2.5 优化了中间插列 |
| DROP COLUMN | **Offline** | 需要物理删除数据 |
| RENAME COLUMN | **Online** | 纯元数据 |
| MODIFY COLUMN 改类型（如 CHAR→VARCHAR） | **Offline** | 需要数据转换重写 |
| MODIFY COLUMN 扩类型（如 INT→BIGINT） | **Online** | 兼容扩展，不需重写 |
| CREATE INDEX | **Online** | 后台异步构建，DML 不阻塞 |
| ALTER PARTITION | **Offline** | 需要数据重分布 |
| DROP/TRUNCATE PARTITION | **Online** | 不需要数据重组 |
| ADD COMMENT | **Online** | 纯元数据 |
| 改主键 | **Offline** | 全表重建 |

### 与 MySQL 的关键差异

| 维度 | MySQL | OceanBase |
|------|-------|-----------|
| 存储引擎 | B-Tree，就地更新 | LSM-Tree，追加写入 |
| Online DDL 锁行为 | INPLACE 算法开头/结尾短暂 MDL 锁 | 完全不加表锁 |
| Offline DDL 锁行为 | COPY 算法全程锁 | 全程表锁，TPS=0 |
| 索引构建 | 并行排序 | 分布式排序 + 旁路写入（绕过 MemTable 直接写 SSTable），比 MySQL 快 3-4 倍 |
| Schema 协调 | 单节点，无需协调 | 多版本 Schema，落后节点限制 DDL 事务但不杀进程 |
| 前缀索引列 | 可修改 | **不可修改** |
| 主键 | 可后加 | 部分版本**必须建表时指定** |
| 触发器 | 支持 | **不支持**（导致 pt-osc 原版不可用） |
| SEQUENCE | 不支持 | 支持 Oracle 风格序列 |

### OceanBase 4.x DDL 改进

- **旁路写入机制（V4.0+）**：DDL 期间数据绕过 MemTable，通过 Paxos 复制直接写入 SSTable，性能比 V3.x 提升约 5 倍
- **分布式执行计划重设计**：采样+扫描和排序+扫描两个子计划跨多节点并行流水线执行
- **V4.3+ 增强**：行列存在线转换、快速中间列添加、扩展并行 DDL 操作

---

## 二、goInception 中的 OceanBase DDL 审计流程

### 整体判断逻辑

```
ALTER TABLE 进来
       │
       ▼
① checkAlterUseOsc(table)
   表大小 >= osc_min_table_size 且 osc 开关打开？
       │
   是 → useOsc = true        否 → useOsc = false（小表直接执行）
       │
       ▼
② checkDDLInstant(node, table)              ← 只对 MySQL 生效
   MySQL 5.7/8.0 能用 INSTANT 算法？
       │
   是 → useOsc = false（MySQL 原生秒级完成）
       │
       ▼
③ checkDDLOffline(node, table)              ← 只对 OceanBase 生效
   遍历所有子操作，统计 OfflineSpecs / OnlineSpecs
       │
   OfflineSpecs > 0  → useOsc = true （走 pt-osc）
   OfflineSpecs == 0 → useOsc = false（OB 原生 online DDL，不锁表）
       │
       ▼
④ 最终执行
   useOsc == true  → 走 pt-osc 或 gh-ost
   useOsc == false → 直接执行 DDL
```

### Offline DDL 分类逻辑（checkDDLOfflineOperation）

```
ALTER TABLE t 的每个子操作
        │
        ▼
┌─ MODIFY COLUMN 改了类型？         ──→ OfflineSpecs++
├─ 有 AUTO_INCREMENT？              ──→ OfflineSpecs++
├─ 有 GENERATED COLUMN？            ──→ OfflineSpecs++
├─ 有 FIRST/AFTER 位置？            ──→ OB < 4.2.5 时 OfflineSpecs++
├─ DROP COLUMN？                    ──→ OfflineSpecs++
├─ ALTER PARTITION？                ──→ OfflineSpecs++
├─ 改字符集（CONVERT TO）？          ──→ OfflineSpecs++
├─ ADD COLUMN 各选项                ──→ 按具体选项分 Online/Offline
├─ 其他列操作                       ──→ OnlineSpecs++
└─ 其他表操作                       ──→ OnlineSpecs++

最终: OfflineSpecs > 0 → 走 pt-osc
      OfflineSpecs == 0 → OB 原生执行（不锁表）
```

### DDL 冲突检测（checkAlterOperationConflicts）

OceanBase 不允许在一条 ALTER TABLE 中混合某些操作组合：

```sql
-- MySQL OK，OceanBase 报错：
ALTER TABLE t
  MODIFY COLUMN a BIGINT,
  DROP INDEX idx_b;

-- 必须拆成两条：
ALTER TABLE t MODIFY COLUMN a BIGINT;
ALTER TABLE t DROP INDEX idx_b;
```

goInception 使用位掩码检测以下冲突组合并在审计阶段拦截：
- 改列类型 + 删索引 → 拒绝
- 删列 + 删索引 → 拒绝
- 改列/加列 + 改主键 → 拒绝

### 前缀索引列保护

```sql
-- 假设 col 有前缀索引 INDEX idx(col(10))
ALTER TABLE t MODIFY COLUMN col VARCHAR(200);
-- MySQL: OK
-- OceanBase: 报错！
```

goInception 在审计时扫描表的所有索引，如果目标列参与了前缀索引就拦截。

---

## 三、pt-osc 的 OceanBase 适配

### 为什么需要 pt-osc

OceanBase 的 Offline DDL 会**锁全表**。对于生产大表，可能意味着几分钟甚至更久的业务中断。pt-osc 通过"建新表 → 逐行拷贝 → 交换"的方式避免长时间锁表。

### 适配要点

OceanBase 不是传统主从架构，很多 MySQL 概念不存在：

| MySQL pt-osc 参数 | OceanBase 适配 | 原因 |
|---|---|---|
| `--critical-load Threads_running` | 移除 | OB 无此 status 变量 |
| `--max-lag` | 移除 | OB 无主从复制延迟 |
| `--recurse=1` | 移除 | OB 无从库可检查 |
| `--check-plan` | 加 `--nocheck-plan` | OB 执行计划不兼容 |
| `--analyze-before-swap` | 加 `--noanalyze-before-swap` | OB 不需要 |
| （新增）`--table_mode` | 拷贝时设为 "queuing"，完成后改回 "normal" | 利用 LSM-Tree 优化批量写入 |
| （新增）`--degree` | 控制并行度 | OB 分布式统计收集 |

### TABLE_MODE 优化原理

pt-osc 拷贝数据时，把影子表设为 "queuing" 模式（优化批量写入的 Compaction 策略），拷贝完交换前再改回 "normal"（优化正常读写）。这是利用 OceanBase LSM-Tree 特性的针对性优化。

### 表大小查询适配

MySQL 直接查 `information_schema.tables`，但 OceanBase 的这个视图不准确。改为查 OB 内部视图：

```sql
SELECT /*+ READ_CONSISTENCY(WEAK) */ SUM(data_size)
FROM oceanbase.DBA_OB_TABLE_LOCATIONS l
JOIN oceanbase.DBA_OB_TABLET_REPLICAS r
  ON l.tablet_id = r.tablet_id
WHERE l.table_name = ? AND l.database_name = ?
```

用 `READ_CONSISTENCY(WEAK)` hint 避免强一致性读的开销。

---

## 四、gh-ost 的 OceanBase 适配情况

**当前未做任何 OceanBase 适配。**

gh-ost 代码中所有参数都按 MySQL 主从架构硬编码：

| 参数 | 问题 |
|------|------|
| `--max-lag-millis` | OB 无主从延迟概念 |
| `--assume-master-host` | OB 无主从概念 |
| `--critical-load Threads_running` | OB 无此变量 |
| `--replication-lag-query` | OB 无复制延迟查询 |

gh-ost 理论上可以适配 OceanBase（因为它用 binlog 解析而非 trigger，OB 的 Binlog Service 兼容 MySQL binlog 协议），但实际适配工作尚未完成。如需使用 gh-ost，需要补一套类似 pt-osc 的参数适配逻辑。

---

## 五、OceanBase 特有语法支持

### 序列（SEQUENCE）

OceanBase 支持 Oracle 风格的序列，goInception 支持两种语法：

```sql
INSERT INTO t VALUES (seq.NEXTVAL, 'data');      -- 点号风格
INSERT INTO t VALUES (NEXTVAL('seq'), 'data');    -- 函数调用风格
```

审计时会查询 `DBA_SEQUENCES` 确认引用的序列是否存在。

### 版本检测

OceanBase 版本格式：`OceanBase_CE-v4.2.5.0` → 解析为整数 `42500`

关键版本阈值：
- `< 42500`（4.2.5 之前）：ADD COLUMN 带 FIRST/AFTER 是 offline DDL
- `< 43100`（4.3.1 之前）：不支持全文索引

---

## 参考资料

- [OceanBase Online/Offline DDL (V4.3.3)](https://en.oceanbase.com/docs/common-oceanbase-database-10000000001784252)
- [OceanBase Online/Offline DDL (V4.2.1)](https://en.oceanbase.com/docs/common-oceanbase-database-10000000001106314)
- [How to Make DDL Execution Efficient in a Distributed Database](https://en.oceanbase.com/blog/4934724608)
- [OceanBase DDL vs MySQL DDL 分析](https://blog.csdn.net/OceanBaseGFBK/article/details/138126035)
- [OceanBase V4 ALTER TABLE DDL](https://blog.csdn.net/OceanBaseGFBK/article/details/139203749)
- [OceanBase Binlog Service](https://blog.csdn.net/OceanBaseGFBK/article/details/140102397)

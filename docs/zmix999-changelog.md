# zmix999 Fork 改造详解（50 commits）

基于 zmix999/goInception 相对于 hanchuanchuan/goInception 的 50 个 commit，改动 117 个文件，+37,737 行 / -16,580 行，从 2025-10 持续到 2026-04-10。

---

## 一、OceanBase 适配（12 commits）

让 goInception 从一个纯 MySQL 审计工具变成支持 OceanBase 的工具。

### 版本号解析（`9ed9a4c`）

MySQL 版本格式 `5.7.36`（3 段），OceanBase 格式 `OceanBase_CE-v4.2.5.0`（4 段）。原版解析器无法处理。

```go
// 原版：MySQL 3 段 → 50736
fmt.Sprintf("%s%02s%02s", parts[0], parts[1], parts[2])

// zmix999：OceanBase 4 段 → 42500
// 先按 "-" 拆出 "v4.2.5.0"，去掉 "v"，再按 "." 拆成 4 段
fmt.Sprintf("%s%s%s%02s", parts[0], parts[1], parts[2], parts[3])
```

版本整数 `42500` 用于后续所有版本特性判断（如 `< 43100` 不支持全文索引）。

### 序列支持（`fcf3441` + `f1e0e4f`）

OceanBase 支持 Oracle 风格序列：

```sql
INSERT INTO t VALUES (seq.NEXTVAL, 'data');      -- 点号风格
INSERT INTO t VALUES (NEXTVAL('seq'), 'data');    -- 函数调用风格
```

两个 commit 分别处理了：
1. 审计逻辑中遇到列名引用时查 `DBA_SEQUENCES` 确认是否为有效序列
2. Parser 中增加 `NEXTVAL(string)` 的函数调用语法

### DDL 冲突检测（`32e24b6`）

OceanBase 不允许某些 DDL 操作组合出现在一条 ALTER TABLE 中：

```sql
-- MySQL OK，OceanBase 报错：
ALTER TABLE t MODIFY COLUMN a BIGINT, DROP INDEX idx_b;
```

用位掩码检测以下冲突并拦截：
- 改列类型 + 删索引
- 删列 + 删索引
- 改列/加列 + 改主键

### 前缀索引列保护（`731ea3c`）

OceanBase 不支持修改参与前缀索引的列，审计时扫描所有索引拦截。

### Offline DDL 检测 + pt-osc 集成（`fc861cb` + `1ad11e2` + `9fe6ae3`）

最核心的 3 个 commit，渐进式完善：

1. `fc861cb`：首版实现，遇到第一个 offline 操作即返回
2. `1ad11e2`：修正 DROP/TRUNCATE PARTITION 不应归类为 offline
3. `9fe6ae3`：重写为计数模式（OnlineSpecs/OfflineSpecs），支持混合 DDL 判断

### pt-osc 参数适配（`90c8c23` + `904cf77`）

- 移除 OB 不支持的参数（`Threads_running`、`--recurse`、`--max-lag`）
- 新增 OB 特有参数（`--table_mode`、`--degree`、`--nocheck-plan`）
- TABLE_MODE 优化：拷贝时设为 "queuing"，完成后改回 "normal"

### Binlog 模式检查（`cd3c691` → `cb41550` 回退）

尝试检查 OB 是否开启 binlog，后因逻辑不成熟回退。同时清理了影响 OB 的 DRDS 相关查询。

---

## 二、MySQL 8/9 兼容性（6 commits）

| commit | 做了什么 |
|--------|---------|
| `8b00d6f` | 支持 MySQL 8.0.20+ 的 binlog 压缩（`TRANSACTION_PAYLOAD_EVENT`） |
| `00772cd` | 支持 `PARTIAL_UPDATE_ROWS_EVENT`（`binlog_row_value_options=PARTIAL_JSON`） |
| `bfa575a` | **关键 bugfix**：MySQL 8 开启 binlog 压缩且事务批量 >1 时备份失败 |
| `1ee2ea7` | 兼容 MySQL 5 的 TLS RSA 密钥交换算法 |
| `1de9004` | MySQL 9 的 `explain_format` 改为 `TRADITIONAL`（MySQL 9 改了默认值） |
| `feb6d6f` | 修复 `hypergraph_optimizer` 和 `explain_format=TRADITIONAL` 的不兼容问题 |

没有这些改动，goInception 连 MySQL 8.0.20+ 的压缩 binlog 都解析不了，备份功能直接报错。MySQL 9 的 explain 也会崩。

---

## 三、依赖现代化（5 commits）

| commit | 从 → 到 |
|--------|---------|
| `ac62598` `94585c0` | **Go 1.22.1 → Go 1.24.0** |
| `df6b7a4` | **gorm v1 → gorm v2**（`jinzhu/gorm` → `gorm.io/gorm`），涉及备份数据库的全部 ORM 调用 |
| `40ffe66` | MySQL driver `v1.4.1` → `v1.9.3` |
| `0d14af8` | 升级 `go-mysql` 包，支持加密 binlog 并发解析 |

其他依赖同步升级：
- `golang.org/x/net` 0.23 → 0.38
- `google.golang.org/grpc` 1.62 → 1.66
- `prometheus` 客户端、`vitess`、`pingcap/parser` 等

原版依赖有已知安全漏洞（CVE），gorm v1 早已停止维护。

---

## 四、SQL Parser 增强（6 commits）

| commit | 新支持的语法 |
|--------|------------|
| `5db223a` | `DEFAULT(column_name)` 作为列默认值 |
| `6df78f0` | 存储过程中的 `GET DIAGNOSTICS` 语句 |
| `4c7006b` | CREATE INDEX 的 table hints |
| `b6fede7` | SELECT ... INTO 子句 |
| `745b8be` | `COLUMN BEFORE` 语法 |
| `2b33d6d` | 将 `NUMBER` 从关键字降级为非关键字（避免误报） |

原版 parser 基于 TiDB v2.1.1，很多新语法不认识，审计时直接报语法错误。

---

## 五、备份功能增强（4 commits）

| commit | 做了什么 |
|--------|---------|
| `a76859d` | 完善外键操作备份（原版外键操作不生成回滚语句） |
| `79f11aa` | 完善存储过程/函数的检查和备份 |
| `a3566570` | 增加备份解析 binlog 心跳参数（长时间 DDL 不超时） |
| `fdf7977` | gh-ost 1.1.8 的 checkpoint 和 metadata lock 检查参数 |

---

## 六、审计规则完善（5 commits）

| commit | 做了什么 |
|--------|---------|
| `63760bb` | 修复转换字符集语句没走 pt-osc/gh-ost |
| `1db7347` | INSERT 操作的数据类型校验（值与列类型不匹配时告警） |
| `c40a465` `6af14e6` `c57cf52` | 完善分区表检查（分区列校验、非分区表直接 ADD PARTITION 的校验） |
| `354389f` | 索引键长度限制只在 MySQL 下生效（避免 OB 误报） |
| `3510673` | 非分区表直接 ADD PARTITION 时报错 |

---

## 七、代码清理（2 commits）

| commit | 做了什么 |
|--------|---------|
| `ca102b9` | 清理 DRDS 相关代码（阿里 DRDS 已下线，属于死代码） |
| `10d71e6` | 删除 SET PERSIST 参数（开发不应通过工单修改数据库参数，且无法备份/回滚） |

---

## 改造时间线

```
2025-10 ~ 2025-11  基础适配：版本解析、序列支持、分区检查
2025-11 ~ 2025-12  审计规则：DDL 冲突检测、前缀索引保护
2026-01 ~ 2026-02  MySQL 8/9 兼容（binlog 压缩、TLS）
2026-03            核心功能：offline DDL 检测 + pt-osc 集成
2026-03-30         持续优化：混合 DDL 计数、分区操作修正
2026-04            新功能：存储过程/函数备份、外键备份
```

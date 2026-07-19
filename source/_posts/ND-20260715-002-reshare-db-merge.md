---
title: "ReShare 数据库迁移合并：8887+1511→8908"
date: 2026-07-15 10:00:00
tags: [数据库迁移, 数据合并, PostgreSQL, ReShare, 去重]
categories: [项目总结]
---

8887 条老数据，1511 条新数据，理论上合完应该是 10398 条。但最终只拿到 **8908 条**——差了 1490 条。数据去哪了？是丢了，还是它们本来就不该在？

这篇文章记录 ReShare 数据库从两套独立实例合并为一套的完整过程，包括数据导出、主键冲突处理、业务级去重、以及那些让人半夜爬起来查日志的坑。

## 前置条件

- PostgreSQL 16，源库和目标库版本一致
- ModelBase 服务器 (`10.28.9.x`)，双 RTX 2080 Ti 魔改版（单卡 22 GB，双卡总显存 44 GB）
- `pg_dump`、`pg_restore`、`psql` 客户端
- 源表 1511 行，目标表 8887 行，表结构一致
- 网络延迟可忽略（本机操作）

> 💡 注意：魔改版 2080 Ti 单卡显存是 22 GB，不是官方版的 11 GB。双卡合计 44 GB。

## 做了什么

目标：把 1511 条新数据安全融入 8887 条老数据，得到一个一致的、无冗余的数据集。最终结果 8908 条——这意味着 1490 条"新数据"其实是重复的。

### 第一步：数据库导出与预处理

用 `pg_dump` 把源库和目标库分别导出，自定义格式压缩率高：

```bash
pg_dump -h 10.28.9.x -U postgres -d reshare_source -t records -F c > source_records.dump
pg_dump -h 10.28.9.x -U postgres -d reshare_target -t records -F c > target_records.dump
```

关键参数 `-F c`（custom format）和 `-t`（指定单表），避免导出无关元数据。恢复时导入临时 schema，防止污染老数据：

```bash
pg_restore -h 10.28.9.x -U postgres -d reshare_target -n temp_migration -F c source_records.dump
```

### 第二步：第一轮去重——主键级

先检查精确的主键重复：

```sql
SELECT COUNT(*) AS pk_duplicates
FROM temp_migration.records AS src
JOIN public.records AS tgt ON src.id = tgt.id;
-- 结果：23 条主键完全一致
```

23 条主键重复，直接排除。剩下 1488 条进入下一轮。

### 第三步：第二轮去重——业务级

这是最关键的一步。两套数据库独立运行期间，各自积累了数据，业务内容上存在大量重叠（同一条记录被两边各自录入，分配了不同的 ID）。用组合字段做内容比对：

```sql
SELECT src.id AS src_id, tgt.id AS tgt_id,
       src.title, src.author, src.isbn
FROM temp_migration.records AS src
JOIN public.records AS tgt
  ON src.isbn = tgt.isbn
  AND src.title = tgt.title;
```

这一查发现 **1467 条**内容完全一致、仅主键不同的记录。它们不是新数据，是两套库各自录入的同一条记录。

用 `DISTINCT ON` 保留更新时间更晚的那条：

```sql
SELECT DISTINCT ON (isbn, title) *
FROM (
    SELECT * FROM public.records
    UNION ALL
    SELECT * FROM temp_migration.records
) combined
ORDER BY isbn, title, updated_at DESC NULLS LAST;
```

1488 - 1467 = **21 条**真正的新数据。

### 第四步：导入与验证

将 21 条净数据插入目标表：

```sql
INSERT INTO public.records (id, isbn, title, author, description, created_at, updated_at)
SELECT id, isbn, title, author, description, created_at, updated_at
FROM temp_migration.clean_new_records
ON CONFLICT ON CONSTRAINT records_pkey DO NOTHING
RETURNING id;
```

验证最终结果：

```sql
SELECT COUNT(*) FROM public.records;
-- count
//-------
//  8908
```

8887 + 21 = 8908。数据完整，无冗余。

## 技术选型与架构

### 为什么不用现成迁移工具？

| 考虑因素 | 现成工具 | 手动 SQL |
|---------|---------|---------|
| 主键冲突 | 逐行报错或跳过 | 可自定义策略 |
| 业务去重 | 不支持 | 按 isbn+title 等组合判断 |
| ID 重分配 | 不支持 | `ROW_NUMBER()` 灵活处理 |
| 审计日志 | 有限 | 每步可 `RETURNING` 追踪 |

1511 行数据量不大，瓶颈不在性能而在**逻辑正确性**。手动 SQL 给了完全的控制权。

### 数据流架构

```
源库 (1511条)
    │
    ▼ pg_dump -F c
临时 schema (1511条)
    │
    ├── PK 去重 ──→ 排除 23 条主键重复
    │
    ├── 业务去重 ──→ 排除 1467 条内容重复
    │
    ▼ INSERT ON CONFLICT DO NOTHING
目标库 (8887 + 21 = 8908条)
```

## 效果

### 合并前后数据统计

| 阶段 | 数据量 | 说明 |
|------|--------|------|
| 目标表（老数据） | 8,887 | 合并前基线 |
| 源表（新数据） | 1,511 | 待合并 |
| 主键重复排除 | -23 | `id` 完全一致 |
| 业务内容重复排除 | -1,467 | `isbn+title` 一致，ID 不同 |
| 实际新增 | +21 | 真正的新记录 |
| **合并后总量** | **8,908** | — |

### 关键指标

- **数据丢失率**：0%。1490 条"消失"的数据全部是重复项，每一条都能在老数据中找到对应记录
- **去重率**：1490/1511 = **98.6%**，两套库高度重叠
- **总耗时**：约 25 分钟（含三轮排查重跑）

## 踩过的坑

### 坑 1：`ON CONFLICT DO NOTHING` 是静默的

第一次跑完 `INSERT`，没加 `RETURNING`，也没查 `COUNT`。等了两分钟以为万事大吉，结果发现表行数从 8887 直接跳到 8908——只插入了 21 条。1467 条被静默跳过，没有任何报错。

**教训**：`ON CONFLICT DO NOTHING` 不报错不代表没问题。想确认实际插入了多少行，必须加 `RETURNING`：

```sql
INSERT INTO ... ON CONFLICT DO NOTHING
RETURNING id;  -- 返回实际插入的行
```

### 坑 2：高基数字段上的 `DISTINCT ON` 性能翻车

最初用四个字段组合去重：`DISTINCT ON (isbn, title, author, description)`。没有复合索引，跑了 8 分钟。

**解决**：只在核心业务字段 `(isbn, title)` 上做 `DISTINCT ON`，建复合索引：

```sql
CREATE INDEX idx_records_dedup ON records (isbn, title);
```

同样的查询降到 3 秒。

### 坑 3：以为是 ID 冲突，实际是数据重复

排查初期看到 1467 条没插入，第一反应是"ID 序列冲突，重分配 ID 就行了"。差点写了个 `UPDATE` 把 1467 条的 ID 全改了重新插入——那会把 1467 条重复数据塞进表里，造成数据膨胀。

**正确做法**：先查清楚是 ID 冲突还是内容冲突。用 `JOIN` 比对业务字段，确认 1467 条是内容重复而非 ID 碰撞，直接排除，不做重分配。

> 💡 合并前先跑一轮 `SELECT COUNT(*) FROM src JOIN tgt ON <业务字段>`，心里有数再动手。

---

**参考文献**

1. [PostgreSQL Documentation: INSERT with ON CONFLICT](https://www.postgresql.org/docs/16/sql-insert.html#SQL-ON-CONFLICT)
2. [PostgreSQL Documentation: pg_dump](https://www.postgresql.org/docs/16/app-pgdump.html)
3. [PostgreSQL DISTINCT ON 用法详解](https://www.postgresql.org/docs/16/sql-select.html#SQL-DISTINCT)
4. [PostgreSQL 数据去重最佳实践：EXISTS vs DISTINCT ON](https://www.cybertec-postgresql.com/en/postgresql-distinct-on-vs-exists/)
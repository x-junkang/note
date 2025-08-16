# MySQL索引详解

## 目录
- [概述](#概述)
- [索引的分类](#索引的分类)
- [索引的数据结构](#索引的数据结构)
- [索引的创建和管理](#索引的创建和管理)
- [索引优化策略](#索引优化策略)
- [索引性能分析](#索引性能分析)
- [常见索引问题](#常见索引问题)
- [最佳实践](#最佳实践)
- [索引监控和维护](#索引监控和维护)

## 概述

索引是MySQL数据库中提高查询性能的关键技术。合理的索引设计可以显著提升查询速度，而不当的索引则可能导致性能下降。理解MySQL的索引机制对于数据库性能优化至关重要。

## 索引的分类

### 按存储结构分类

#### 1. B+树索引（默认）
- **特点**：平衡树结构，支持范围查询
- **适用场景**：等值查询、范围查询、排序、分组
- **存储引擎**：InnoDB、MyISAM等

#### 2. Hash索引
- **特点**：基于哈希表，查询速度快
- **适用场景**：等值查询，不支持范围查询
- **存储引擎**：Memory引擎

#### 3. Fulltext索引
- **特点**：全文搜索索引
- **适用场景**：文本内容的模糊搜索
- **存储引擎**：InnoDB、MyISAM

### 按功能分类

#### 1. 主键索引（Primary Key）
```sql
-- 创建表时指定主键
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100)
);

-- 或者单独添加主键
ALTER TABLE users ADD PRIMARY KEY (id);
```

#### 2. 唯一索引（Unique Index）
```sql
-- 创建唯一索引
CREATE UNIQUE INDEX idx_email ON users(email);

-- 或者创建表时指定
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE
);
```

#### 3. 普通索引（Normal Index）
```sql
-- 创建普通索引
CREATE INDEX idx_username ON users(username);

-- 或者创建表时指定
CREATE TABLE users (
    id INT PRIMARY KEY,
    username VARCHAR(50),
    INDEX idx_username (username)
);
```

#### 4. 复合索引（Composite Index）
```sql
-- 创建复合索引
CREATE INDEX idx_name_status ON users(last_name, first_name, status);

-- 或者创建表时指定
CREATE TABLE users (
    id INT PRIMARY KEY,
    last_name VARCHAR(50),
    first_name VARCHAR(50),
    status ENUM('active', 'inactive'),
    INDEX idx_name_status (last_name, first_name, status)
);
```

#### 5. 前缀索引（Prefix Index）
```sql
-- 创建前缀索引（只索引前10个字符）
CREATE INDEX idx_email_prefix ON users(email(10));
```

## 索引的数据结构

### B+树索引结构

```
                    [根节点]
                /              \
        [内部节点1]          [内部节点2]
       /          \         /          \
  [叶子节点1] [叶子节点2] [叶子节点3] [叶子节点4]
     |            |          |            |
  数据指针1    数据指针2   数据指针3    数据指针4
```

**特点：**
- 所有叶子节点在同一层
- 叶子节点包含所有数据
- 支持范围查询和排序
- 树的高度通常为3-4层

### 聚簇索引 vs 非聚簇索引

#### 聚簇索引（Clustered Index）
- **特点**：数据行按照索引顺序物理存储
- **数量限制**：每个表只能有一个聚簇索引
- **默认情况**：InnoDB中主键自动成为聚簇索引
- **优势**：范围查询性能好，避免随机I/O

#### 非聚簇索引（Secondary Index）
- **特点**：索引和数据分离存储
- **数量限制**：可以有多个
- **存储方式**：存储索引值和主键值
- **查询过程**：先查索引，再查主键，最后查数据

## 索引的创建和管理

### 创建索引

#### 1. 单列索引
```sql
-- 基本语法
CREATE INDEX index_name ON table_name (column_name);

-- 示例
CREATE INDEX idx_user_email ON users(email);
CREATE INDEX idx_order_date ON orders(order_date);
```

#### 2. 复合索引
```sql
-- 多列索引
CREATE INDEX idx_user_name_status ON users(last_name, first_name, status);

-- 函数索引（MySQL 8.0+）
CREATE INDEX idx_user_name_upper ON users(UPPER(last_name));
```

#### 3. 唯一索引
```sql
-- 唯一索引
CREATE UNIQUE INDEX idx_user_email ON users(email);

-- 唯一复合索引
CREATE UNIQUE INDEX idx_user_username_email ON users(username, email);
```

#### 4. 全文索引
```sql
-- 全文索引
CREATE FULLTEXT INDEX idx_article_content ON articles(title, content);

-- 使用全文索引搜索
SELECT * FROM articles 
WHERE MATCH(title, content) AGAINST('MySQL database' IN NATURAL LANGUAGE MODE);
```

### 删除索引

```sql
-- 删除索引
DROP INDEX index_name ON table_name;

-- 或者使用ALTER TABLE
ALTER TABLE table_name DROP INDEX index_name;

-- 示例
DROP INDEX idx_user_email ON users;
ALTER TABLE users DROP INDEX idx_user_name_status;
```

### 查看索引信息

```sql
-- 查看表的索引
SHOW INDEX FROM table_name;

-- 查看表的索引（详细信息）
SHOW INDEX FROM table_name\G

-- 查看表的创建语句
SHOW CREATE TABLE table_name;

-- 查看索引统计信息
SELECT 
    table_name,
    index_name,
    cardinality,
    sub_part,
    packed,
    null,
    index_type
FROM information_schema.statistics 
WHERE table_schema = 'your_database' 
  AND table_name = 'your_table';
```

## 索引优化策略

### 索引设计原则

#### 1. 最左前缀原则
```sql
-- 复合索引：idx_name_status (last_name, first_name, status)

-- 有效查询（使用索引）
SELECT * FROM users WHERE last_name = 'Smith';
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'John';
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'John' AND status = 'active';

-- 无效查询（不使用索引）
SELECT * FROM users WHERE first_name = 'John';  -- 跳过last_name
SELECT * FROM users WHERE status = 'active';    -- 跳过前面的列
```

#### 2. 选择性原则
```sql
-- 查看列的选择性
SELECT 
    COUNT(DISTINCT column_name) / COUNT(*) as selectivity
FROM table_name;

-- 示例：查看email列的选择性
SELECT 
    COUNT(DISTINCT email) / COUNT(*) as email_selectivity
FROM users;
```

#### 3. 覆盖索引
```sql
-- 创建覆盖索引
CREATE INDEX idx_user_info ON users(last_name, first_name, email, status);

-- 查询时使用覆盖索引（避免回表）
SELECT last_name, first_name, email, status 
FROM users 
WHERE last_name = 'Smith' AND status = 'active';
```

### 索引优化技巧

#### 1. 避免索引失效
```sql
-- 避免在索引列上使用函数
-- 错误示例
SELECT * FROM users WHERE YEAR(created_at) = 2023;

-- 正确示例
SELECT * FROM users WHERE created_at >= '2023-01-01' AND created_at < '2024-01-01';

-- 避免使用!=或<>操作符
-- 错误示例
SELECT * FROM users WHERE status != 'inactive';

-- 正确示例
SELECT * FROM users WHERE status IN ('active', 'pending', 'suspended');
```

#### 2. 合理使用OR条件
```sql
-- 避免在OR条件中使用不同列
-- 错误示例
SELECT * FROM users WHERE last_name = 'Smith' OR email = 'john@example.com';

-- 正确示例：使用UNION
SELECT * FROM users WHERE last_name = 'Smith'
UNION
SELECT * FROM users WHERE email = 'john@example.com';
```

#### 3. 索引列顺序优化
```sql
-- 将选择性高的列放在前面
-- 假设status的选择性比last_name低
CREATE INDEX idx_user_status_name ON users(status, last_name);

-- 查询时先过滤status，再过滤last_name
SELECT * FROM users WHERE status = 'active' AND last_name = 'Smith';
```

## 索引性能分析

### 使用EXPLAIN分析查询

#### 1. 基本EXPLAIN
```sql
-- 分析查询执行计划
EXPLAIN SELECT * FROM users WHERE last_name = 'Smith' AND status = 'active';

-- 输出字段说明：
-- id: 查询的标识符
-- select_type: 查询类型
-- table: 表名
-- type: 访问类型（system > const > eq_ref > ref > range > index > ALL）
-- possible_keys: 可能使用的索引
-- key: 实际使用的索引
-- key_len: 索引长度
-- ref: 索引的哪一列被使用了
-- rows: 预计扫描的行数
-- Extra: 额外信息
```

#### 2. 详细EXPLAIN
```sql
-- 获取更详细的执行计划
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE last_name = 'Smith';

-- 查看查询的优化器跟踪
SET optimizer_trace = 'on';
SELECT * FROM users WHERE last_name = 'Smith';
SELECT * FROM information_schema.optimizer_trace;
SET optimizer_trace = 'off';
```

### 索引使用情况分析

#### 1. 查看索引使用统计
```sql
-- 查看索引使用情况
SELECT 
    object_schema,
    object_name,
    index_name,
    count_read,
    count_write,
    count_fetch,
    count_insert,
    count_update,
    count_delete
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_database'
  AND object_name = 'your_table';
```

#### 2. 查看慢查询日志
```sql
-- 启用慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 2;  -- 2秒以上的查询记录

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';

-- 分析慢查询日志
mysqldumpslow /var/log/mysql/slow.log
```

## 常见索引问题

### 1. 索引过多
**问题**：过多的索引会增加写入开销和存储空间
**解决方案**：
```sql
-- 删除未使用的索引
-- 合并功能相似的索引
-- 定期分析索引使用情况
```

### 2. 索引失效
**常见原因**：
- 在索引列上使用函数
- 使用!=或<>操作符
- 字符串不加引号
- 使用OR连接不同列

**解决方案**：
```sql
-- 重写查询语句
-- 创建函数索引（MySQL 8.0+）
-- 使用UNION替代OR
```

### 3. 索引碎片
**问题**：删除和更新操作可能导致索引碎片
**解决方案**：
```sql
-- 重建索引
ALTER TABLE table_name ENGINE=InnoDB;

-- 或者使用OPTIMIZE TABLE
OPTIMIZE TABLE table_name;
```

## 最佳实践

### 1. 索引设计原则
- **选择性原则**：选择区分度高的列建立索引
- **最左前缀原则**：复合索引的列顺序很重要
- **覆盖索引**：尽量使用覆盖索引避免回表
- **适度原则**：不要过度索引，影响写入性能

### 2. 查询优化建议
```sql
-- 使用LIMIT限制结果集
SELECT * FROM users WHERE status = 'active' LIMIT 100;

-- 避免SELECT *
SELECT id, username, email FROM users WHERE status = 'active';

-- 为什么大数据量时推荐用EXISTS替代IN？
-- 因为IN会先将子查询结果全部取出再与外表比对，数据量大时效率低；
-- 而EXISTS在找到第一个匹配时就返回TRUE，通常能更快终止，尤其是外表数据量大时。
SELECT * FROM orders o 
WHERE EXISTS (
    SELECT 1 FROM users u 
    WHERE u.id = o.user_id AND u.status = 'active'
);
```

### 3. 定期维护
```sql
-- 定期分析表
ANALYZE TABLE table_name;

-- 定期优化表
OPTIMIZE TABLE table_name;

-- 定期检查索引使用情况
SELECT 
    table_name,
    index_name,
    cardinality
FROM information_schema.statistics 
WHERE table_schema = 'your_database';
```

## 索引监控和维护

### 1. 监控索引使用情况
```sql
-- 查看索引使用统计
SELECT 
    s.table_schema,
    s.table_name,
    s.index_name,
    s.cardinality,
    s.sub_part,
    s.packed,
    s.null,
    s.index_type
FROM information_schema.statistics s
WHERE s.table_schema = 'your_database'
ORDER BY s.table_name, s.index_name;
```

### 2. 监控索引性能
```sql
-- 查看索引I/O等待
SELECT 
    object_schema,
    object_name,
    index_name,
    count_star,
    sum_timer_wait,
    avg_timer_wait
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'your_database';
```

### 3. 索引维护脚本
```sql
-- 检查未使用的索引
SELECT 
    t.table_schema,
    t.table_name,
    s.index_name,
    s.cardinality
FROM information_schema.tables t
JOIN information_schema.statistics s 
    ON t.table_schema = s.table_schema 
    AND t.table_name = s.table_name
WHERE t.table_schema = 'your_database'
  AND s.cardinality = 0;  -- 基数为0的索引可能未使用
```

## 总结

MySQL索引是数据库性能优化的核心技术。合理的索引设计可以显著提升查询性能，而不当的索引则可能导致性能下降。在实际应用中，需要：

1. **理解索引原理**：掌握B+树索引的结构和特点
2. **合理设计索引**：遵循最左前缀原则和选择性原则
3. **持续优化**：定期分析索引使用情况，删除无用索引
4. **监控性能**：使用EXPLAIN分析查询计划，监控索引性能

通过科学的索引设计和持续的优化维护，可以充分发挥MySQL数据库的性能潜力。

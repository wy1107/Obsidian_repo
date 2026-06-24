## 1. 问题描述

在 PostgreSQL（兼容 Oracle 的 MogDB/OpenGauss 环境）中，使用 `pg_oracle."TIMESTAMP"` 类型定义的字段，存储的时间数据带有6位小数秒，例如：

2026-06-24 14:43:51.955000

期望格式为不含小数秒：

  
2026-06-24 14:43:51

## 2. 表定义示例

  
CREATE TABLE "AST_CUSMG_SIT"."CUST_MAIN" (  
    ...  
    "CREATE_TIME" pg_oracle."TIMESTAMP" NOT NULL,  
    "UPDATE_TIME" pg_oracle."TIMESTAMP" NULL,  
    ...  
);

`pg_oracle."TIMESTAMP"` 是 Oracle 兼容模式下的自定义类型，底层存储默认保留6位小数秒（微秒精度）。

## 3. 标准 TIMESTAMP 的处理方式

对于标准 PostgreSQL `TIMESTAMP` 类型，可直接在建表或 ALTER 时指定精度：

-- 建表时指定精度为0  
"CREATE_TIME" TIMESTAMP(0) NOT NULL  
​  
-- 已有表修改精度  
ALTER TABLE "schema"."table" ALTER COLUMN "column" TYPE TIMESTAMP(0);

`TIMESTAMP(0)` 表示不保留小数秒，输出格式为 `YYYY-MM-DD HH:MI:SS`。

## 4. pg_oracle.TIMESTAMP 的特殊性

### 4.1 information_schema 中的表现

`pg_oracle."TIMESTAMP"` 在 `information_schema.columns` 中的元数据如下：

|字段|值|说明|
|---|---|---|
|data_type|USER-DEFINED|非标准类型，标记为用户自定义|
|udt_name|TIMESTAMP|类型名称（注意：与标准 timestamp 的 udt_name 不同）|
|datetime_precision|**0**|**错误值！实际存储为6位精度，但元数据报告为0**|

对比标准 TIMESTAMP 的元数据：

|字段|值|
|---|---|
|data_type|timestamp without time zone|
|udt_name|timestamp|
|datetime_precision|6|

### 4.2 查询陷阱

**错误查询 — 按 datetime_precision > 0 查找：**

  
-- 此查询无法找到 pg_oracle.TIMESTAMP 列，因为其 datetime_precision 报告为 0  
SELECT table_name, column_name  
FROM information_schema.columns  
WHERE table_schema = 'AST_CUSMG_SIT'  
  AND data_type IN ('timestamp without time zone', 'timestamp with time zone')  
  AND datetime_precision > 0;

结果：**空，找不到任何列**。

**正确查询 — 按 udt_name 查找：**

  
SELECT table_name, column_name  
FROM information_schema.columns  
WHERE table_schema = 'AST_CUSMG_SIT'  
  AND udt_name = 'TIMESTAMP';

结果：**能找到所有 pg_oracle."TIMESTAMP" 类型的列**。

### 4.3 根因分析

`pg_oracle.TIMESTAMP` 是通过 `CREATE TYPE` 创建的自定义域类型（DOMAIN 或复合类型），`information_schema` 对自定义类型的精度信息报告不准确，`datetime_precision` 始终显示为0，无法反映实际存储精度。

## 5. 修复方案

### 5.1 查找所有需要修改的字段

  
SELECT table_name, column_name  
FROM information_schema.columns  
WHERE table_schema = 'AST_CUSMG_SIT'  
  AND udt_name = 'TIMESTAMP'  
ORDER BY table_name, column_name;

### 5.2 批量生成 ALTER 语句

  
SELECT 'ALTER TABLE "AST_CUSMG_SIT"."' || table_name || '" ALTER COLUMN "' || column_name || '" TYPE TIMESTAMP(0);'  
FROM information_schema.columns  
WHERE table_schema = 'AST_CUSMG_SIT'  
  AND udt_name = 'TIMESTAMP'  
ORDER BY table_name, column_name;

执行此查询后，复制输出的所有 ALTER 语句并批量执行即可。

### 5.3 修改效果

将 `pg_oracle."TIMESTAMP"` 改为标准 `TIMESTAMP(0)` 后：

- 数据存储不再保留小数秒
    
- 查询输出格式：`2026-06-24 14:43:51`（无 `.955000`）
    
- 已有数据中的小数秒部分会被截断（四舍五入到秒）
    

### 5.4 建表时预防

新建表时直接使用标准 `TIMESTAMP(0)` 代替 `pg_oracle."TIMESTAMP"`：

  
"CREATE_TIME" TIMESTAMP(0) NOT NULL,  
"UPDATE_TIME" TIMESTAMP(0) NULL,

## 6. 关键要点总结

|项目|说明|
|---|---|
|问题根源|pg_oracle."TIMESTAMP" 默认6位小数秒，且元数据不报告真实精度|
|识别方法|通过 `udt_name = 'TIMESTAMP'` 查找，而非 `datetime_precision > 0`|
|修复方式|`ALTER COLUMN TYPE TIMESTAMP(0)` 将自定义类型转为标准0精度类型|
|预防措施|新表直接使用 `TIMESTAMP(0)` 替代 `pg_oracle."TIMESTAMP"`|
|注意事项|修改类型后不再属于 pg_oracle 兼容体系，Oracle 函数/运算可能需评估|
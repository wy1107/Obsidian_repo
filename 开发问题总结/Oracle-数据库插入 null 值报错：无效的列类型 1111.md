## 1. 问题描述

在使用 MyBatis-Plus + Oracle 数据库执行 INSERT 时，当实体类中某些字段值为 `null`，抛出如下异常：

org.apache.ibatis.type.TypeException: Error setting null for parameter #9 with JdbcType OTHER.  
Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property.  
Cause: java.sql.SQLException: 无效的列类型: 1111

## 2. 根因分析

### 2.1 触发条件（三者缺一不可）

| 条件                                       | 说明                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------ |
| Oracle 数据库                               | Oracle 驱动在 `setNull()` 时不接受 `JdbcType.OTHER`（类型代码 1111），必须明确指定具体类型 |
| `insertStrategy = FieldStrategy.IGNORED` | MyBatis-Plus 策略为"忽略"，即字段值为 null 时也参与 INSERT 语句                     |
| 字段实际值为 null                              | 业务上该字段为空（如 `@JsonIgnore` 导致反序列化后为 null）                            |

### 2.2 调用链路

  
1. 实体字段值为 null  
2. insertStrategy=IGNORED → null 字段仍出现在 INSERT SQL 中  
3. MyBatis 未指定 jdbcType → 默认使用 JdbcType.OTHER (1111)  
4. Oracle 驱动 PreparedStatement.setNull(paramIndex, Types.OTHER)  
5. Oracle 不识别 OTHER → 抛出 SQLException: 无效的列类型: 1111

### 2.3 本项目实际场景

`CustMain` 实体中有三个 `@JsonIgnore` + `insertStrategy = IGNORED` 的脱敏字段：

// 证件号码前缀（脱敏用）  
@TableField(value = "CERT_NO_PREFIX", insertStrategy = FieldStrategy.IGNORED, updateStrategy = FieldStrategy.IGNORED)  
@JsonIgnore  
private String certNoPrefix;  
​  
// 证件号码后缀（脱敏用）  
@TableField(value = "CERT_NO_SUFFIX", insertStrategy = FieldStrategy.IGNORED, updateStrategy = FieldStrategy.IGNORED)  
@JsonIgnore  
private String certNoSuffix;  
​  
// 客户名称前缀（脱敏用）  
@TableField(value = "CUST_NM_PREFIX", insertStrategy = FieldStrategy.IGNORED, updateStrategy = FieldStrategy.IGNORED)  
@JsonIgnore  
private String custNmPrefix;

- `@JsonIgnore` 使得 JSON 反序列化时这些字段始终为 `null`
    
- `insertStrategy = IGNORED` 使得 `null` 值也会出现在 INSERT 语句中
    
- 两者叠加，必然触发 Oracle 的 1111 错误
    

## 3. 解决方案

### 方案一：指定 jdbcType（推荐，已采用）

在 `@TableField` 注解中显式指定 `jdbcType = JdbcType.VARCHAR`：

  
@TableField(value = "CERT_NO_PREFIX", insertStrategy = FieldStrategy.IGNORED,   
            updateStrategy = FieldStrategy.IGNORED, jdbcType = JdbcType.VARCHAR)  
@JsonIgnore  
private String certNoPrefix;

**注意**：`JdbcType` 的 import 路径是 `org.apache.ibatis.type.JdbcType`，不是 `com.baomidou.mybatisplus.annotation`：

  
import org.apache.ibatis.type.JdbcType;

### 方案二：全局配置 jdbcTypeForNull

在 `application.yaml` 中配置 MyBatis 对 null 值默认使用 VARCHAR：

  
mybatis-plus:  
  configuration:  
    jdbc-type-for-null: VARCHAR

> 优点：一行配置解决所有类似问题，无需逐个字段修改缺点：全局生效，可能影响其他本应使用 OTHER 类型的场景

### 方案三：将 insertStrategy 改为 NOT_NULL

  
@TableField(value = "CERT_NO_PREFIX", insertStrategy = FieldStrategy.NOT_NULL,   
            updateStrategy = FieldStrategy.NOT_NULL)  
@JsonIgnore  
private String certNoPrefix;

> 当字段为 null 时不参与 INSERT，自然不会触发 Oracle 类型错误缺点：如果数据库有非空约束则不适用，且与脱敏业务可能需要 null 值入库的需求冲突

## 4. 方案对比

|方案|改动范围|影响范围|推荐度|
|---|---|---|---|
|指定 jdbcType|逐字段修改|仅影响指定字段|★★★★★|
|全局 jdbcTypeForNull|一行配置|全局所有 Mapper|★★★★|
|改 insertStrategy|逐字段修改|仅影响指定字段|★★★|

## 5. 经验总结

1. **Oracle + MyBatis 组合下**，任何可能为 null 且参与 INSERT/UPDATE 的字段，都应在 `@TableField` 中指定 `jdbcType`
    
2. `@JsonIgnore` 字段天然为 null（外部传入的 JSON 无法设置该字段），配合 `insertStrategy = IGNORED` 是高危组合
    
3. MySQL 不存在此问题（MySQL 驱动接受 `Types.OTHER`），此类问题通常在切换到 Oracle 时才暴露
    
4. `JdbcType` 属于 MyBatis 核心 API（`org.apache.ibatis.type.JdbcType`），不属于 MyBatis-Plus
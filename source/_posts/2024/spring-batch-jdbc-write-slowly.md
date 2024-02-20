---
title: Spring Batch 写入数据库慢
date: 2024-01-07 16:21:10
tags:
  - 数据库
  - 生产问题
categories:
  - 解决问题
---

我在项目中使用 Spring Batch 将文本文件导入到数据库中，在将数据导入一个多个字段可能为`null`的宽表时，十万量级的数据约需十分钟，遂进行排查。

<!-- more -->

## 问题复现

数据库是 Oracle 19c，驱动为 ojdbc8:12.2.0.1。

### 准备数据

新建如下表

```sql
CREATE TABLE `excel` (
  `id` int,
  `str` varchar(50),
  `num1` int,
  `num2` decimal(26,6),
  PRIMARY KEY (`id`)
);
```

### 执行批量插入

```java
insert into excel(id,str,num1,num2) values (?,?,?,?)
```

## 原因分析

通过生成导入动作执行期间的火焰图，可以定位到执行耗时长的问题在`StatementCreatorUtils.setNull`中，相关方法源码如下

```java
/**
 * 操作 prepared statement 的工具类
 */
public abstract class StatementCreatorUtils {
    // 是否忽略调用 {@link PreparedStatement#getParameterMetaData()} 方法，默认为 false
    public static final String IGNORE_GETPARAMETERTYPE_PROPERTY_NAME = "spring.jdbc.getParameterType.ignore";

    // 类加载时读取系统属性 spring.jdbc.getParameterType.ignore 的值
    static boolean shouldIgnoreGetParameterType = SpringProperties.getFlag(IGNORE_GETPARAMETERTYPE_PROPERTY_NAME);

    /**
     * 设置 PreparedStatement 参数
     *
     * @param ps         prepared statement 或 callable statement
     * @param paramIndex 参数索引
     * @param sqlType    参数的 SQL 类型
     * @param typeName   参数的类型名称
     * @param scale      数字精度 (DECIMAL 和 NUMERIC 类型)
     * @param inValue    要设置的值
     */
    private static void setParameterValueInternal(PreparedStatement ps, int paramIndex, int sqlType,
            @Nullable String typeName, @Nullable Integer scale, @Nullable Object inValue) throws SQLException {
        String typeNameToUse = typeName;
        int sqlTypeToUse = sqlType;
        Object inValueToUse = inValue;

        // 如果值是 SqlParameterValue 类型，从中取出类型和值
        if (inValue instanceof SqlParameterValue) {
            SqlParameterValue parameterValue = (SqlParameterValue) inValue;
            if (parameterValue.getSqlType() != SqlTypeValue.TYPE_UNKNOWN) {
                sqlTypeToUse = parameterValue.getSqlType();
            }
            if (parameterValue.getTypeName() != null) {
                typeNameToUse = parameterValue.getTypeName();
            }
            inValueToUse = parameterValue.getValue();
        }

        // 如果值是 null 调用 setNull
        if (inValueToUse == null) {
            setNull(ps, paramIndex, sqlTypeToUse, typeNameToUse);
        } else {
            setValue(ps, paramIndex, sqlTypeToUse, typeNameToUse, scale, inValueToUse);
        }
    }

    /**
     * 设置 PreparedStatement 参数为 null
     *
     * @param ps         prepared statement
     * @param paramIndex 参数索引
     * @param sqlType    参数的 SQL 类型
     * @param typeName   参数的类型名称
     */
    private static void setNull(PreparedStatement ps, int paramIndex, int sqlType, @Nullable String typeName)
            throws SQLException {
        // 如果参数 SQL 类型未知，通过数据库信息判断类型
        if (sqlType == SqlTypeValue.TYPE_UNKNOWN || (sqlType == Types.OTHER && typeName == null)) {
            boolean useSetObject = false;
            Integer sqlTypeToUse = null;
            // 尝试调用 java.sql.ParameterMetaData#getParameterType 获取参数 SQL 类型
            if (!shouldIgnoreGetParameterType) {
                try {
                    sqlTypeToUse = ps.getParameterMetaData().getParameterType(paramIndex);
                } catch (SQLException ex) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("JDBC getParameterType call failed - using fallback method instead: " + ex);
                    }
                }
            }
            // 通过数据库信息确定要使用的参数 SQL 类型
            if (sqlTypeToUse == null) {
                sqlTypeToUse = Types.NULL;
                DatabaseMetaData dbmd = ps.getConnection().getMetaData();
                String jdbcDriverName = dbmd.getDriverName();
                String databaseProductName = dbmd.getDatabaseProductName();
                if (databaseProductName.startsWith("Informix") ||
                        (jdbcDriverName.startsWith("Microsoft") && jdbcDriverName.contains("SQL Server"))) {
                    useSetObject = true;
                } else if (databaseProductName.startsWith("DB2") ||
                        jdbcDriverName.startsWith("jConnect") ||
                        jdbcDriverName.startsWith("SQLServer") ||
                        jdbcDriverName.startsWith("Apache Derby")) {
                    sqlTypeToUse = Types.VARCHAR;
                }
            }
            if (useSetObject) {
                // SQL Server 调用 PreparedStatement.setObject
                ps.setObject(paramIndex, null);
            } else {
                // 其他数据库调用 PreparedStatement.setNull
                ps.setNull(paramIndex, sqlTypeToUse);
            }
        } else if (typeName != null) {
            ps.setNull(paramIndex, sqlType, typeName);
        } else {
            ps.setNull(paramIndex, sqlType);
        }
    }
}
```

对于 Oracle，当需要设置的参数值为 `null` 时，StatementCreatorUtils 会调用
`OraclePreparedStatement.getParameterMetaData`方法获取参数元数据，
然后调用`OracleParameterMetaData.getParameterType`获取参数类型。

问题就出在这里，对于每一个值为`null`的参数，Spring 都会调用`OraclePreparedStatement.getParameterMetaData`远程查询数据库，这里花费较多时间；然后调用`OracleParameterMetaData.getParameterType`获取参数类型，期间会调用`OracleParameterMetaData.checkValidIndex`检查索引是否合法，此处会抛出`SQLFeatureNotSupportedException`异常。

```java
void checkValidIndex(int var1) throws SQLException {
    if (this.throwUnsupportedFeature) {
        throw (SQLException) ((SQLException) DatabaseError.createSQLFeatureNotSupportedException("checkValidIndex")
                .fillInStackTrace());
    } else if (var1 < 1 || var1 > this.parameterCount) {
        throw (SQLException) ((SQLException) DatabaseError
                .createSqlException(this.getConnectionDuringExceptionHandling(), 3).fillInStackTrace());
    }
}
```

截至目前，最新的关于该问题的 commit
[Bypass getParameterType by default for PostgreSQL and SQL Server drivers](https://github.com/spring-projects/spring-framework/commit/77b0382a6c86f654958368e4e2709f2343100ada)
对 PostgreSQL and SQL Server 进行了特殊处理，在不配置系统属性`spring.jdbc.getParameterType.ignore`时，将 SQL 类型直接设置为`NULL`。

## 解决方案

设置系统属性`spring.jdbc.getParameterType.ignore`为`true`

## 参考

- [Avoid repeated getParameterType calls for setNull with Oracle 12c driver [SPR-14574] · Issue #19143 · spring-projects/spring-framework (github.com)](https://github.com/spring-projects/spring-framework/issues/19143)
- [Oracle 12c JDBC driver throws inconsistent exception from getParameterType (affecting setNull calls) [SPR-13825] · Issue #18398 · spring-projects/spring-framework (github.com)](https://github.com/spring-projects/spring-framework/issues/18398)
- [Improve default `setNull` performance on PostgreSQL and MS SQL Server (e.g. for `NamedParameterJdbcTemplate` batch updates) · Issue #25679 · spring-projects/spring-framework (github.com)](https://github.com/spring-projects/spring-framework/issues/25679)

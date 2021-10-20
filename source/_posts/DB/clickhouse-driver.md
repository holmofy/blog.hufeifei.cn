---
title: clickhouse-jdbc性能排查
date: 2021-04-19
categories: 数据库
tags: 
- DB
- ClickHouse
keywords:
- Clickhouse-jdbc
- 性能问题
- Clickhouse 批量写入
---

我们项目准备用clickhouse来做数据统计，当前(2021-04-18)使用的是clickhouse官方最新发布的[0.3.0](https://github.com/ClickHouse/clickhouse-jdbc/tree/v0.3.0)版jdbc驱动，使用过程中碰到了几个问题：

1、javacc解析器导致大文本sql语句的解析性能损耗严重

2、`ClickHouseConnection#getMetaData`获取元数据时使用反射，导致`JdbcTemplate.batchUpdate`性能损耗严重。

3、对java8的LocalDateTime支持有问题



# JavaCC解析器的性能问题

按照clickhouse[官方文档中给的性能建议](https://clickhouse.tech/docs/en/introduction/performance/#performance-when-inserting-data)：使用批量插入单个请求里至少1000行。官方给的性能数据是CSV插入MergeTree表，可以达到50～200M/S。

根据这个建议，项目中将kafka的数据按批次读取出来，直接拼成一条条sql去执行，但是线上性能和预期差距特别大：一秒不到一万条数据，一万条trade_info大概5M，也就是不超过5M/s。

## clickhouse-server性能

首先想的是测试一下clickhouse-server性能，是不是言过其实。

我在开发环境测试过，由于开发环境带宽影响也没达到预期：开发环境是公司内网，机器部署在15楼，7楼访问带宽只能达到4M/s，网络影响很大。

所以干脆直接起了个docker，写了个脚本在docker里直接测了下性能：

| rows            | curl                   | clickhouse-client sql  | clickhouse-client csv   |
| --------------- | ---------------------- | ---------------------- | ----------------------- |
| 10000<br />5.2M | 295.195ms<br />17.6M/s | 178.915ms<br />29.0M/s | 131.204ms<br />39.69M/s |

这里使用clickhouse自带的`clickhouse-client`插入csv能达到将近40M/s，所以基本排除了clickhouse-server的问题。

## JavaCC解析器

鉴于此，将我们的测试代码跑了一下[profiler](https://github.com/jvm-profiling-tools/async-profiler)，发现大部分cpu时间都花在了`ClickHouseParser.parse`方法上：

![arthas](https://user-images.githubusercontent.com/19494806/114171847-6c255800-9967-11eb-9537-80e1ad11c101.png)

看clickhouse-jdbc源码发现[ClickHouseSqlParser](https://github.com/ClickHouse/clickhouse-jdbc/blob/v0.3.0/clickhouse-jdbc/src/main/javacc/ClickHouseSqlParser.jj)是使用JavaCC文法分析器自动生成的代码。从github上的[issue](https://github.com/ClickHouse/clickhouse-jdbc/pull/563)来看JavaCC解析器是0.2.6版本新引入的，用于替换之前基于正则表达式的解析器。

我提了个[issue](https://github.com/ClickHouse/clickhouse-jdbc/issues/615)给clickhouse-jdbc项目，得到回复说可以用`use_new_parser=false`禁用JavaCC解析器。

我用了一下发现Connection都无法创建，原因是Connection初始化的时候会执行一条查询clickhouse-server时区的sql。而且由于原来基于正则的解析器bug特别多，它将在0.3.0被移除，为了保证向后兼容没有使用`use_new_parser=false`的方式。

# JdbcTemplate.batchUpdate的问题

clickhouse-driver的开发人员在issue中提到，可以直接用PreparedStatement.addBatch来批量插入数据。

所以我们又转向用`JdbcTemplate.batchUpdate`的方式来执行批量插入，发现性能离预期还是有很大差距。

## SqlExecutor vs. BatchUpdater vs. HttpClientExecutor

最后我写了份测试代码，直接用HttpClient调用clickhouse-server的Http接口。

得到如下的测试结果(纵坐标是执行耗时，单位ms)：

* SqlExecutor是jdbcTemplate.execute直接执行大文本sql
* BatchUpdater是jdbcTemplate.batchUpdate批量插入
* HttpClientExecutor是使用httpclient直接调用clickhouse-server的Http接口执行大文本sql。

![](https://user-images.githubusercontent.com/19494806/114687094-fabe1e80-9d45-11eb-9b34-73f3a75aded6.png)

用httpclient差不多1秒钟可以插入3W条数据，大约15M/s，和直接在docker里用curl插入性能相差不多。

看了一下jdbcTemplate.batchUpdate的消耗的火炬图，发现大部分CPU时间都花在`StatementCreateorUtils.setNull`：

![](https://user-images.githubusercontent.com/19494806/114815429-ba63ac80-9de8-11eb-8e1a-b75717e7908a.png)

从Spring源码看来，是因为**对于字段类型未知的null字段**，Spring会调用`Connection.getMetaData`去获取数据库的类型，从而在setNull的时候兼容不同的数据库。

```java
package org.springframework.jdbc.core;

public abstract class StatementCreatorUtils {
  // ...
		if (sqlType == SqlTypeValue.TYPE_UNKNOWN || (sqlType == Types.OTHER && typeName == null)) {
			boolean useSetObject = false;
			Integer sqlTypeToUse = null;
			if (!shouldIgnoreGetParameterType) {
				try {
					sqlTypeToUse = ps.getParameterMetaData().getParameterType(paramIndex);
				}
				catch (SQLException ex) {
					if (logger.isDebugEnabled()) {
						logger.debug("JDBC getParameterType call failed - using fallback method instead: " + ex);
					}
				}
			}
			if (sqlTypeToUse == null) {
				// Proceed with database-specific checks
				sqlTypeToUse = Types.NULL;
				DatabaseMetaData dbmd = ps.getConnection().getMetaData();
				String jdbcDriverName = dbmd.getDriverName();
				String databaseProductName = dbmd.getDatabaseProductName();
				if (databaseProductName.startsWith("Informix") ||
						(jdbcDriverName.startsWith("Microsoft") && jdbcDriverName.contains("SQL Server"))) {
						// "Microsoft SQL Server JDBC Driver 3.0" versus "Microsoft JDBC Driver 4.0 for SQL Server"
					useSetObject = true;
				}
				else if (databaseProductName.startsWith("DB2") ||
						jdbcDriverName.startsWith("jConnect") ||
						jdbcDriverName.startsWith("SQLServer")||
						jdbcDriverName.startsWith("Apache Derby")) {
					sqlTypeToUse = Types.VARCHAR;
				}
			}
			if (useSetObject) {
				ps.setObject(paramIndex, null);
			}
			else {
				ps.setNull(paramIndex, sqlTypeToUse);
			}
		}
		else if (typeName != null) {
			ps.setNull(paramIndex, sqlType, typeName);
		}
		else {
			ps.setNull(paramIndex, sqlType);
		}
  //...
}
```

而ClickHouseConnection在`getMetaData`时会通过反射来记录trace日志。

```java
package ru.yandex.clickhouse;

public class ClickHouseConnectionImpl implements ClickHouseConnection {
  //...
		@Override
    public DatabaseMetaData getMetaData() throws SQLException {
        return LogProxy.wrap(DatabaseMetaData.class, new ClickHouseDatabaseMetadata(url, this));
    }
	//...
}
```

## 原生Jdbc批量插入性能

最后我在测试中加入了原生Jdbc的Batch方式：

```java
    public int batchInsert(String batchSql, List<Object[]> args) {
        try (Connection conn = dataSource.getConnection();
             PreparedStatement ps = conn.prepareStatement(batchSql)) {
            Preconditions.checkArgument(JdbcUtils.supportsBatchUpdates(conn));
            for (Object[] arg : args) {
                for (int i = 0; i < arg.length; i++) {
                    int paramIndex = i + 1;
                    Object o = arg[i];
                    if (o instanceof CharSequence) {
                        ps.setString(paramIndex, o.toString());
                    } else {
                        // ClickHouse-Jdbc考虑了o为null的情况，不用做特殊处理
                        ps.setObject(paramIndex, o);
                    }
                }
                ps.addBatch();
            }
            int[] ints = ps.executeBatch();
            return Arrays.stream(ints).sum();
        } catch (SQLException e) {
            return ExceptionUtils.rethrow(e);
        }
    }
```

测试结果发现，原生Jdbc批量插入和直接用HttpClient基本没什么性能差距。

![](https://user-images.githubusercontent.com/19494806/114811097-0c540480-9de0-11eb-9d8f-5ceb9f9899c6.png)

上线后Kafka消费速度从原来的不到5M/s(均速3M/s)提升到了15M/s

![](https://p.pstatp.com/origin/pgc-image/999908a2de3649528da7316decd5a9a4)

<!--

[clickhouse](https://clickhouse.tech/blog/en/2019/how-to-speed-up-lz4-decompression-in-clickhouse/)使用牺牲部分[压缩比提高速度LZ4压缩](https://github.com/lz4/lz4#benchmarks)算法，ClickHouse JDBC-Driver对http一样也使用了[LZ4算法](https://github.com/ClickHouse/clickhouse-jdbc/blob/v0.3.0/clickhouse-jdbc/src/main/java/ru/yandex/clickhouse/LZ4EntityWrapper.java)。

使用spring framework的批量插入数据时，null字段获取元数据

clickhouse-jdbc使用[TabSeparated](https://clickhouse.tech/docs/en/interfaces/formats/#tabseparated)格式进行批量插入，没有进行任何压缩。

-->

# LocalDateTime问题

我们项目直接把Kafka中同步的json拼成sql写入到clickhouse，没有先转换成Java对象再交给ORM框架处理。

我们kafka中存储的时间类型都是`yyyy-MM-dd HH:mm:ss`标准格式的字符串，好在clickhouse-jdbc也是使用ClickHouseValueFormatter直接将所有类型转换成字符串，并且时间的格式也是`yyyy-MM-dd HH:mm:ss`。

但如果是直接使用JdbcTemplate，插入LocalDateTime类型可能就会有问题，因为[clickhouse-jdbc 2.6版本不支持LocalDateTime类型](https://github.com/ClickHouse/clickhouse-jdbc/issues/583)，`PreparedStatement.setObject()`会直接调用`LocalDateTime.toString()`方法，toString方法格式是`uuuu-MM-dd'T'HH:mm:ss`，服务端执行时将无法识别。

[clickhouse-jdbc3.0](https://github.com/ClickHouse/clickhouse-jdbc/pull/418)添加了对java.time的支持: [ClickHousePreparedStatementImpl](https://github.com/ClickHouse/clickhouse-jdbc/blob/v0.3.0/clickhouse-jdbc/src/main/java/ru/yandex/clickhouse/ClickHousePreparedStatementImpl.java#L307)、[ClickHouseResultSet](https://github.com/ClickHouse/clickhouse-jdbc/blob/v0.3.0/clickhouse-jdbc/src/main/java/ru/yandex/clickhouse/response/ClickHouseResultSet.java#L742)。

但是在返回值解析时，被统一转成了UTC时区的LocalDateTime: 

```java
package ru.yandex.clickhouse.response.parser;

abstract class ClickHouseDateValueParser<T> extends ClickHouseValueParser<T> {
// ...
    protected LocalDateTime dateTimeToLocalDateTime(String value, ClickHouseColumnInfo columnInfo, TimeZone timeZone) {
        TimeZone serverTimeZone = columnInfo.getTimeZone();
        LocalDateTime localDateTime = parseAsLocalDateTime(value);
        if (serverTimeZone != null
            && (serverTimeZone.useDaylightTime() || serverTimeZone.getRawOffset() > 0)) { // non-UTC
            localDateTime = localDateTime.atZone(columnInfo.getTimeZone().toZoneId())
                .withZoneSameInstant(java.time.ZoneId.of("UTC")).toLocalDateTime();
        }

        return localDateTime;
    }
//...
}
```

如果是使用MyBatis等ORM框架，MyBatis会提供LocalDateTimeTypeHandler将LocalDateTime以TimeStamp形式进行处理，clickhouse-jdbc0.2.6和0.3.0版本都会处理Timestamp类型，就没有这些问题了。



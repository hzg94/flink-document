# Flink SQL API

# Connector 

## FileSystem  

### 参数

##### * `path` = 'file:///' + 路径

##### as:

```scala
envTable.executeSql(      
"""
        |create table add (
        |`UserId` bigint
        |) WITH (
        |'connector' = 'filesystem',
        |'path' = 'file:///D:\project\java\reflink\reflink\source\UserBehavior.csv',
        |'format' = 'csv'
        |)
        |""".stripMargin)
```

## JDBC

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-jdbc_2.11</artifactId>
  <version>1.13.6</version>
</dependency>
```

##### * `url`JDBC 数据库 url。

##### `table-name` 连接到 JDBC 表的名称。

##### `driver` 用于连接到此 URL 的 JDBC 驱动类名，如果不设置，将自动从 URL 中推导。

##### `username` JDBC 用户名

##### `password` JDBC 密码。

##### 注意没有foramt		

##### as:

```scala
envTable.executeSql(      
"""
        |create table add (
        |`UserId` bigint
        |) WITH (
        |'connector' = 'jdbc',
        |'url' = 'jdbc:mysql://localhost:3306/test'
        | 
        |)
        |""".stripMargin)
```

## Kafka

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka_2.11</artifactId>
  <version>${flink.version}</version>
</dependency>
```

### 参数

##### * `topic Kafka` 记录的 Topic 名。

##### `partition Kafka` 记录的 partition ID。

##### `headers 二进制 Map` 类型的 Kafka 记录头（Header）

##### `leader-epoch` Kafka记录的 Leader epoch（如果可用）

##### `offset` Kafka 记录在 partition 中的 offset。

##### `timestamp` Kafka 记录的时间戳。

##### `timestamp-type` Kafka 记录的时间戳类型。可能的类型有 "NoTimestampType"， "CreateTime"（会在写入元数据时设置），或 "LogAppendTime"。

##### * `properties.bootstrap.servers` 逗号分隔的 Kafka broker 列表。

##### * `properties.group.id` kafak 组id

##### `properties.*` 可以设置和传递任意 Kafka 的配置项,后缀名必须匹配在 [Kafka 配置文档](https://kafka.apache.org/documentation/#configuration) 中定义的配置键

### 一致性保证 EOS

开启checkpoint

 参数 `sink.semantic` 

`none` 不保证任何语义

`at-least-once` (默认设置)  至少一次

`exactly-once ` 精确一次

##### as:

```scala
envTable.executeSql(      
"""
        |create table add (
        |`UserId` bigint
        |) WITH (
        |'connector' = 'kafka',
        |'topic' = 'demo' ,
        |'format' = 'csv'
        |)
        |""".stripMargin)
```

# format 

## Csv 

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-csv</artifactId>
  <version>${flink.version}</version>
</dependency>
```

### 参数

##### csv.field-delimiter  字段分隔符 (默认`','`)

##### csv.disable-quote-character 是否禁止对引用的值使用引号 (默认是 false) 

##### csv.quote-character 用于围住字段值的引号字符 (默认`"`) 

##### csv.allow-comments 是否允许忽略注释行（默认不允许）

##### csv.ignore-parse-errors  当解析异常时，是跳过当前字段或行，还是抛出错误失败（默认为 false，即抛出错误失败）。如果忽略字段的解析异常，则会将该字段值设置为`null`。

##### csv.array-element-delimiter 分隔数组和行元素的字符串(默认`';'`).

##### csv.escape-character 转义字符(默认关闭).

##### csv.null-literal 是否将 "null" 字符串转化为 null 值

## Json

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-json</artifactId>
  <version>${flink.version}</version>
</dependency>
```

## Avro

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-avro</artifactId>
  <version>${flink.version}</version>
</dependency>
```

# Function

## FROM_UNIXTIME

FROM_UNIXTIME(BIGINT)

传入一个十位数 转化 时间字符串

## TO_TIMESTAMP

1. FROM_UNIXTIME(BIGINT) 传入一个十三位数 转化 时间类型
2. FROM_TIMESTAMP(DATE)  传入时间字符串 转化 时间类型

# WATERMARK 

严格递增时间戳：

```scala
WATERMARK FOR rowtime_column AS rowtime_column
```

发出到目前为止已观察到的最大时间戳的 watermark ，时间戳大于最大时间戳的行被认为没有迟到。

递增时间戳:

延时 5 秒 生成 watermark 等同于 watermark 允许 5 秒 迟到

```sql
WATERMARK FOR time_ltz AS time_ltz - INTERVAL '5' SECOND
```

发出到目前为止已观察到的最大时间戳减 1 的 watermark ，时间戳大于或等于最大时间戳的行被认为没有迟到。

# Select

## !!! WINDOW 窗口

### TUMBLE 滚动窗口

```sql
TUMBLE(TABLE data,DESCRIPTOR(timecol), size)
```

data 表名 e.g. `TABLE demo`

timecol timestamp 字段 e.g. `DESCRIPTOR(bidtime)`

size 滚动窗口大小 e.g. `INTERVAL '10' MINUTES`

e.g.

```sql
select * from TABLE(
    TUMBLE(
        TABLE demo,
        DESCRIPTOR(bidtime),
        INTERVAL '10' MINUTES
    )
)
```

or

```sql
select * from TABLE(
    TUMBLE(
        DATA => TABLE demo,
        TIMECOL => DESCRIPTOR(bidtime),
        SIZE => INTERVAL '10' MINUTES
    )
)
```

实际使用:

```scala
// 实际使用中加上window_start, window_end 
envTable.sqlQuery(
   """
    |select `window_start`,`window_end`,`UserId` from TABLE(
    |   TUMBLE (
    |       TABLE add,
    |       DESCRIPTOR(`TimeStamp`),
    |       INTERVAL '10' MINUTES      
    |   )
    |)
    |where action = 'pv'
    |GROUP BY `window_start`, `window_end`,`UserId`
    |""".stripMargin).limit(1)
```



### HOP 滑动窗口

```sql
HOP(TABLE data, DESCRIPTOR(timecol), slide, size [, offset ])
```

data 表名 e.g. `TABLE demo

timecol timestamp 字段 e.g. `DESCRIPTOR(bidtime)`

slide 滑动大小 e.g. `INTERVAL '5' MINUTES`

size  滑动窗口大小 e.g. `INTERVAL '10' MINUTES`

e.g.

```sql
select * from TABLE(
    HOP(
        TABLE demo,
        DESCRIPTOR(bidtime),
        INTERVAL '5' MINUTES,
        INTERVAL '10' MINUTES
       )
)
```

or

```sql
select * from TABLE(
    HOP(
        DATA => TABLE demo,
        TIMECOL => DESCRIPTOR(bidtime),
        SLIDE => INTERVAL '5' MINUTES,
        SIZE => INTERVAL '10' MINUTES
       )
)
```

# Flink Table API

## Operations

### Scan, Projection, and Filter 

### from

```scala
tableEnv.from('Orders') 
等同于 select * from Orders
```

本身等同于sql中 `from`

### FromValues

```scala
val table = tableEnv.fromValues(
	row(1,"ABC"),
    row(2,"ABCDE")
)
```

效果如下：

```sql
root
| -- f0 BIGINT NOT NULL
| -- f1 VARCHAR(5) NOT NULL
```

默认自动识别类型 可以指定类型 如下:

```scala
val table = tableEnv.fromValues(
	DataTypes.ROW(
    	DataTypes.FIELD("id",DataTypes.DECIMAL(10,2)),
        DataTypes.FIELD("name",DataTypes.STRING())
    ),
    row(1,"ABC"),
    row(2,"ABCDE")
)
```

结构如下:

```sql
root
| -- id DECIMAL(10,2)
| -- name STRING
```

本身等同于sql中 `values`

### Select

```scala
val orders = tableEnv.from("Orders")
Table result = orders.select($"a", $"c" as "d")
// or
Table result = orders.select($"*")
```

本身等同于sql中 `select`

### As

```scala
val orders = tableEnv.from("Orders");
val result = orders.as("x, y, z, t");
```

### Where / Filter

```scala
val orders = tableEnv.from("Orders");
val result = orders.where($("b").isEqual("red")); 
// select * from Orders where b = 'red'
// or
val orders = tableEnv.from("Orders");
val result = orders.filter($("b").isEqual("red"));
// select * from Orders where b != 'red'
```

## 列操作

### ~~AddColumns~~

执行字段添加操作。 如果所添加的字段已经存在，将抛出异常。

```scala
val orders = tableEnv.from("Orders");
val result = orders.addColumns(concat($"c"));

```

### ~~AddOrReplaceColumns~~

执行字段添加操作。 如果添加的列名称和已存在的列名称相同，则已存在的字段将被替换。 此外，如果添加的字段里面有重复的字段名，则会使用最后一个字段。

```scala
val orders = tableEnv.from("Orders")
val result = orders.addOrReplaceColumns(concat($"c", "Sunny") as "desc")

```

### DropColumns

```scala
val orders = tableEnv.from("Orders")
val result = orders.dropColumns($"b")
```

### ~~RenameColumns~~

```scala
val orders = tableEnv.from("Orders")
val result = orders.renameColumns($"b" as "b2")
```

## Aggregations

### GroupBy Aggregation

```scala
val orders: Table = tableEnv.from("Orders")
val result = orders.groupBy($"a").select($"a", $"b".sum().as("d"))
```

### GroupBy Window Aggregation 

```scala
val orders: Table = tableEnv.from("Orders")
val result: Table = orders
    .window(Tumble over 5.minutes on $"rowtime" as "w") // 定义窗口
    .groupBy($"a", $"w") // 按窗口和键分组
    .select($"a", $"w".start, $"w".end, $"w".rowtime, $"b".sum as "d") // 访问窗口属性并聚合
```

### Distinct Aggregation

```scala
val orders: Table = tableEnv.from("Orders")
// 按属性分组后的的互异（互不相同、去重）聚合
val groupByDistinctResult = orders
    .groupBy($"a")
    .select($"a", $"b".sum.distinct as "d")
// 按属性、时间窗口分组后的互异（互不相同、去重）聚合
val groupByWindowDistinctResult = orders
    .window(Tumble over 5.minutes on $"rowtime" as "w").groupBy($"a", $"w")
    .select($"a", $"b".sum.distinct as "d")
// over window 上的互异（互不相同、去重）聚合
val result = orders
    .window(Over
        partitionBy $"a"
        orderBy $"rowtime"
        preceding UNBOUNDED_RANGE
        as $"w")
    .select($"a", $"b".avg.distinct over $"w", $"b".max over $"w", $"b".min over $"w")
```

# Flink Type

## char

```sql
char
char(n)
```

n 字符串长度

## varchar

```sql
VARCHAR
VARCHAR(n)
STRING
```

n 字符串长度

## BINARY

```sql
BINARY
BINARY(n)
```

n 二进制字符串

## VARBINARY/BYTES

```sql
VARBINARY
VARBINARY(n)

BYTES
```

n 二进制字符串

## DECIMAL

```sql
DECIMAL
DECIMAL(p)
DECIMAL(p, s)

DEC
DEC(p)
DEC(p, s)

NUMERIC
NUMERIC(p)
NUMERIC(p, s)
```


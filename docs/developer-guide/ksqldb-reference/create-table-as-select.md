---
layout: page
title: CREATE TABLE AS SELECT
tagline:  ksqlDB CREATE TABLE AS SELECT statement
description: Syntax for the CREATE TABLE AS SELECT statement in ksqlDB
keywords: ksqlDB, create, table, push query
---

## Synopsis

```sql
CREATE [OR REPLACE] TABLE table_name
  [WITH ( property_name = expression [, ...] )]
  AS SELECT  select_expr [, ...]
  FROM from_item
  [[ LEFT | FULL | INNER ] JOIN [join_table | join_stream] ON join_criteria]* 
  [ WINDOW window_expression ]
  [ WHERE condition ]
  [ GROUP BY grouping_expression ]
  [ HAVING having_expression ]
  [ EMIT CHANGES ];
```

## Description

Create a new ksqlDB materialized table view, along with the corresponding Kafka topic, and
stream the result of the query as a changelog into the topic.

The WINDOW clause can only be used if the `from_item` is a stream and the query contains
a `GROUP BY` clause.

### Joins

Joins to streams can use any stream column. If the join criteria is not the key column of the stream
ksqlDB will internally repartition the data. 

!!! important
    {{ site.ak }} guarantees the relative order of any two messages from
    one source partition only if they are also both in the same partition
    *after* the repartition. Otherwise, {{ site.ak }} is likely to interleave
    messages. The use case will determine if these ordering guarantees are
    acceptable.

Joins to tables must use the table's PRIMARY KEY as the join criteria: non-key joins are 
not supported. For more information, see [Join Event Streams with ksqlDB](../joins/join-streams-and-tables.md).

See [Partition Data to Enable Joins](../joins/partition-data.md) for more information about how to
correctly partition your data for joins.

!!! note

    - Partitioning streams and tables is especially important for stateful or otherwise
      intensive queries. For more information, see
      [Parallelization](/operate-and-deploy/performance-guidelines/#parallelization).
    - Once a table is created, you can't change the number of partitions.
      To change the partition count, you must drop the table and create it again.

The primary key of the resulting table is determined by the following rules, in order of priority:

1. If the query has a  `GROUP BY`, then the resulting number of primary key
   columns will match the number of grouping expressions. For each grouping
   expression: 

    1. If the grouping expression is a single source-column reference, the
       corresponding primary key column matches the name, type, and contents
       of the source column.

    2. If the grouping expression is a reference to a field within a
       `STRUCT`-type column, the corresponding primary key column matches the
       name, type, and contents of the `STRUCT` field.

    3. If the `GROUP BY` is any other expression, the primary key has a
       system-generated name, unless you provide an alias in the projection,
       and matches the type and contents of the result of the expression.

1. If the query has a join. For more information, see
   [Join Synthetic Key Columns](/developer-guide/joins/synthetic-keys).
1. Otherwise, the primary key matches the name, unless you provide an alias
   in the projection, and type of the source table's primary key.
 
The projection must include all columns required in the result, including any primary key columns.

### Serialization

For supported [serialization formats](/reference/serialization),
ksqlDB can integrate with the [Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/index.html).
ksqlDB registers the value schema of the new table with {{ site.sr }} automatically. 
The schema is registered under the subject `<topic-name>-value`.
ksqlDB can also use [Schema Inference With ID](/operate-and-deploy/schema-inference-with-id) to enable 
using physical schema for data serialization.

### Windowed aggregation

Specify the WINDOW clause to create a windowed aggregation. For more information,
see [Time and Windows in ksqlDB](../../concepts/time-and-windows-in-ksqldb-queries.md).

## Table properties

Specify details about your table by using the WITH clause, which supports the
following properties:

|     Property      |                                             Description                                              |
| ----------------- | ---------------------------------------------------------------------------------------------------- |
| KAFKA_TOPIC       | The name of the Kafka topic that backs this table. If this property is not set, then the name of the table will be used as default. |
| KEY_FORMAT        | Specifies the serialization format of the message key in the topic. For supported formats, see [Serialization Formats](/reference/serialization). If this property is not set, the format from the left-most input stream/table is used. |
| KEY_SCHEMA_ID     | Specifies the schema ID of key schema in {{ site.sr }}. The schema will be used for data serialization. See [Schema Inference With Schema ID](/operate-and-deploy/schema-inference-with-id). |
| VALUE_FORMAT      | Specifies the serialization format of the message value in the topic. For supported formats, see [Serialization Formats](/reference/serialization). If this property is not set, the format from the left-most input stream/table is used. |
| VALUE_SCHEMA_ID   | Specifies the schema ID of value schema in {{ site.sr }}. The schema will be used for data serialization. See [Schema Inference With Schema ID](/operate-and-deploy/schema-inference-with-id). |
| FORMAT            | Specifies the serialization format of both the message key and value in the topic. It is not valid to supply this property alongside either `KEY_FORMAT` or `VALUE_FORMAT`. For supported formats, see [Serialization Formats](/reference/serialization). |
| VALUE_DELIMITER   | Used when VALUE_FORMAT='DELIMITED'. Supports single character to be a delimiter, defaults to ','. For space and tab delimited values you must use the special values 'SPACE' or 'TAB', not an actual space or tab character. |
| PARTITIONS        | The number of partitions in the backing topic. If this property is not set, then the number of partitions of the input stream/table will be used. In join queries, the property values are taken from the left-most stream or table. You can't change the number of partitions on a table. To change the partition count, you must drop the table and create it again. |
| REPLICAS          | The replication factor for the topic. If this property is not set, then the number of replicas of the input stream or table will be used. In join queries, the property values are taken from the left-most stream or table. |
| TIMESTAMP         | Sets a column within this stream's schema to be used as the default source of `ROWTIME` for any downstream queries. Downstream queries that use time-based operations, such as windowing, will process records in this stream based on the timestamp in this column. The column will be used to set the timestamp on any records emitted to Kafka. Timestamps have a millisecond accuracy. If not supplied, the `ROWTIME` of the source stream is used. <br>**Note**: This doesn't affect the processing of the query that populates this stream. For example, given the following statement:<br><pre>CREATE STREAM foo WITH (TIMESTAMP='t2') AS<br>&#0009;SELECT * FROM bar<br>&#0009;WINDOW TUMBLING (size 10 seconds);<br>&#0009;EMIT CHANGES;</pre>The window into which each row of `bar` is placed is determined by bar's `ROWTIME`, not `t2`. |
| TIMESTAMP_FORMAT  | Used in conjunction with TIMESTAMP. If not set, ksqlDB timestamp column must be of type `bigint` or `timestamp`. If it is set, then the TIMESTAMP column must be of type varchar and have a format that can be parsed with the Java `DateTimeFormatter`. If your timestamp format has characters requiring single quotes, you can escape them with two successive single quotes, `''`, for example: `'yyyy-MM-dd''T''HH:mm:ssX'`. For more information on timestamp formats, see [DateTimeFormatter](https://cnfl.io/java-dtf). |
| WRAP_SINGLE_VALUE | Controls how values are serialized where the values schema contains only a single column. The setting controls how the query will serialize values with a single-column schema.<br>If set to `true`, ksqlDB will serialize the column as a named column within a record.<br>If set to `false`, ksqlDB will serialize the column as an anonymous value.<br>If not supplied, the system default, defined by [ksql.persistence.wrap.single.values](/reference/server-configuration#ksqlpersistencewrapsinglevalues), then the format's default is used.<br>**Note:** `null` values have special meaning in ksqlDB. Care should be taken when dealing with single-column schemas where the value can be `null`. For more information, see [Single column (un)wrapping](/reference/serialization#single-field-unwrapping).<br>**Note:** Supplying this property for formats that do not support wrapping, for example `DELIMITED`, or when the value schema has multiple columns, will result in an error. |


!!! note
	  - To use Avro or Protobuf, you must have {{ site.sr }} enabled and
    `ksql.schema.registry.url` must be set in the ksqlDB server configuration
    file. See [Configure ksqlDB for Avro, Protobuf, and JSON schemas](../../operate-and-deploy/installation/server-config/avro-schema.md#configure-avro-and-schema-registry-for-ksql).
    - Avro and Protobuf field names are not case sensitive in ksqlDB. This matches the ksqlDB
    column name behavior.

Examples
--------

```sql
-- Derive a new view from an existing table:
CREATE TABLE derived AS
  SELECT
    a,
    b,
    d
  FROM source
  WHERE A is not null
  EMIT CHANGES;
```

```sql
-- Derive a new view from an existing table with value serialization 
-- schema defined by VALUE_SCHEMA_ID:
CREATE TABLE derived WITH (
    VALUE_SCHEMA_ID=1
  ) AS
  SELECT
    a,
    b,
    d
  FROM source
  WHERE A is not null
  EMIT CHANGES;
```

```sql
-- Join a stream of play events to a songs table, windowing weekly, to create a weekly chart:
CREATE TABLE weeklyMusicCharts AS
   SELECT
      s.songName,
      count(1) AS playCount
   FROM playStream p
      JOIN songs s ON p.song_id = s.id
   WINDOW TUMBLING (7 DAYS)
   GROUP BY s.songName
   EMIT CHANGES;
```

```sql
-- Window retention: configure the number of windows in the past that ksqlDB retains.
CREATE TABLE pageviews_per_region AS
  SELECT regionid, COUNT(*) FROM pageviews
  WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS, RETENTION 7 DAYS, GRACE PERIOD 30 MINUTES)
  WHERE UCASE(gender)='FEMALE' AND LCASE (regionid) LIKE '%_6'
  GROUP BY regionid
  EMIT CHANGES;
```

```sql
-- out-of-order events: accept events for up to two hours after the window ends.
CREATE TABLE top_orders AS
  SELECT orderzip_code, TOPK(order_total, 5) FROM orders
  WINDOW TUMBLING (SIZE 1 HOUR, GRACE PERIOD 2 HOURS) 
  GROUP BY order_zipcode
  EMIT CHANGES;
```

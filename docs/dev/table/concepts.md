---
title: "Concepts"
nav-parent_id: tableapi
nav-pos: 0
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

The Table API introduces a lot of new concepts which users might be unfamiliar with. In this document, we try to explain each of these new concepts. This will help users during development and debugging the Table API code.

* This will be replaced by the TOC
{:toc}

Planners
--------

Planners are responsible for transforming the Table API or SQL code to Flink execution Graph. The execution graph consists of multiple Datastreams or Datasets. Flink provides two planners out of the box - 

- Blink Planner

This is the default planner from Flink 1.10 onwards. Blink Planner was originally written by Alibaba and then contributed back to Flink. Blink Planner was made the default planner due to various performance advantages over the original Planner.

- Flink Planner (Deprecated)

This is the original Planner which is present in Flink 1.9 and below. 

Some of the main difference between the two planners are as follows -


1. Blink treats batch jobs as a special case of streaming. As such, the conversion between Table and DataSet is also not supported, and batch jobs will not be translated into `DateSet` programs but translated into `DataStream` programs, the same as the streaming jobs.
2. The Blink planner does not support `BatchTableSource`, use bounded `StreamTableSource` instead of it.
3. The implementations of `FilterableTableSource` for the old planner and the Blink planner are incompatible. The old planner will push down `PlannerExpression`s into `FilterableTableSource`, while the Blink planner will push down `Expression`s.
4. String based key-value config options (Please see the documentation about [Configuration]({{ site.baseurl }}/dev/table/config.html) for details) are only used for the Blink planner.
5. The implementation(`CalciteConfig`) of `PlannerConfig` in two planners is different.
6. The Blink planner will optimize multiple-sinks into one DAG on both `TableEnvironment` and `StreamTableEnvironment`. The old planner will always optimize each sink into a new DAG, where all DAGs are independent of each other.
7. The old planner does not support catalog statistics now, while the Blink planner does.


Data Types
-----------


Unbounded Data Processing
---------------------------

In Traditional SQL systems, users deal with bounded data all the times. Flink Tables, however, can contain unbounded data backed by a streaming data source such as Apache Kafka. The data keeps on getting populated in real time. The user then has to take care of this fundamental difference while writing queries. Let's go through some scenarios in which unbounded data can cause issues - 

- Getting count of rows in Table. Since the table is getting populated in real time, the number of rows also keep on increasing. Secondly, there is not a defined starting point in the table from which you can start counting rows. 

- Group By operations become non-trivial. The problems are similar to the ones described in the previous point.

- Joining two tables. Join is a trivial operation for bounded data source. But for unbounded data, it is fairly non-trivial. e.g. Let us assume we have to joing two tables which contain key and value. KeyA, ValueA arrives at time t1 in table A. KeyA, ValueB arrives at time t2 in table B. Following scenarios can occur -

a. t1 < t2 - Here KeyA, ValueA will need to be stored in some intermediate state such as heap or rocksDB so that it can be joined by t2
b. t2 < t1 - Here KeyA, ValueB will need to be stored in some intermediate state such as heap or rocksDB so that it can be joined by t1

The issue is we don't know the time difference abs(t2 - t1) i.e. after how long will we receive the value for same key from the joined stream. What if the KeyA never arrives in TableB. Will you keep on holding the KeyA in intermediate state? If you want to set a TTL, what should be correct value?

Unbounded streaming data is represented as Dynamic Tables in Flink. 

Dynamic Tables
----------------

*Dynamic tables* are the core concept of Flink's Table API and SQL support for streaming data. In contrast to the static tables that represent batch data, dynamic tables are changing over time. They can be queried like static batch tables. Querying dynamic tables yields a *Continuous Query*. A continuous query never terminates and produces a dynamic table as result. The query continuously updates its (dynamic) result table to reflect the changes on its (dynamic) input tables.

Dynamic tables can be thought of as similar although not exactly same to Materialized views in most of the databases. In contrast to a virtual view, a materialized view caches the result of the query such that the query does not need to be evaluated when the view is accessed. A common challenge for caching is to prevent a cache from serving outdated results. A materialized view becomes outdated when the base tables of its definition query are modified. *Eager View Maintenance* is a technique to update a materialized view as soon as its base tables are updated.

It is important to note that the result of a continuous query is always semantically equivalent to the result of the same query being executed in batch mode on a snapshot of the input tables.

The following figure visualizes the relationship of streams, dynamic tables, and  continuous queries:

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/stream-query-stream.png" width="80%">
</center>

1. A stream is converted into a dynamic table.
1. A continuous query is evaluated on the dynamic table yielding a new dynamic table.
1. The resulting dynamic table is converted back into a stream.

**Note:** Dynamic tables are foremost a logical concept. Dynamic tables are not necessarily (fully) materialized during query execution.

In the following, we will explain the concepts of dynamic tables and continuous queries with a stream of click events that have the following schema:

{% highlight plain %}
[
  user:  VARCHAR,   // the name of the user
  cTime: TIMESTAMP, // the time when the URL was accessed
  url:   VARCHAR    // the URL that was accessed by the user
]
{% endhighlight %}

### Continuous Queries
----------------------

A continuous query is evaluated on a dynamic table and produces a new dynamic table as result. In contrast to a batch query, a continuous query never terminates and updates its result table according to the updates on its input tables. At any point in time, the result of a continuous query is semantically equivalent to the result of the same query being executed in batch mode on a snapshot of the input tables.

In the following we show two example queries on a `clicks` table that is defined on the stream of click events.

The first query is a simple `GROUP-BY COUNT` aggregation query. It groups the `clicks` table on the `user` field and counts the number of visited URLs. The following figure shows how the query is evaluated over time as the `clicks` table is updated with additional rows.

<center>
<img alt="Continuous Non-Windowed Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-cnt.png" width="90%">
</center>

When the query is started, the `clicks` table (left-hand side) is empty. The query starts to compute the result table, when the first row is inserted into the `clicks` table. After the first row `[Mary, ./home]` was inserted, the result table (right-hand side, top) consists of a single row `[Mary, 1]`. When the second row `[Bob, ./cart]` is inserted into the `clicks` table, the query updates the result table and inserts a new row `[Bob, 1]`. The third row `[Mary, ./prod?id=1]` yields an update of an already computed result row such that `[Mary, 1]` is updated to `[Mary, 2]`. Finally, the query inserts a third row `[Liz, 1]` into the result table, when the fourth row is appended to the `clicks` table.

The second query is similar to the first one but groups the `clicks` table in addition to the `user` attribute also on an [hourly tumbling window]({{ site.baseurl }}/dev/table/sql/queries.html#group-windows) before it counts the number of URLs (time-based computations such as windows are based on special [time attributes](time_attributes.html), which are discussed later.). Again, the figure shows the input and output at different points in time to visualize the changing nature of dynamic tables.

<center>
<img alt="Continuous Group-Window Query" src="{{ site.baseurl }}/fig/table-streaming/query-groupBy-window-cnt.png" width="100%">
</center>

As before, the input table `clicks` is shown on the left. The query continuously computes results every hour and updates the result table. The clicks table contains four rows with timestamps (`cTime`) between `12:00:00` and `12:59:59`. The query computes two results rows from this input (one for each `user`) and appends them to the result table. For the next window between `13:00:00` and `13:59:59`, the `clicks` table contains three rows, which results in another two rows being appended to the result table. The result table is updated, as more rows are appended to `clicks` over time.

### Update and Append Queries

Although the two example queries appear to be quite similar (both compute a grouped count aggregate), they differ in one important aspect:
- The first query updates previously emitted results, i.e., the changelog stream that defines the result table contains `INSERT` and `UPDATE` changes.
- The second query only appends to the result table, i.e., the changelog stream of the result table only consists of `INSERT` changes.

Whether a query produces an append-only table or an updated table has some implications:
- Queries that produce update changes usually have to maintain more state (see the following section).
- The conversion of an append-only table into a stream is different from the conversion of an updated table.

When converting a dynamic table into a stream or writing it to an external system, these changes need to be encoded. Flink's Table API and SQL support three ways to encode the changes of a dynamic table:

* **Append-only stream:** A dynamic table that is only modified by `INSERT` changes can be  converted into a stream by emitting the inserted rows.

* **Retract stream:** A retract stream is a stream with two types of messages, *add messages* and *retract messages*. A dynamic table is converted into an retract stream by encoding an `INSERT` change as add message, a `DELETE` change as retract message, and an `UPDATE` change as a retract message for the updated (previous) row and an add message for the updating (new) row. The following figure visualizes the conversion of a dynamic table into a retract stream.

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/undo-redo-mode.png" width="85%">
</center>
<br><br>

* **Upsert stream:** An upsert stream is a stream with two types of messages, *upsert messages* and *delete messages*. A dynamic table that is converted into an upsert stream requires a (possibly composite) unique key. A dynamic table with unique key is converted into a stream by encoding `INSERT` and `UPDATE` changes as upsert messages and `DELETE` changes as delete messages. The stream consuming operator needs to be aware of the unique key attribute in order to apply messages correctly. The main difference to a retract stream is that `UPDATE` changes are encoded with a single message and hence more efficient. The following figure visualizes the conversion of a dynamic table into an upsert stream.

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/redo-mode.png" width="85%">
</center>
<br><br>

The API to convert a dynamic table into a `DataStream` is discussed on the [Common Concepts]({{ site.baseurl }}/dev/table/common.html#convert-a-table-into-a-datastream) page. Please note that only append and retract streams are supported when converting a dynamic table into a `DataStream`. The `TableSink` interface to emit a dynamic table to an external system are discussed on the [TableSources and TableSinks](../sourceSinks.html#define-a-tablesink) page.

{% top %}

---
title: "Unbounded Data Processing"
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

In Traditional SQL systems, users deal with bounded data all the times. Flink Tables, however, can contain unbounded data backed by a streaming data source such as Apache Kafka. The data keeps on getting populated in real time. The user then has to take care of this fundamental difference while writing queries. Let's go through some scenarios in which unbounded data can cause issues - 

- Getting count of rows in Table. Since the table is getting populated in real time, the number of rows also keep on increasing. Secondly, there is not a defined starting point in the table from which you can start counting rows. 

- Group By operations become non-trivial. The problems are similar to the ones described in the previous point.

- Joining two tables. Join is a trivial operation for bounded data source. But for unbounded data, it is fairly non-trivial. e.g. Let us assume we have to joing two tables which contain key and value. KeyA, ValueA arrives at time t1 in table A. KeyA, ValueB arrives at time t2 in table B. Following scenarios can occur -

a. t1 < t2 - Here KeyA, ValueA will need to be stored in some intermediate state such as heap or rocksDB so that it can be joined by t2
b. t2 < t1 - Here KeyA, ValueB will need to be stored in some intermediate state such as heap or rocksDB so that it can be joined by t1

The issue is we don't know the time difference abs(t2 - t1) i.e. after how long will we receive the value for same key from the joined stream. What if the KeyA never arrives in TableB. Will you keep on holding the KeyA in intermediate state? If you want to set a TTL, what should be correct value?

Unbounded streaming data is represented as *Dynamic Tables* in Flink. 

Dynamic Tables
----------------

*Dynamic tables* are the core concept of Flink's Table API and SQL support for streaming data. In contrast to the static tables that represent batch data, dynamic tables are changing over time although they can be queried just like batch tables. 

Dynamic tables can be thought of as similar although not exactly same to Materialized views in most of the databases. In contrast to a virtual view, a materialized view caches the result of the query such that the query does not need to be evaluated when the view is accessed. A common challenge for caching is to prevent a cache from serving outdated results. A materialized view becomes outdated when the base tables of its definition query are modified. There are multiple techniques to update the view. *Eager View Maintenance* is one such technique which updates a materialized view as soon as its base tables are updated.

Querying dynamic tables yields a *Continuous Query*. A continuous query never terminates and keep on updating its (dynamic) result table to reflect the changes on its (dynamic) input tables. It is important to note that the result of a continuous query is always semantically equivalent to the result of the same query being executed in batch mode on a snapshot of the input tables.

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

In order to process a stream with a relational query, it has to be converted into a `Table`. Conceptually, each record of the stream is interpreted as an `INSERT` modification on the resulting table. Essentially, we are building a table from an `INSERT`-only changelog stream.

The following figure visualizes how the stream of click event (left-hand side) is converted into a table (right-hand side). The resulting table is continuously growing as more records of the click stream are inserted.

<center>
<img alt="Append mode" src="{{ site.baseurl }}/fig/table-streaming/append-mode.png" width="60%">
</center>

**Note:** A table which is defined on a stream is internally not materialized.

### Continuous Queries

A continuous query is evaluated on a dynamic table and produces a new dynamic table as result. In contrast to a batch query, a continuous query never terminates and updates its result table according to the updates on its input tables. At any point in time, the result of a continuous query is semantically equivalent to the result of the same query being executed in batch mode on a snapshot of the input tables.

Let's try to understand it with two example queries on a `clicks` table that is defined on the stream of click events.

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

Let's take a look at how both of these queries differ when the table needs to be converted to a Datastream.

### Update and Append Queries

Although the two example queries appear to be quite similar (both compute a grouped count aggregate), they differ in one important aspect:
- The first query updates previously emitted results, i.e., the changelog stream that defines the result table contains `INSERT` and `UPDATE` changes.
  Such queries need to maintain more state when converted to Datastreams.
    
- The second query only appends to the result table, i.e., the changelog stream of the result table only consists of `INSERT` changes.

When converting a dynamic table into a stream or writing it to an external system, the changes in the data need to be encoded. Flink's Table API and SQL support three ways to encode the changes of a dynamic table:

* **Append-only stream:** A dynamic table that is only modified by `INSERT` changes can be  converted into a stream by emitting the inserted rows.

* **Retract stream:** A retract stream is a stream with two types of messages, *add messages* and *retract messages*. A dynamic table is converted into an retract stream by encoding an `INSERT` change as add message, a `DELETE` change as retract message, and an `UPDATE` change as a retract message for the updated (previous) row and an add message for the updating (new) row. The following figure visualizes the conversion of a dynamic table into a retract stream.

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/undo-redo-mode.png" width="85%">
</center>
<br><br>

* **Upsert stream:** An upsert stream is a stream with two types of messages, *upsert messages* and *delete messages*. A dynamic table that is converted into an upsert stream requires a (possibly composite) unique key. The table is converted into a stream by encoding `INSERT` and `UPDATE` changes as upsert messages and `DELETE` changes as delete messages. The stream consuming operator needs to be aware of the unique key attribute in order to apply messages correctly. 

   The primary difference to a retract stream is that `UPDATE` changes are encoded with a single message and hence more efficient. The following figure visualizes the conversion of a dynamic table into an upsert stream.

<center>
<img alt="Dynamic tables" src="{{ site.baseurl }}/fig/table-streaming/redo-mode.png" width="85%">
</center>
<br><br>

The API to convert a dynamic table into a `DataStream` is discussed on the [Common Concepts]({{ site.baseurl }}/dev/table/common.html#convert-a-table-into-a-datastream) page. Please note that only append and retract streams are supported when converting a dynamic table into a `DataStream`. The `TableSink` interface to emit a dynamic table to an external system are discussed on the [TableSources and TableSinks](../sourceSinks.html#define-a-tablesink) page.

{% top %} 

Joins in Continuous Queries
----------------------------

Joins are a common and well-understood operation in batch data processing to connect the rows of two relations. However, the semantics of joins on [dynamic tables](dynamic_tables.html) are much less obvious or even confusing.

Because of that, there are a couple of ways to actually perform a join using either Table API or SQL.

For more information regarding the syntax, please check the join sections in [Table API](../tableApi.html#joins) and [SQL]({{ site.baseurl }}/dev/table/sql/queries.html#joins).

### Regular Joins

Regular joins are the most generic type of join in which any new records or changes to either side of the join input are visible and are affecting the whole join result.
For example, if there is a new record on the left side, it will be joined with all of the previous and future records on the right side.

{% highlight sql %}
SELECT * FROM Orders
INNER JOIN Product
ON Orders.productId = Product.id
{% endhighlight %}

These semantics allow for any kind of updating (insert, update, delete) input tables.

However, this operation has an important implication: it requires to keep both sides of the join input in Flink's state forever.
Thus, the resource usage will grow indefinitely as well, if one or both input tables are continuously growing. This is because data for a key can arrive at any point in time in future or may never even arrive.

### Interval Joins

A interval join is defined by a join predicate, that checks if the [time attributes](time_attributes.html) of the input
records are within certain time constraints, i.e., a time window.

{% highlight sql %}
SELECT *
FROM
  Orders o,
  Shipments s
WHERE o.id = s.orderId AND
      o.ordertime BETWEEN s.shiptime - INTERVAL '4' HOUR AND s.shiptime
{% endhighlight %}

Compared to a regular join operation, this kind of join only supports append-only tables with time attributes. Since time attributes are quasi-monotonic increasing, Flink can remove old values from its state without affecting the correctness of the result.

### Join with a Temporal Table Function

A join with a temporal table function joins an append-only table (left input/probe side) with a temporal table (right input/build side),
i.e., a table that changes over time and tracks its changes. Please check the corresponding page for more information about [temporal tables](temporal_tables.html).

The following example shows an append-only table `Orders` that should be joined with the continuously changing currency rates table `RatesHistory`.

`Orders` is an append-only table that represents payments for the given `amount` and the given `currency`.
For example at `10:15` there was an order for an amount of `2 Euro`.

{% highlight sql %}
SELECT * FROM Orders;

rowtime amount currency
======= ====== =========
10:15        2 Euro
10:30        1 US Dollar
10:32       50 Yen
10:52        3 Euro
11:04        5 US Dollar
{% endhighlight %}

`RatesHistory` represents an ever changing append-only table of currency exchange rates with respect to `Yen` (which has a rate of `1`).
For example, the exchange rate for the period from `09:00` to `10:45` of `Euro` to `Yen` was `114`. From `10:45` to `11:15` it was `116`.

{% highlight sql %}
SELECT * FROM RatesHistory;

rowtime currency   rate
======= ======== ======
09:00   US Dollar   102
09:00   Euro        114
09:00   Yen           1
10:45   Euro        116
11:15   Euro        119
11:49   Pounds      108
{% endhighlight %}

Given that we would like to calculate the amount of all `Orders` converted to a common currency (`Yen`).

For example, we would like to convert the following order using the appropriate conversion rate for the given `rowtime` (`114`).

{% highlight text %}
rowtime amount currency
======= ====== =========
10:15        2 Euro
{% endhighlight %}

Without using the concept of [temporal tables](temporal_tables.html), one would need to write a query like:

{% highlight sql %}
SELECT
  SUM(o.amount * r.rate) AS amount
FROM Orders AS o,
  RatesHistory AS r
WHERE r.currency = o.currency
AND r.rowtime = (
  SELECT MAX(rowtime)
  FROM RatesHistory AS r2
  WHERE r2.currency = o.currency
  AND r2.rowtime <= o.rowtime);
{% endhighlight %}

With the help of a temporal table function `Rates` over `RatesHistory`, we can express such a query in SQL as:

{% highlight sql %}
SELECT
  o.amount * r.rate AS amount
FROM
  Orders AS o,
  LATERAL TABLE (Rates(o.rowtime)) AS r
WHERE r.currency = o.currency
{% endhighlight %}

Each record from the probe side will be joined with the version of the build side table at the time of the correlated time attribute of the probe side record.
In order to support updates (overwrites) of previous values on the build side table, the table must define a primary key.

In our example, each record from `Orders` will be joined with the version of `Rates` at time `o.rowtime`. The `currency` field has been defined as the primary key of `Rates` before and is used to connect both tables in our example. If the query were using a processing-time notion, a newly appended order would always be joined with the most recent version of `Rates` when executing the operation.

In contrast to [regular joins](#regular-joins), this means that if there is a new record on the build side, it will not affect the previous results of the join.
This again allows Flink to limit the number of elements that must be kept in the state.

Compared to [interval joins](#interval-joins), temporal table joins do not define a time window within which bounds the records will be joined.
Records from the probe side are always joined with the build side's version at the time specified by the time attribute. Thus, records on the build side might be arbitrarily old.
As time passes, the previous and no longer needed versions of the record (for the given primary key) will be removed from the state.

Such behaviour makes a temporal table join a good candidate to express stream enrichment in relational terms.

### Usage

After [defining temporal table function](temporal_tables.html#defining-temporal-table-function), we can start using it.
Temporal table functions can be used in the same way as normal table functions would be used.

The following code snippet solves our motivating problem of converting currencies from the `Orders` table:

<div class="codetabs" markdown="1">
<div data-lang="SQL" markdown="1">
{% highlight sql %}
SELECT
  SUM(o_amount * r_rate) AS amount
FROM
  Orders,
  LATERAL TABLE (Rates(o_proctime))
WHERE
  r_currency = o_currency
{% endhighlight %}
</div>
<div data-lang="java" markdown="1">
{% highlight java %}
Table result = orders
    .joinLateral("rates(o_proctime)", "o_currency = r_currency")
    .select("(o_amount * r_rate).sum as amount");
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val result = orders
    .joinLateral(rates('o_proctime), 'r_currency === 'o_currency)
    .select(('o_amount * 'r_rate).sum as 'amount)
{% endhighlight %}
</div>
</div>

**Note**: State retention defined in a [query configuration](query_configuration.html) is not yet implemented for temporal joins.
This means that the required state to compute the query result might grow infinitely depending on the number of distinct primary keys for the history table.

### Processing-time Temporal Joins

With a processing-time time attribute, it is impossible to pass _past_ time attributes as an argument to the temporal table function.
By definition, it is always the current timestamp. Thus, invocations of a processing-time temporal table function will always return the latest known versions of the underlying table
and any updates in the underlying history table will also immediately overwrite the current values.

Only the latest versions (with respect to the defined primary key) of the build side records are kept in the state.
Updates of the build side will have no effect on previously emitted join results.

One can think about a processing-time temporal join as a simple `HashMap<K, V>` that stores all of the records from the build side.
When a new record from the build side has the same key as some previous record, the old value is just simply overwritten.
Every record from the probe side is always evaluated against the most recent/current state of the `HashMap`.

### Event-time Temporal Joins

With an event-time time attribute (i.e., a rowtime attribute), it is possible to pass _past_ time attributes to the temporal table function.
This allows for joining the two tables at a common point in time.

Compared to processing-time temporal joins, the temporal table does not only keep the latest version (with respect to the defined primary key) of the build side records in the state
but stores all versions (identified by time) since the last watermark.

For example, an incoming row with an event-time timestamp of `12:30:00` that is appended to the probe side table
is joined with the version of the build side table at time `12:30:00` according to the [concept of temporal tables](temporal_tables.html).
Thus, the incoming row is only joined with rows that have a timestamp lower or equal to `12:30:00` with
applied updates according to the primary key until this point in time.

By definition of event time, [watermarks]({{ site.baseurl }}/dev/event_time.html) allow the join operation to move
forward in time and discard versions of the build table that are no longer necessary because no incoming row with
lower or equal timestamp is expected.

### Join with a Temporal Table

A join with a temporal table joins an arbitrary table (left input/probe side) with a temporal table (right input/build side),
i.e., an external dimension table that changes over time. Please check the corresponding page for more information about [temporal tables](temporal_tables.html#temporal-table).

<span class="label label-danger">Attention</span> Users can not use arbitrary tables as a temporal table, but need to use a table backed by a `LookupableTableSource`. A `LookupableTableSource` can only be used for temporal join as a temporal table. See the page for more details about [how to define LookupableTableSource](../sourceSinks.html#defining-a-tablesource-with-lookupable).

The following example shows an `Orders` stream that should be joined with the continuously changing currency rates table `LatestRates`.

`LatestRates` is a dimension table that is materialized with the latest rate. At time `10:15`, `10:30`, `10:52`, the content of `LatestRates` looks as follows:

{% highlight sql %}
10:15> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        114
Yen           1

10:30> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        114
Yen           1


10:52> SELECT * FROM LatestRates;

currency   rate
======== ======
US Dollar   102
Euro        116     <==== changed from 114 to 116
Yen           1
{% endhighlight %}

The content of `LastestRates` at time `10:15` and `10:30` are equal. The Euro rate has changed from 114 to 116 at `10:52`.

`Orders` is an append-only table that represents payments for the given `amount` and the given `currency`.
For example at `10:15` there was an order for an amount of `2 Euro`.

{% highlight sql %}
SELECT * FROM Orders;

amount currency
====== =========
     2 Euro             <== arrived at time 10:15
     1 US Dollar        <== arrived at time 10:30
     2 Euro             <== arrived at time 10:52
{% endhighlight %}

Given that we would like to calculate the amount of all `Orders` converted to a common currency (`Yen`).

For example, we would like to convert the following orders using the latest rate in `LatestRates`. The result would be:

{% highlight text %}
amount currency     rate   amout*rate
====== ========= ======= ============
     2 Euro          114          228    <== arrived at time 10:15
     1 US Dollar     102          102    <== arrived at time 10:30
     2 Euro          116          232    <== arrived at time 10:52
{% endhighlight %}


With the help of temporal table join, we can express such a query in SQL as:

{% highlight sql %}
SELECT
  o.amout, o.currency, r.rate, o.amount * r.rate
FROM
  Orders AS o
  JOIN LatestRates FOR SYSTEM_TIME AS OF o.proctime AS r
  ON r.currency = o.currency
{% endhighlight %}

Each record from the probe side will be joined with the current version of the build side table. In our example, the query is using the processing-time notion, so a newly appended order would always be joined with the most recent version of `LatestRates` when executing the operation. Note that the result is not deterministic for processing-time.

In contrast to [regular joins](#regular-joins), the previous results of the temporal table join will not be affected despite the changes on the build side. Also, the temporal table join operator is very lightweight and does not keep any state.

Compared to [interval joins](#interval-joins), temporal table joins do not define a time window within which the records will be joined.
Records from the probe side are always joined with the build side's latest version at processing time. Thus, records on the build side might be arbitrarily old.

Both [temporal table function join](#join-with-a-temporal-table-function) and temporal table join come from the same motivation but have different SQL syntax and runtime implementations:
* The SQL syntax of the temporal table function join is a join UDTF, while the temporal table join uses the standard temporal table syntax introduced in SQL:2011.
* The implementation of temporal table function joins actually joins two streams and keeps them in state, while temporal table joins just receive the only input stream and look up the external database according to the key in the record.
* The temporal table function join is usually used to join a changelog stream, while the temporal table join is usually used to join an external table (i.e. dimension table).

Such behaviour makes a temporal table join a good candidate to express stream enrichment in relational terms.

In the future, the temporal table join will support the features of temporal table function joins, i.e. support to temporal join a changelog stream.

### Usage

The syntax of temporal table join is as follows:

{% highlight sql %}
SELECT [column_list]
FROM table1 [AS <alias1>]
[LEFT] JOIN table2 FOR SYSTEM_TIME AS OF table1.proctime [AS <alias2>]
ON table1.column-name1 = table2.column-name1
{% endhighlight %}

Currently, only support INNER JOIN and LEFT JOIN. The `FOR SYSTEM_TIME AS OF table1.proctime` should be followed after temporal table. `proctime` is a [processing time attribute](time_attributes.html#processing-time) of `table1`.
This means that it takes a snapshot of the temporal table at processing time when joining every record from left table.

For example, after [defining temporal table](temporal_tables.html#defining-temporal-table), we can use it as following.

<div class="codetabs" markdown="1">
<div data-lang="SQL" markdown="1">
{% highlight sql %}
SELECT
  SUM(o_amount * r_rate) AS amount
FROM
  Orders
  JOIN LatestRates FOR SYSTEM_TIME AS OF o_proctime
  ON r_currency = o_currency
{% endhighlight %}
</div>
</div>

<span class="label label-danger">Attention</span> It is only supported in Blink planner.

<span class="label label-danger">Attention</span> It is only supported in SQL, and not supported in Table API yet.

<span class="label label-danger">Attention</span> Flink does not support event time temporal table joins currently.

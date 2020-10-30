---
title: "Planners"
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

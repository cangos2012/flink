---
title: "窗口函数"
weight: 6
type: docs
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

# Windowing table-valued functions (Windowing TVFs)

{{< label Streaming >}}

窗口是处理无限数据流的核心。窗口把流拆分成有限大小的桶“bucket”，我们可以将其应用于计算。
这篇文章聚焦的是在FLINK SQL中窗口是如何运转的，以及程序员们如何从其提供的功能中获得最大的好处。
Windows are at the heart of processing infinite streams. Windows split the stream into “buckets” of finite size, over which we can apply computations. This document focuses on how windowing is performed in Flink SQL and how the programmer can benefit to the maximum from its offered functionality.

Apache Flink提供了几个开窗函数（TVF）来将你的表中的元素分割到窗口中，包含以下：
Apache Flink provides several window table-valued functions (TVF) to divide the elements of your table into windows, including:
- [滚动窗口](#tumble)
- [Tumble Windows](#tumble)
- [滑动窗口](#hop)
- [Hop Windows](#hop)
- [累计窗口](#cumulate)
- [Cumulate Windows](#cumulate)
- 会话窗口（很快就会支持）
- Session Windows (will be supported soon)

需要注意的是，根据你使用的开窗函数，每个元素可能会被分到不止一个窗口中。例如：一个元素可能会落到滑动窗口中的多个交叠的窗口中。
Note that each element can logically belong to more than one window, depending on the windowing table-valued function you use. For example, HOP windowing creates overlapping windows wherein a single element can be assigned to multiple windows.

开窗函数是FLINK中定义的多态表函数（PTF）。PTF是SQL2016标准的一部分，是一个可以把表作为参数的特殊的表函数。
Windowing TVFs are Flink defined Polymorphic Table Functions (abbreviated PTF). PTF is part of the SQL 2016 standard, a special table-function, but can have a table as a parameter. PTF is a powerful feature to change the shape of a table. Because PTFs are used semantically like tables, their invocation occurs in a `FROM` clause of a `SELECT` statement.

Windowing TVFs is a replacement of legacy [Grouped Window Functions]({{< ref "docs/dev/table/sql/queries/window-agg" >}}#group-window-aggregation-deprecated). Windowing TVFs is more SQL standard compliant and more powerful to support complex window-based computations, e.g. Window TopN, Window Join. However, [Grouped Window Functions]({{< ref "docs/dev/table/sql/queries/window-agg" >}}#group-window-aggregation) can only support Window Aggregation.

See more how to apply further computations based on windowing TVF:
- [Window Aggregation]({{< ref "docs/dev/table/sql/queries/window-agg" >}})
- [Window TopN]({{< ref "docs/dev/table/sql/queries/window-topn">}})
- Window Join (will be supported soon)

## 开窗函数
## Window Functions

Apache Flink provides 3 built-in windowing TVFs: TUMBLE, `HOP` and `CUMULATE`. The return value of windowing TVF is a new relation that includes all columns of original relation as well as additional 3 columns named "window_start", "window_end", "window_time" to indicate the assigned window. The "window_time" field is a [time attributes]({{< ref "docs/dev/table/concepts/time_attributes" >}}) of the window after windowing TVF which can be used in subsequent time-based operations, e.g. another windowing TVF, or <a href="{{< ref "docs/dev/table/sql/queries/joins" >}}#interval-joins">interval joins</a>, <a href="{{< ref "docs/dev/table/sql/queries/over-agg" >}}">over aggregations</a>. The value of `window_time` always equal to `window_end - 1ms`.

### TUMBLE

The `TUMBLE` function assigns each element to a window of specified window size. Tumbling windows have a fixed size and do not overlap. For example, suppose you specify a tumbling window with a size of 5 minutes. In that case, Flink will evaluate the current window, and a new window started every five minutes, as illustrated by the following figure.

{{< img src="/fig/tumbling-windows.svg" alt="Tumbling Windows" width="70%">}}

The `TUMBLE` function assigns a window for each row of a relation based on a [time attribute]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column. The return value of `TUMBLE` is a new relation that includes all columns of original relation as well as additional 3 columns named "window_start", "window_end", "window_time" to indicate the assigned window. The original time attribute "timecol" will be a regular timestamp column after window TVF.

`TUMBLE` function takes three required parameters:

```sql
TUMBLE(TABLE data, DESCRIPTOR(timecol), size)
```

- `data`: is a table parameter that can be any relation with a time attribute column.
- `timecol`: is a column descriptor indicating which [time attributes]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column of data should be mapped to tumbling windows.
- `size`: is a duration specifying the width of the tumbling windows.

Here is an example invocation on the `Bid` table:

```sql
-- tables must have time attribute, e.g. `bidtime` in this table
Flink SQL> desc Bid;
+-------------+------------------------+------+-----+--------+---------------------------------+
|        name |                   type | null | key | extras |                       watermark |
+-------------+------------------------+------+-----+--------+---------------------------------+
|     bidtime | TIMESTAMP(3) *ROWTIME* | true |     |        | `bidtime` - INTERVAL '1' SECOND |
|       price |         DECIMAL(10, 2) | true |     |        |                                 |
|        item |                 STRING | true |     |        |                                 |
+-------------+------------------------+------+-----+--------+---------------------------------+

Flink SQL> SELECT * FROM Bid;
+------------------+-------+------+
|          bidtime | price | item |
+------------------+-------+------+
| 2020-04-15 08:05 |  4.00 | C    |
| 2020-04-15 08:07 |  2.00 | A    |
| 2020-04-15 08:09 |  5.00 | D    |
| 2020-04-15 08:11 |  3.00 | B    |
| 2020-04-15 08:13 |  1.00 | E    |
| 2020-04-15 08:17 |  6.00 | F    |
+------------------+-------+------+

-- NOTE: Currently Flink doesn't support evaluating individual window table-valued function,
--  window table-valued function should be used with aggregate operation,
--  this example is just used for explaining the syntax and the data produced by table-valued function.
Flink SQL> SELECT * FROM TABLE(
   TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES));
-- or with the named params
-- note: the DATA param must be the first
Flink SQL> SELECT * FROM TABLE(
   TUMBLE(
     DATA => TABLE Bid,
     TIMECOL => DESCRIPTOR(bidtime),
     SIZE => INTERVAL '10' MINUTES));
+------------------+-------+------+------------------+------------------+-------------------------+
|          bidtime | price | item |     window_start |       window_end |            window_time  |
+------------------+-------+------+------------------+------------------+-------------------------+
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
+------------------+-------+------+------------------+------------------+-------------------------+

-- apply aggregation on the tumbling windowed table
Flink SQL> SELECT window_start, window_end, SUM(price)
  FROM TABLE(
    TUMBLE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '10' MINUTES))
  GROUP BY window_start, window_end;
+------------------+------------------+-------+
|     window_start |       window_end | price |
+------------------+------------------+-------+
| 2020-04-15 08:00 | 2020-04-15 08:10 | 11.00 |
| 2020-04-15 08:10 | 2020-04-15 08:20 | 10.00 |
+------------------+------------------+-------+
```

*Note: in order to better understand the behavior of windowing, we simplify the displaying of timestamp values to not show the trailing zeros, e.g. `2020-04-15 08:05` should be displayed as `2020-04-15 08:05:00.000` in Flink SQL Client if the type is `TIMESTAMP(3)`.*


### HOP

The `HOP` function assigns elements to windows of fixed length. Like a `TUMBLE` windowing function, the size of the windows is configured by the window size parameter. An additional window slide parameter controls how frequently a hopping window is started. Hence, hopping windows can be overlapping if the slide is smaller than the window size. In this case, elements are assigned to multiple windows. Hopping windows are also known as "sliding windows".

For example, you could have windows of size 10 minutes that slides by 5 minutes. With this, you get every 5 minutes a window that contains the events that arrived during the last 10 minutes, as depicted by the following figure.

{{< img src="/fig/sliding-windows.svg" alt="Hopping windows" width="70%">}}

The `HOP` function assigns windows that cover rows within the interval of size and shifting every slide based on a [time attribute]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column. The return value of `HOP` is a new relation that includes all columns of original relation as well as additional 3 columns named "window_start", "window_end", "window_time" to indicate the assigned window. The original time attribute "timecol" will be a regular timestamp column after windowing TVF.

`HOP` takes three required parameters.

```sql
HOP(TABLE data, DESCRIPTOR(timecol), slide, size [, offset ])
```

- `data`: is a table parameter that can be any relation with an time attribute column.
- `timecol`: is a column descriptor indicating which [time attributes]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column of data should be mapped to hopping windows.
- `slide`: is a duration specifying the duration between the start of sequential hopping windows
- `size`: is a duration specifying the width of the hopping windows.

Here is an example invocation on the `Bid` table:

```sql
-- NOTE: Currently Flink doesn't support evaluating individual window table-valued function,
--  window table-valued function should be used with aggregate operation,
--  this example is just used for explaining the syntax and the data produced by table-valued function.
> SELECT * FROM TABLE(
    HOP(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '5' MINUTES, INTERVAL '10' MINUTES));
-- or with the named params
-- note: the DATA param must be the first
> SELECT * FROM TABLE(
    HOP(
      DATA => TABLE Bid,
      TIMECOL => DESCRIPTOR(bidtime),
      SLIDE => INTERVAL '5' MINUTES,
      SIZE => INTERVAL '10' MINUTES));
+------------------+-------+------+------------------+------------------+-------------------------+
|          bidtime | price | item |     window_start |       window_end |           window_time   |
+------------------+-------+------+------------------+------------------+-------------------------+
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:05 | 2020-04-15 08:15 | 2020-04-15 08:14:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:15 | 2020-04-15 08:25 | 2020-04-15 08:24:59.999 |
+------------------+-------+------+------------------+------------------+-------------------------+

-- apply aggregation on the hopping windowed table
> SELECT window_start, window_end, SUM(price)
  FROM TABLE(
    HOP(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '5' MINUTES, INTERVAL '10' MINUTES))
  GROUP BY window_start, window_end;
+------------------+------------------+-------+
|     window_start |       window_end | price |
+------------------+------------------+-------+
| 2020-04-15 08:00 | 2020-04-15 08:10 | 11.00 |
| 2020-04-15 08:05 | 2020-04-15 08:15 | 15.00 |
| 2020-04-15 08:10 | 2020-04-15 08:20 | 10.00 |
| 2020-04-15 08:15 | 2020-04-15 08:25 |  6.00 |
+------------------+------------------+-------+
```

### CUMULATE

Cumulating windows are very useful in some scenarios, such as tumbling windows with early firing in a fixed window interval. For example, a daily dashboard draws cumulative UVs from 00:00 to every minute, the UV at 10:00 represents the total number of UV from 00:00 to 10:00. This can be easily and efficiently implemented by CUMULATE windowing.

The `CUMULATE` function assigns elements to windows that cover rows within an initial interval of step size and expand to one more step size (keep window start fixed) every step until the max window size.
You can think `CUMULATE` function as applying `TUMBLE` windowing with max window size first, and split each tumbling windows into several windows with same window start and window ends of step-size difference. So cumulating windows do overlap and don't have a fixed size.

For example, you could have a cumulating window for 1 hour step and 1 day max size, and you will get windows: `[00:00, 01:00)`, `[00:00, 02:00)`, `[00:00, 03:00)`, ..., `[00:00, 24:00)` for every day.

{{< img src="/fig/cumulating-windows.png" alt="Cumulating Windows" width="70%">}}

The `CUMULATE` functions assigns windows based on a [time attribute]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column. The return value of `CUMULATE` is a new relation that includes all columns of original relation as well as additional 3 columns named "window_start", "window_end", "window_time" to indicate the assigned window. The original time attribute "timecol" will be a regular timestamp column after window TVF.

`CUMULATE` takes three required parameters.

```sql
CUMULATE(TABLE data, DESCRIPTOR(timecol), step, size)
```

- `data`: is a table parameter that can be any relation with an time attribute column.
- `timecol`: is a column descriptor indicating which [time attributes]({{< ref "docs/dev/table/concepts/time_attributes" >}}) column of data should be mapped to tumbling windows.
- `step`: is a duration specifying the increased window size between the end of sequential cumulating windows.
- `size`: is a duration specifying the max width of the cumulating windows. size must be an integral multiple of step .

Here is an example invocation on the Bid table:

```sql
-- NOTE: Currently Flink doesn't support evaluating individual window table-valued function,
--  window table-valued function should be used with aggregate operation,
--  this example is just used for explaining the syntax and the data produced by table-valued function.
> SELECT * FROM TABLE(
    CUMULATE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '2' MINUTES, INTERVAL '10' MINUTES));
-- or with the named params
-- note: the DATA param must be the first
> SELECT * FROM TABLE(
    CUMULATE(
      DATA => TABLE Bid,
      TIMECOL => DESCRIPTOR(bidtime),
      STEP => INTERVAL '2' MINUTES,
      SIZE => INTERVAL '10' MINUTES));
+------------------+-------+------+------------------+------------------+-------------------------+
|          bidtime | price | item |     window_start |       window_end |            window_time  |
+------------------+-------+------+------------------+------------------+-------------------------+
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:06 | 2020-04-15 08:05:59.999 |
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:08 | 2020-04-15 08:07:59.999 |
| 2020-04-15 08:05 |  4.00 | C    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:08 | 2020-04-15 08:07:59.999 |
| 2020-04-15 08:07 |  2.00 | A    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:09 |  5.00 | D    | 2020-04-15 08:00 | 2020-04-15 08:10 | 2020-04-15 08:09:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:12 | 2020-04-15 08:11:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:14 | 2020-04-15 08:13:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:16 | 2020-04-15 08:15:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:18 | 2020-04-15 08:17:59.999 |
| 2020-04-15 08:11 |  3.00 | B    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:14 | 2020-04-15 08:13:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:16 | 2020-04-15 08:15:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:18 | 2020-04-15 08:17:59.999 |
| 2020-04-15 08:13 |  1.00 | E    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:18 | 2020-04-15 08:17:59.999 |
| 2020-04-15 08:17 |  6.00 | F    | 2020-04-15 08:10 | 2020-04-15 08:20 | 2020-04-15 08:19:59.999 |
+------------------+-------+------+------------------+------------------+-------------------------+

-- apply aggregation on the cumulating windowed table
> SELECT window_start, window_end, SUM(price)
  FROM TABLE(
    CUMULATE(TABLE Bid, DESCRIPTOR(bidtime), INTERVAL '2' MINUTES, INTERVAL '10' MINUTES))
  GROUP BY window_start, window_end;
+------------------+------------------+-------+
|     window_start |       window_end | price |
+------------------+------------------+-------+
| 2020-04-15 08:00 | 2020-04-15 08:06 |  4.00 |
| 2020-04-15 08:00 | 2020-04-15 08:08 |  6.00 |
| 2020-04-15 08:00 | 2020-04-15 08:10 | 11.00 |
| 2020-04-15 08:10 | 2020-04-15 08:12 |  3.00 |
| 2020-04-15 08:10 | 2020-04-15 08:14 |  4.00 |
| 2020-04-15 08:10 | 2020-04-15 08:16 |  4.00 |
| 2020-04-15 08:10 | 2020-04-15 08:18 | 10.00 |
| 2020-04-15 08:10 | 2020-04-15 08:20 | 10.00 |
+------------------+------------------+-------+
```

{{< top >}}

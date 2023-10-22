---
sidebar_position: 2
---

# Materialized Views and Stream Processing

The core functionality of RisingWave is stream processing, and the way stream processing is presented in streaming databases is through materialized views.
This section explains the purpose and usage of RisingWave materialized views.

## Why is RisingWave's Materialized View Unique?

Materialized views are not unique to streaming databases. In fact, traditional databases such as PostgreSQL, data warehouses like Redshift and Snowflake, and real-time OLAP databases like ClickHouse and Apache Druid, all have materialized view capabilities. However, RisingWave's materialized views have several important features compared to those in other databases:


* **Real-time**: Many databases update materialized views asynchronously or require users to update them manually. But on RisingWave, materialized views are updated synchronously, so users can always query the freshest results. Even for complex queries with join, windowing, etc., RisingWave can efficiently handle them synchronously, ensuring the freshness of materialized views;

* **Consistency**: Some databases only provide eventually consistent materialized views, meaning the results on materialized views seen by users are approximate or erroneous. Especially when users create multiple materialized views, due to different refresh strategies for each materialized view, it's hard to see consistent results across materialized views. However, materialized views on RisingWave are consistent; even when accessing multiple materialized views, users will always see correct results, without inconsistencies;

* **High Availability**: RisingWave persists materialized views and sets high-frequency checkpoints to ensure rapid fault recovery. When the physical nodes hosting RisingWave crash, RisingWave can achieve second-level recovery and update computation results to the latest state within seconds;

* **Stream Processing Semantics**: In the stream processing domain, users can use higher-order syntax, like time windows, watermarks, etc., to process data streams. Traditional databases do not have these semantics, so users often rely on external systems to handle these semantics. RisingWave is a stream processing system, equipped with various complex stream processing semantics, allowing users to operate with SQL statements.

## Stream Processing without Materialized Views

In RisingWave, although materialized views are an important presentation of stream processing, it does not mean that users can only create materialized views to perform stream processing. In fact, for simple ETL computations, i.e., using RisingWave merely as a stream processing pipeline to process data generated by upstream systems and send it to downstream systems, there is no need to use materialized views. Users can simply use the [`create sink` statement](https://docs.risingwave.com/docs/current/sql-create-sink/) to directly perform stream processing and export the results.


## Sample Code


Most of you are likely familiar with how to create materialized views in PostgreSQL. Here, we demonstrate how to create stacked materialized views in RisingWave, that is, overlaying materialized views on top of materialized views.

We want to create tables `t1` and `t2`, as well as sources `s1` and `s2`. After that, we create a materialized view `mv1` on top of `t1` and `t2`, then materialized view `mv2` on top of `mv1` and `mv2`. Finally, we create a materialized view `mv` on `mv1` and `mv2`.

Let's first create `t1` `t2` `s1` `s2`:

```sql
CREATE TABLE t1 (v1 int, v2 int) 
WITH (
     connector = 'datagen',

     fields.v1.kind = 'sequence',
     fields.v1.start = '1',
  
     fields.v2.kind = 'random',
     fields.v2.min = '-10',
     fields.v2.max = '10',
     fields.v2.seed = '1',
  
     datagen.rows.per.second = '10'
 ) ROW FORMAT JSON;

CREATE TABLE t2 (v3 int, v4 int) 
WITH (
     connector = 'datagen',

     fields.v3.kind = 'sequence',
     fields.v3.start = '1',
  
     fields.v4.kind = 'random',
     fields.v4.min = '-10',
     fields.v4.max = '10',
     fields.v4.seed = '1',
  
     datagen.rows.per.second = '10'
 ) ROW FORMAT JSON;

CREATE SOURCE s1 (w1 int, w2 int) 
WITH (
     connector = 'datagen',
  
     fields.w1.kind = 'sequence',
     fields.w1.start = '1',
  
     fields.w2.kind = 'random',
     fields.w2.min = '-10',
     fields.w2.max = '10',
     fields.w2.seed = '1',

     datagen.rows.per.second = '10'
 ) ROW FORMAT JSON;


CREATE SOURCE s2 (w3 int, w4 int) 
WITH (
     connector = 'datagen',
  
     fields.w3.kind = 'sequence',
     fields.w3.start = '1',
  
     fields.w4.kind = 'random',
     fields.w4.min = '-10',
     fields.w4.max = '10',
     fields.w4.seed = '1',

     datagen.rows.per.second = '10'
 ) ROW FORMAT JSON;
```

Then create materialized views `mv1` and `mv2`:

```sql
create materialized view mv1 as select v2, w2 from t1, s1 where v1 = w1;
create materialized view mv2 as select v4, w4 from t2, s2 where v3 = w3;
```

Finally we create `mv`:

```sql
create materialized view mv as select w2, w4 from mv1, mv2 where v2 = v4;
```

Let's verify whether the stacked materialized views have been updated in a timely manner. We will repeatedly perform the following query:


```sql
select count(*) from mv;
```

We should see the results continuously changing. Below are the sample results:


```sql
 count
-------
  8092
(1 row)
```
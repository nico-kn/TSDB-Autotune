# Continuous Aggregates Experiments (TimescaleDB)

When querying time-ranges, query latency increases over increasing time spans. This expected behavior can be seen in the figure below:

![Timescale Query Interval Increasing](img/TS_QUERY_INT_INC_QUERY.png)
![Timescale Query Interval Increasing](img/TS_QUERY_INT_INC_QUERY_2.png)

Test List:
- AGG_TEST_01: No continuous queries specified. Ingestion test, query test with increasing query_interval

## AGG_TEST_01

First results as follows:

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "5"  | "20"             | "20"            | "20"             | "300"            | "100"         | "10"         | "1"                  | "5000"        | "150515\.6"             |
| "2"  | "20"             | "20"            | "20"             | "300"            | "100"         | "20"         | "1"                  | "5000"        | "120539\.77"            |

Ingestion Throughput for 12000000 points with no continuous aggregates present is at 120539.77 points per second. For 6000000 points even only at 117169.53 points per sec.

For query_interval of 2000000 (= 2000 sec or 33.3 min), the average latencies are: AGG_RANGE 113.90, AGG_VALUE 797.14 and AGG_RANGE_VALUE 317.38 (ms).

Now introduce continuous aggregate on query like

```sql
-- aggRangeQuery like:
EXPLAIN ANALYZE -- will show how long the query executed
SELECT device, count(s_214) FROM benchmark WHERE (device='d_1') 
    AND (time >= 1579449640000 and time <= 1580054440000) -- time range of one week
    GROUP BY device
```

results in:
```
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.00..28202.55 rows=200 width=12) (actual time=35.183..35.183 rows=1 loops=1)
   Group Key: _hyper_1_1_chunk.device
   ->  Append  (cost=0.00..28180.60 rows=3988 width=12) (actual time=0.096..34.653 rows=3976 loops=1)
         ->  Seq Scan on _hyper_1_1_chunk  (cost=0.00..28180.60 rows=3988 width=12) (actual time=0.095..34.290 rows=3976 loops=1)
               Filter: (("time" >= '1579449640000'::bigint) AND ("time" <= '1580054440000'::bigint) AND (device = 'd_1'::text))
               Rows Removed by Filter: 76024
 Planning time: 0.657 ms
 Execution time: 35.221 ms
(8 rows)
```

Because the table saves time in BIGINT in millis since epoch, its hypertable time column needs to be set with an integer_now function before issuing the continuous aggregate: 

```sql
CREATE OR REPLACE FUNCTION unix_now() returns BIGINT LANGUAGE SQL STABLE as $$ SELECT (extract(epoch from now()) * 1000) ::BIGINT $$;

SELECT set_integer_now_func('benchmark', 'unix_now');
```

the continuous aggregate will look like:
```sql
CREATE VIEW device_summary
WITH (timescaledb.continuous) --This flag is what makes the view continuous
AS
SELECT
  time_bucket('60000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 60000 = 1 min
  device,
  count(s_214) as metric_count --We can use any parallelizable aggregate
FROM
  benchmark
GROUP BY bucket, device; --We have to group by the bucket column, but can also add other group-by columns
```

results in 

```
NOTICE:  adding index _materialized_hypertable_2_device_bucket_idx ON _timescaledb_internal._materialized_hypertable_2 USING BTREE(device, bucket)
```

The original query will then not experience any performance improvements. Instead, the view needs to be called explicitly like

```sql
EXPLAIN ANALYZE -- will show how long the query executed
SELECT * FROM device_summary
WHERE device = 'd_1'
  AND bucket >= '1579449640000' AND bucket < '1580054440000'; -- time format like  2020-02-25 13:08:39.912372+00
```

this resulted in the following analyze statement:

```
                                                                                      QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=33.75..35.82 rows=166 width=20) (actual time=0.390..0.428 rows=166 loops=1)
   Group Key: _hyper_2_2_chunk.bucket, _hyper_2_2_chunk.device
   ->  Append  (cost=4.60..32.50 rows=166 width=21) (actual time=0.041..0.150 rows=166 loops=1)
         ->  Bitmap Heap Scan on _hyper_2_2_chunk  (cost=4.60..32.50 rows=166 width=21) (actual time=0.041..0.135 rows=166 loops=1)
               Recheck Cond: ((device = 'd_1'::text) AND (bucket >= '1579449640000'::bigint) AND (bucket < '1580054440000'::bigint))
               Heap Blocks: exact=25
               ->  Bitmap Index Scan on _hyper_2_2_chunk__materialized_hypertable_2_device_bucket_idx  (cost=0.00..4.56 rows=166 width=0) (actual time=0.032..0.032 rows=166 loops=1)
                     Index Cond: ((device = 'd_1'::text) AND (bucket >= '1579449640000'::bigint) AND (bucket < '1580054440000'::bigint))
 Planning time: 0.331 ms
 Execution time: 0.512 ms
(10 rows)
```

So execution time for the count statement could be reduced from 35.221 ms to 0.512 ms.

When setting up the continuous aggregate query, peak CPU consumption rose to 43% for a short period of time:

![CPU rise](img/cpu_rise_cont_aggs.PNG)

After issuing this one continuous aggregate (with vanilla settings), ingestion performed as follows:

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "5"  | "20"             | "20"            | "20"             | "300"            | "100"         | "10"         | "1"                  | "5000"        | "122117.04"             |

So no real difference concerning ingestion throughput, but with vanilla settings (optimized to not amplify ingestion latency).

Therefore, I tuned the hyperparameters to a more real-time continuous aggregate scenario like so. From 12 hours to 5 sec refresh interval and from 120000 ms to 5000 ms refresh_lag.

```sql
ALTER VIEW device_summary SET (timescaledb.refresh_interval = '5 sec');
ALTER VIEW device_summary SET (timescaledb.refresh_lag = '5000');
```

When altering these settings, the recalculation of the view was performed immediately because intermediate new data was inserted.

![CPU rise again](img/cpu_rise_cont_aggs2.PNG)

After altering the view, I reran the ingestion scenario. This was the result:

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "5"  | "20"             | "20"            | "20"             | "300"            | "100"         | "10"         | "1"                  | "5000"        | "121782.17"             |

A bit lower but not really significant.

Let's change the benchmark settings to be more appropriate for measuring this behavior (only one sensor and only inserting for this one). This low cardinality is very much not ideal for timescaledb, as CPU load in this scenario is very high! Almost 100% all the time the ingestion test ran.

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "6"  | "20"             | "1"             | "1"              | "1"              | "5000"        | "50"         | "1"                  | "5000"        | "21642.48"              |

![CPU during regular ingestion scenario](img/cpu_during_ingestion_regular.PNG)

Now execute the following aggRange query:

```sql
-- aggRangeQuery like:
EXPLAIN ANALYZE -- will show how long the query executed
SELECT device, count(s_0) FROM agg_test_04 WHERE (device='d_0') 
    AND (time >= 1579449640000 and time <= 1580054440000) -- time range of one week
    GROUP BY device
```
Count will measure 120961 data points within that time span but needed to count them all from the raw data. Analyze query results in the following:

```
                                                                                    QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 GroupAggregate  (cost=0.42..4857.66 rows=200 width=12) (actual time=45.801..45.801 rows=1 loops=1)
   Group Key: _hyper_4_93_chunk.device
   ->  Append  (cost=0.42..4250.86 rows=120960 width=12) (actual time=0.014..34.376 rows=120961 loops=1)
         ->  Index Scan using _hyper_4_93_chunk_agg_test_04_time_idx on _hyper_4_93_chunk  (cost=0.42..2224.05 rows=63369 width=12) (actual time=0.013..14.496 rows=63369 loops=1)
               Index Cond: (("time" >= '1579449640000'::bigint) AND ("time" <= '1580054440000'::bigint))
               Filter: (device = 'd_0'::text)
         ->  Index Scan using _hyper_4_92_chunk_agg_test_04_time_idx on _hyper_4_92_chunk  (cost=0.42..2026.82 rows=57591 width=12) (actual time=0.018..12.364 rows=57592 loops=1)
               Index Cond: (("time" >= '1579449640000'::bigint) AND ("time" <= '1580054440000'::bigint))
               Filter: (device = 'd_0'::text)
 Planning time: 0.539 ms
 Execution time: 50.603 ms
(11 rows)
```

Build continuous aggregate on this:

```sql
CREATE VIEW device_count
WITH (timescaledb.continuous) --This flag is what makes the view continuous
AS
SELECT
  time_bucket('60000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 60000 = 1 min
  device,
  count(s_0) as metric_count --We can use any parallelizable aggregate
FROM
  agg_test_04
GROUP BY bucket, device; --We have to group by the bucket column, but can also add other group-by columns
```

Now explicitly query the view:

```sql
EXPLAIN ANALYZE -- will show how long the query executed
SELECT SUM(metric_count) FROM device_count
WHERE device = 'd_0'
  AND bucket >= '1579449640000' AND bucket < '1580054440000'; -- time format like  2020-02-25 13:08:39.912372+00
```

This will yield 120960 as an aggregated count (off by 1) on the buckets within this time range. Query will execute as fast as:

```
                                                                                                    QUERY PLAN                                                                                                
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=481.20..481.21 rows=1 width=32) (actual time=23.693..23.693 rows=1 loops=1)
   ->  HashAggregate  (cost=454.52..467.86 rows=1067 width=20) (actual time=20.584..22.926 rows=10080 loops=1)
         Group Key: _hyper_5_133_chunk.bucket, _hyper_5_133_chunk.device
         ->  Append  (cost=0.29..374.49 rows=10671 width=21) (actual time=0.041..5.022 rows=10080 loops=1)
               ->  Index Scan Backward using _hyper_5_133_chunk__materialized_hypertable_5_bucket_idx on _hyper_5_133_chunk  (cost=0.29..374.49 rows=10671 width=21) (actual time=0.040..4.031 rows=10080 loops=1)
                     Index Cond: ((bucket >= '1579449640000'::bigint) AND (bucket < '1580054440000'::bigint))
                     Filter: (device = 'd_0'::text)
 Planning time: 0.626 ms
 Execution time: 19.891 ms
(9 rows)
```
So this will reduce the latency of the count query to something smaller than 1/2 of the original query time. Now re-execute the same ingestion test, yielding the following throughput, which is only marginally lower.

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "6"  | "20"             | "1"             | "1"              | "1"              | "5000"        | "50"         | "1"                  | "5000"        | "20606.45"              |

Now set the hyperparameters to a more real-time aggregation scenario:

```sql
ALTER VIEW device_count SET (timescaledb.refresh_interval = '5 sec');
ALTER VIEW device_count SET (timescaledb.refresh_lag = '5000');
```

And rerun the ingestion benchmark test. Following results:

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "6"  | "20"             | "1"             | "1"              | "1"              | "5000"        | "50"         | "1"                  | "5000"        | "19818.99"              |

Throughput not that different for ingestion but CPU load much higher and stayed high for a longer period of time with many workers recalculating the view:

![CPU during ingestion compare continuous aggregation settings](img/cpu_during_ingestion_compare_cont_agg_settings.PNG)

After more data points have been inserted, the regular SQL query counts 362883 data points for the specified interval while the view query counts 362880 (off by 3). 
- Regular query Execution time: 159.562 ms
- Cont. Agg query Execution time: 21.899 ms

## Average instead of count:

```sql
CREATE VIEW device_average
WITH (timescaledb.continuous) --This flag is what makes the view continuous
AS
SELECT
  time_bucket('60000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 60000 = 1 min
  device,
  avg(s_0) as metric_avg --We can use any parallelizable aggregate
FROM
  agg_test_04
GROUP BY bucket, device; --We have to group by the bucket column, but can also add other group-by columns
```

```sql
-- aggRangeQuery like:
EXPLAIN ANALYZE -- will show how long the query executed
SELECT device, avg(s_0) FROM agg_test_04 WHERE (device='d_0') 
    AND (time >= 1579449640000 and time <= 1580054440000) -- time range of one week
    GROUP BY device; 
``` 
Execution time: 153.490 ms, Result: 3.98699704863508

```sql
ALTER VIEW device_average SET (timescaledb.refresh_interval = '5 sec');
ALTER VIEW device_average SET (timescaledb.refresh_lag = '5000');
```

```sql
EXPLAIN ANALYZE -- will show how long the query executed
SELECT AVG(metric_avg) FROM device_average
WHERE device = 'd_0'
  AND bucket >= '1579449640000' AND bucket < '1580054440000'; -- time format like  2020-02-25 13:08:39.912372+00
```
Execution time: 27.909 ms, Result:  3.98698139880956

CPU consumption when the continuous aggregate got enabled: 
![CPU when enable cont. aggregate](img/cpu_consumption_when_cont_agg_enabled.PNG)

Rerun ingestion test with ingestion throughput significantly lowered to 18469.41 points/s.

| "id" | "CLIENT\_NUMBER" | "GROUP\_NUMBER" | "DEVICE\_NUMBER" | "SENSOR\_NUMBER" | "BATCH\_SIZE" | "LOOP\_RATE" | "REAL\_INSERT\_RATE" | "POINT\_STEP" | "INGESTION\_THROUGHPUT" |
|------|------------------|-----------------|------------------|------------------|---------------|--------------|----------------------|---------------|-------------------------|
| "7"  | "20"             | "1"             | "1"              | "1"              | "5000"        | "50"         | "1"                  | "5000"        | "18469.41"              |

CPU consumption when both continuous aggregates (count and average) got enabled: 
![CPU when enable cont. aggregate](img/cpu_consumption_when_2_cont_agg_enabled.PNG)

# Experiment: granularities speedup

One query that will want aggregates of 1 week for one month of data. How much benefit will it gain for pre-aggregations of different granularities? But what are the implications? Ingestion speed, cpu load etc.

```sql
-- raw query
EXPLAIN ANALYZE
SELECT
  time_bucket('604800000', time) as week, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 604800000 = 1 week
  device,
  avg(s_0) as average --We can use any parallelizable aggregate
FROM
  agg_test_04
WHERE
  time >= '1579132800000' AND time <= '1581552000000' -- one month of data: Jan 16th til Feb 13th 2020
GROUP BY week, device ORDER BY week ASC;
```

yields result in Planning time: 1.234 ms Execution time: 1005.488 ms (average of 4 runs)

     week      | device |     average
---------------+--------+------------------
 1579132800000 | d_0    | 3.98709317129588
 1579737600000 | d_0    | 3.98704794973504
 1580342400000 | d_0    | 3.98718526785673
 1580947200000 | d_0    | 3.98695436507895
 1581552000000 | d_0    |             4.92

 On existing 1 min interval averages, the query looks like:

 ```sql
EXPLAIN ANALYZE -- will show how long the query executed
SELECT 
    time_bucket('604800000', bucket) as week, -- corresponds to one week
    AVG(metric_avg) as week_average
FROM device_average
WHERE device = 'd_0'
  AND bucket >= '1579132800000' AND bucket <= '1581552000000'
GROUP BY week ORDER BY week ASC;
```
resulting in  Planning time: 0.692 ms Execution time: 77.191 ms
     week      |   week_average
---------------+------------------
 1579132800000 | 3.98709317129632
 1579737600000 | 3.98704794973548
 1580342400000 | 3.98718526785717
 1580947200000 | 3.98695436507939
 1581552000000 | 4.77083333333333

However, the results of the last week differ quite a bit from 4.92 to 4.77.

Now issuing a few new continuous aggregates on different granularities, but all from raw data! not based on top of each other

```sql
-- 5 min avg
CREATE VIEW device_average_5_min
WITH (timescaledb.continuous, timescaledb.refresh_interval = '5 min', timescaledb.refresh_lag = '300000') --This flag is what makes the view continuous
AS
SELECT
  time_bucket('300000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 300.000 = 5 min
  device,
  avg(s_0) as metric_avg
FROM
  agg_test_04
GROUP BY bucket, device;

-- 1 hour avg
CREATE VIEW device_average_1_hour
WITH (timescaledb.continuous, timescaledb.refresh_interval = '1 hour', timescaledb.refresh_lag = '3600000') --This flag is what makes the view continuous
AS
SELECT
  time_bucket('3600000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 3.600.000 = 1 hour
  device,
  avg(s_0) as metric_avg
FROM
  agg_test_04
GROUP BY bucket, device;

-- 1 day avg
CREATE VIEW device_average_1_day
WITH (timescaledb.continuous, timescaledb.refresh_interval = '1 day', timescaledb.refresh_lag = '86400000') --This flag is what makes the view continuous
AS
SELECT
  time_bucket('86400000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 86.400.000 = 1 day
  device,
  avg(s_0) as metric_avg
FROM
  agg_test_04
GROUP BY bucket, device;

-- 1 week avg
CREATE VIEW device_average_1_week
WITH (timescaledb.continuous, timescaledb.refresh_interval = '1 week', timescaledb.refresh_lag = '604800000') --This flag is what makes the view continuous
AS
SELECT
  time_bucket('604800000', time) as bucket, --time_bucket is required, bucket will be of integer, as timestamp is in BIGINT. 604.800.000 = 1 week
  device,
  avg(s_0) as metric_avg
FROM
  agg_test_04
GROUP BY bucket, device;
```
`NOTICE:  adding index _materialized_hypertable_7_device_bucket_idx ON _timescaledb_internal._materialized_hypertable_7 USING BTREE(device, bucket)`

Queries:

```sql
EXPLAIN ANALYZE
SELECT 
    time_bucket('604800000', bucket) as week, -- corresponds to one week
    AVG(metric_avg) as week_average
FROM device_average_1_week -- interchange this view name
WHERE device = 'd_0'
  AND bucket >= '1579132800000' AND bucket <= '1581552000000'
GROUP BY week ORDER BY week ASC;
```

Results:
5 min averages --> Planning time: 0.557 ms Execution time: 22.159 ms
     week      |   week_average
---------------+------------------
 1579132800000 | 3.98709317129627
 1579737600000 | 3.98704794973542
 1580342400000 | 3.98718526785711
 1580947200000 | 3.98695436507933
 1581552000000 |           4.2275

1 hour averages --> Planning time: 0.367 ms Execution time: 2.566 ms
     week      |   week_average
---------------+------------------
 1579132800000 | 3.98709317129632
 1579737600000 | 3.98704794973547
 1580342400000 | 3.98718526785716
 1580947200000 | 3.98695436507938
 1581552000000 | 4.00598611111113

1 day averages --> Planning time: 0.332 ms Execution time: 0.183 ms
     week      |   week_average
---------------+------------------
 1579132800000 | 3.98709317129632
 1579737600000 | 3.98704794973547
 1580342400000 | 3.98718526785717
 1580947200000 | 3.98695436507939
 1581552000000 | 3.98752256944446

1 week averages --> Planning time: 0.589 ms Execution time: 0.194 ms
     week      |   week_average
---------------+------------------
 1579132800000 | 3.98709317129803
 1579737600000 | 3.98704794973719
 1580342400000 | 3.98718526785888
 1580947200000 | 3.98695436508111
** Only 4 result rows instead of 5 ** --> time window in filter would need to be adapted to retrieve 5 rows 

Now running ingestion tests, same as before. With settings specified
1 real-time cont. aggs, 4 vanilla --> 18292.80 points/s
2 real-time cont. aggs, 3 vanilla --> 16761.59 points/s
3 real-time cont. aggs, 2 vanilla --> 16479.44 points/s
4 real-time cont. aggs, 1 vanilla --> 15788.28 points/s
5 real-time cont. aggs, 0 vanilla --> 16200.98 points/s

```sql
ALTER VIEW device_average_1_week SET (timescaledb.refresh_interval = '5 sec');
ALTER VIEW device_average_1_week SET (timescaledb.refresh_lag = '5000');
```

CPU consumption was as follows for these 5 ingestion load tests:
![CPU Consumptions](img/cpu_consumption_when_5_cont_agg_enabled.PNG)

# Util

Get information about background jobs concerning continuous aggregates:
```sql
select * from timescaledb_information.continuous_aggregate_stats;
```
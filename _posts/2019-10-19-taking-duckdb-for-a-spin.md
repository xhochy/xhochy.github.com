---
layout: post
title: Taking DuckDB for a spin
feature_image: "/images/ben-pattinson-_Wo1Oq38tVU-unsplash.jpg"
---

_TL;DR: Recently, [DuckDB](https://www.duckdb.org/) a database that promises to become the SQLite-of-analytics, was released and I took it for an initial test drive_. Install it via `conda install python-duckdb` or `pip install duckdb`.

[sqlite](https://www.sqlite.org/index.html) is really nice solution when you want to work locally on any database-related code or just keep using SQL for handling relational data.
With being in-process, it gets rid of the additional overhead of running an extra service and make things much simpler to start/setup.
While this often comes at a certain cost the more your application grows, it definitely makes the local development environment much simpler.
Often, people in the big data / data science / data analytics world also use SQLite for local data aggregation, either because they are most-fluent in SQL or because the queries are meant later to be executed on a much larger dataset using of the typical MPP (massively parallel processing) database.

In contrast to the larger system, SQLite is a row-based data store whereas the distributed analytical engines are all columnar.
Thus this makes SQLite well-suited for workloads where you act on whole rows or in transaction-heavy scenarios.
In data analytics though, you do operations that work on a large volume of rows but there only on a subset of columns.
This requires a different kind of architecture to provide a good performance.
[DuckDB](https://www.duckdb.org/) is a recent addition to the vast world of databases that promises to full our needs: being simple by being embedaable into a process like SQLite but in contrast being architectured for analytical workloads and thus a good basis for doing (local) data science work.

While the current development focus of DuckDB is on features and correctness, it already shines with performance due to its architecture choices.
You should expect it to get faster in future but I already gave it a spin and fed [my simple machine learning example on the NYC taxi trip dataset](https://uwekorn.com/2019/08/22/why-the-nyc-trd-is-a-nice-training-dataset.html) into and compared the times with SQLite.

## Load data into DuckDB

DuckDB has a [DB-API 2.0](https://www.python.org/dev/peps/pep-0249/) interface which we will use later in this article but to load data into it, we are going to use the CSV export option.
As with all databases, this comes with a decent speed. We connect to the DB in Python and import the data using the following snippet:

```python
import duckdb
conn = duckdb.connect('ytd.duckdb')
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE yellow_tripdata_2016_01 (   
    VendorID bigint,
    tpep_pickup_datetime timestamp,
    tpep_dropoff_datetime timestamp,
    passenger_count bigint,
    trip_distance double,
    pickup_longitude double,
    pickup_latitude double,
    RatecodeID bigint,
    store_and_fwd_flag varchar,
    dropoff_longitude double,
    dropoff_latitude double,
    payment_type bigint,
    fare_amount double,
    extra double,
    mta_tax double,
    tip_amount double,
    tolls_amount double,
    improvement_surcharge double,
    total_amount double
) 
""")

cursor.execute("""
COPY yellow_tripdata_2016_01 FROM '/Users/uwe/Development/data-science-io-benchmarks/data/yellow_tripdata_2016-01.csv'
WITH HEADER
""")

cursor.close()
connection.close()
```

To compare the performance with SQLite, we load the same CSV:

```
% sqlite3 db.sqlite
SQLite version 3.24.0 2018-06-04 14:10:15
Enter ".help" for usage hints.
sqlite> mode csv
sqlite> .import /Users/uwe/Development/data-science-io-benchmarks/data/yellow_tripdata_2016-01.csv yellow_tripdata_2016_01
```

## COUNT DISTINCT

One of the more heavy operations in my example was the value count over all rows.
This is a similar situation here as this query basically benchmarks the hash table implementation.
We can use the exact same query here for DuckDB and SQLite.

```sql
SELECT
    COUNT(DISTINCT VendorID),
    -- COUNT(DISTINCT tpep_pickup_datetime),
    -- COUNT(DISTINCT tpep_dropoff_datetime),
    COUNT(DISTINCT passenger_count),
    COUNT(DISTINCT trip_distance),
    -- COUNT(DISTINCT pickup_longitude),
    -- COUNT(DISTINCT pickup_latitude),
    COUNT(DISTINCT RatecodeID),
    COUNT(DISTINCT store_and_fwd_flag),
    -- COUNT(DISTINCT dropoff_longitude),
    -- COUNT(DISTINCT dropoff_latitude),
    COUNT(DISTINCT payment_type),
    COUNT(DISTINCT fare_amount),
    COUNT(DISTINCT extra),
    COUNT(DISTINCT mta_tax),
    COUNT(DISTINCT tip_amount),
    COUNT(DISTINCT tolls_amount),
    COUNT(DISTINCT improvement_surcharge),
    COUNT(DISTINCT total_amount)
FROM yellow_tripdata_2016_01
```

```python
%%timeit
# DuckDB
cursor.execute(query)
cursor.fetchdf()
# 5.58 s ± 54 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

```python
%%time
# SQLite
pd.read_sql(query, conn)
# 25.2 s ± 263 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

Here we directly see that DuckDB has a much better performance than SQLite for the simple hash table case.
Furthermore, we also use here DuckDB's custom `fetchdf()` function that directly returns a `pandas.DataFrame`.
It actually fetches the data directly in C++ into `numpy` arrays without going the overhead through Python objects like `pd.read_sql` does for SQLite.

## Frequency of events

As part of the NYC taxi trip dataset introduction, I showed that the data has a decent frequency of events.
This type of query is a nice and simple form for testing the aggregation performance.

```sql
-- DuckDB
SELECT
    MIN(cnt),
    AVG(cnt),
    -- MEDIAN(cnt),
    MAX(cnt)
FROM
(
    SELECT 
        COUNT(*) as cnt
    FROM yellow_tripdata_2016_01
    GROUP BY  
        EXTRACT(DOY FROM tpep_pickup_datetime::DATE),
        EXTRACT(HOUR FROM tpep_pickup_datetime)
) stats
-- 2.05 s ± 22.7 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

```SQL
-- SQLite
SELECT
    MIN(cnt),
    AVG(cnt),
    -- MEDIAN(cnt),
    MAX(cnt)
FROM
(
    SELECT 
        COUNT(*) as cnt
    FROM yellow_tripdata_2016_01
    GROUP BY  
        strftime('%j', tpep_pickup_datetime),
        strftime('%H', tpep_pickup_datetime)
) AS stats
-- 10.2 s ± 40.7 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

## Simple fare regression

And as a final performance comparison, we have a look at the very simple machine learning example I have made in the NYC taxi trip dataset post.
Luckily, this is so simple that you can implement it mostly in SQL.
Still, as SQLite doesn't have a function to compute the standard deviation, I had to split the query for it into two parts and do some basic maths in Python.
In contrast, DuckDB has a `STDDEV_SAMP` function and thus we could do the whole regression in a single statement.

```python
# DuckDB
cursor.execute(f"""
SELECT
    (SUM(trip_distance * fare_amount) - SUM(trip_distance) * SUM(fare_amount) / COUNT(*)) /
    (SUM(trip_distance * trip_distance) - SUM(trip_distance) * SUM(trip_distance) / COUNT(*)) AS beta,
    AVG(fare_amount) AS avg_fare_amount,
    AVG(trip_distance) AS avg_trip_distance
FROM 
    yellow_tripdata_2016_01,
    (
        SELECT 
            AVG(fare_amount) + 3 * STDDEV_SAMP(fare_amount) as max_fare,
            AVG(trip_distance) + 3 * STDDEV_SAMP(trip_distance) as max_distance
        FROM yellow_tripdata_2016_01
    ) AS sub
WHERE 
    fare_amount > 0 AND
    fare_amount < sub.max_fare AND 
    trip_distance > 0 AND
    trip_distance < sub.max_distance
""")
beta, avg_fare_amount, avg_trip_distance = cursor.fetchone()
alpha = avg_fare_amount - beta * avg_trip_distance
# 972 ms ± 7.44 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

The same regression in SQLite but here we separated the query as there was not function to compute the standard deviation fully in SQL.

```python
cursor.execute("""
    SELECT 
        count(*),
        sub.avg_fa as avg_fare,
        sum((fare_amount - sub.avg_fa) * (fare_amount - sub.avg_fa)) as var_fare,
        sub.avg_td as avg_distance,
        sum((trip_distance - sub.avg_td) * (trip_distance - sub.avg_td)) as var_distance
    FROM 
        yellow_tripdata_2016_01,
        (
            SELECT
                AVG(fare_amount) as avg_fa,
                AVG(trip_distance) as avg_td
            FROM yellow_tripdata_2016_01
        ) as sub
""")
n, avg_fare, var_fare, avg_distance, var_distance = cursor.fetchone()
max_fare = avg_fare + 3 * math.sqrt(var_fare / (n - 1))
max_distance = avg_distance + 3 * math.sqrt(var_distance / (n - 1))
max_fare, max_distance

cursor.execute(f"""
SELECT
    (SUM(trip_distance * fare_amount) - SUM(trip_distance) * SUM(fare_amount) / COUNT(*)) /
    (SUM(trip_distance * trip_distance) - SUM(trip_distance) * SUM(trip_distance) / COUNT(*)) AS beta,
    AVG(fare_amount) AS avg_fare_amount,
    AVG(trip_distance) AS avg_trip_distance
FROM yellow_tripdata_2016_01
WHERE 
    fare_amount > 0 AND
    fare_amount < {max_fare} AND 
    trip_distance > 0 AND
    trip_distance < {max_distance}
""")
beta, avg_fare_amount, avg_trip_distance = cursor.fetchone()
alpha = avg_fare_amount - beta * avg_trip_distance

# 11.7 s ± 34 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

## Conclusion

As one can see in the above examples, usage of DuckDB and SQLite is very similar and thus DuckDB also provides the comfort of an easily embeddedable database.
This helps a lot with local testing of analytics as in the above examples performance is always 5-10x faster then the comparable query in SQLite.
At the current stage, we cannot make a bold statement like "DuckDB is 10x faster then SQLite", mostly because I expect DuckDB to get faster once the core development of it focuses on performance.
Also I haven't spent much time optimising the queries in this post, so the performance comparison here is more on the level "how fast is it when you prototype SQL" (yes, it's fast!).

Overall, the performance boost that DuckDB brings is very nice and I'm looking forward to when it is even faster.
Everyone in data analytics should definitely keep it on their horizon and use it when they want to work locally but still execute their analytical SQL.
There are still some things missing that to make it usable for me like [having an SQLAlchemy dialect](https://github.com/cwida/duckdb/issues/305).

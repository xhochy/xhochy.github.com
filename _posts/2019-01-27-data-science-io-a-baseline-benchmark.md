---
layout: post
title: Data Science I/O - A baseline benchmark for 2019
feature_image: "https://picsum.photos/2560/600?image=1063"
---

Data Science and Machine Learning are tasks that have their own requirements on I/O.
As many other tasks, they start out on tabular data in most cases.
In contrast to a typical reporting task, they don't work on aggregates but require the data on the most granular level.
Some machine learning algorithms are able to directly work on aggregates but most workflows pass over the data in its most granular form.

Although data science has been around for a while and the size of data we have to deal with are steadily increasing,
a lot of the drivers and interfaces are still optimised for the case of small amounts of row-wise data.
For the initial load into our data pipelines but also for checkpointing, we need to deal with larger amounts of rows.
The better the read/write times are, the less infrastructure cost we have and the faster are our cycle times on improving
our machine learning models.

I plan to investigate the available I/O options for data sources like file formats or databases that result in a pandas DataFrame.
This should get an overview on what options are available for a certain tool and how they compare to each other.
It also is personally for me a motivation to look at the different storage options available.

Furthermore, it will also be a field where Apache Arrow will play a big role.
With its memory format and libraries for fast processing of columnar data, we should be able to build better clients for these storage.
Better may mean in this case either "better performing" or "less code".
The former will be achieved by the tuned algorithms that are implemented as part of the Arrow libraries.
The latter is enabled by using Apache Arrow as set of building blocks to connect columnar systems.
As it will be used to connect multiple systems with each other, we can use it as the base for a lot of code that would otherwise be repeated when writing efficient clients.

The Environment
---------------

To have a stable performance testing environment, I will run all the benchmarks on the laptop where I also write my blog posts.
This is 2018 MacBook Pro 13" with touch bar, a `Intel(R) Core(TM) i5-8259U CPU @ 2.30GHz` and 16GiB of RAM.
While this may not have a sheer load of CPU power, it does represent a typical setup where a lot of machine learning is done.
In late 2018, most data science projects still have a data volume that can be handled on a single machine.
Additionally, while you might have a larger data size and bigger machines, scaling down the data so it fits on a single machine also provides the benefit of shorter cycle times.

I intend to provide for each benchmark a `conda` environment to run the benchmarks in and docker containers for all things that don't natively run on a Mac.
While the docker containers may not provide the perfect I/O performance for the storage system, we are mostly interested in the relative performance of the different drivers that connect to these systems.
My focus in the benchmarks will be on to/from Pandas but depending on my time and motivation, I may also extend it to R.

The Baseline
------------

There are two main things, we want to benchmark: READ and WRITE.
Due to the nature of the data type, every benchmark is run with and without strings as they can have a great impact on performance.
In comparison to numbers, strings can have a different length per row.
While one can also allocate a fixed row size for a string column, this often is impractical when there are a (few) long strings.

My personal suggestion is always to [Apache Parquet](https://parquet.apache.org/) as the format to persist data.
Your initial input and final output stores may be different things like a company's data warehouse but as long as you're in the experimentation phase of your project or only persist intermediate checkpoints in your pipeline, stick to Apache Parquet.
Parquet provides [efficient storage for most types of data](https://tech.blue-yonder.com/efficient-dataframe-storage-with-apache-parquet/) you store in a DataFrame and [provides decent performance with the default options](http://matthewrocklin.com/blog/work/2017/06/28/use-parquet).

For the setup of the benchmark, we are using the conda-forge packages as of 19th January 2019. We have setup the environment as follows:

```
conda create -p dsib-baseline-2019 python=3.7 pyarrow=0.11.1 pandas=0.23.4 jupyterlab -c conda-forge
source activate $(pwd)/dsib-baseline-2019
```

### NYC taxi trips as real life dataset

Instead of testing using artificial data, we're using the [New York Taxi & Limousine Commission Trip Record Data](http://www.nyc.gov/html/tlc/html/about/trip_record_data.shtml) from 2016.
This datasets contains a variety of types incl. datetimes and booleans.
As we also want to test the I/O with and without strings, we augment the dataset with a string column.
We store in the string the sh256 hash of the of the other columns in the row.
In a second variation, we store only the first three digits of the hash so that we have a string column but with a lot of duplicate values.
This should show the performance difference for readers between high entropy and categorical columns.

We convert the dataset of the first six month of 2016 with the following code from the original CSVs to Parquet:

```python
from pathlib import Path

import glob
import hashlib
import pandas as pd
import pyarrow as pa
import pyarrow.parquet as pq


csv_files = glob.glob('../data/yellow_tripdata_2016-*.csv')

for csv_file in csv_files:
    # Change the file suffix
    p = Path(csv_file)
    parquet_file = p.parent / f"{p.name[:-3]}parquet"
    str_parquet_file = p.parent / f"str_{p.name[:-3]}parquet"
    cat_parquet_file = p.parent / f"cat_{p.name[:-3]}parquet"
    
    # Read in the CSV and already convert datetime columns
    df = pd.read_csv(
        csv_file,
        parse_dates=["tpep_pickup_datetime", "tpep_dropoff_datetime"],
        index_col=False,
        infer_datetime_format=True,
    )
    
    # store_and_fwd_flag is actually boolean but read in as string.
    # Manually change it to have a more efficient storage.
    df['store_and_fwd_flag'] = df['store_and_fwd_flag'] == 'Y'
    
    # Store it with the default options:
    #  * a single RowGroup, no chunking
    #  * SNAPPY compression
    df.to_parquet(parquet_file, engine="pyarrow")
    
    df['str'] = df.apply(lambda x: hashlib.sha256(str(x).encode()).hexdigest(), axis = 1)
    df.to_parquet(str_parquet_file, engine="pyarrow")
    
    df['str'] = df['str'].apply(lambda s: f"{s[0]}-{s[1]}-{s[2]}")
    df.to_parquet(cat_parquet_file, engine="pyarrow")
```

#### Benchmark 1a: READ without strings

As the first benchmark, we take the simple case of reading in a file without a string column.
For a reader, this is the most simple case as all types are fixed-width, i.e. one is able to pre-allocate the necessary memory by just knowing the schema and the number of rows.
To avoid the filesystem cache interfering with the benchmark, we pre-load the file into Python memory.

```python
with open('../data/yellow_tripdata_2016-01.parquet', 'rb') as f:
    parquet_bytes = f.read()
```

With the data cached in RAM, we can now run the benchmark.
Thanks to the runtime of roughly a seconds, we have the nice throughput of 1.6GiB/s.
This includes loading the data from Parquet into Arrow and then converting it further to pandas.

```python
%%timeit
reader = pa.BufferReader(parquet_bytes)
df = pq.read_table(reader).to_pandas()
# 1.08 s ± 79.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 1b: READ with high entropy strings

When we add a string column to the DataFrame, the in-memory size increases to 2.8GiB.
Although this is less that double than the benchmark before, the runtime is nearly four times as high.
The main additional overhead here is the creation of the Python `str` objects.

```python
with open('../data/str_yellow_tripdata_2016-01.parquet', 'rb') as f:
    parquet_bytes = f.read()

%%timeit
reader = pa.BufferReader(parquet_bytes)
df = pq.read_table(reader).to_pandas()
# 3.68 s ± 75.1 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 1c: READ with low entropy strings

With the lower entropy strings, we have only a small increase in the size of the dataframe of about 30MiB.
As we still need to create some Python `str` objects, we see an increase of 40% in the runtime.
Due to the `str` objects being much smaller and with a larger rate of repetition in the data, we have overall a much smaller runtime increase than in the high entropy scenario.

```python
with open('../data/cat_yellow_tripdata_2016-01.parquet', 'rb') as f:
    parquet_bytes = f.read()

%%timeit
reader = pa.BufferReader(parquet_bytes)
df = pq.read_table(reader).to_pandas(categories=['str'])
# 1.45 s ± 8.24 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 2: continuous READ

As the second category of benchmarks, we look at reading data in bulk where the sum of the data is larger than the available RAM.
In the case of simple files, this does not really make a difference to the very first benchmark.
But for later blog posts, we need this as a baseline.
In the case of database clients, we can do a single large query for 6 months of data and then consume the result in batches.
This should then be much faster in comparison to requesting one month at a time as save the latency of submitting a query.

A typical application where this could be useful is online learning.
Here you would consume the data as it arrives but don't hold on to the result.

```python
files = glob.glob('../data/yellow_tripdata_2016-*.parquet')

%%timeit
for f in files:
    df = pq.read_table(f).to_pandas()
    # Here you would normally update e.g. your online algorithm.
    # We skip any work here as we only want to measure I/O time.

# 7.1 s ± 74.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3a: WRITE without strings

When we are done with our calculation or simply want to store an intermediate checkpoint, the write performance becomes critical.
For this `pyarrow` converts the `DataFrame` to a `pyarrow.Table` and then serialises it to Parquet.
In contrast to READ, we have not yet optimised this path in Apache Arrow yet, thus we are seeing over 5x slower performance compared to reading the data.
This is definitely something a follow-up blog post will cover once we had a glance at the bottlenecks in writing Parquet files.

```python
df = pq.read_table('../data/yellow_tripdata_2016-01.parquet').to_pandas()

%%timeit
buf = io.BytesIO()
df.to_parquet(buf, engine='pyarrow')
# 5.66 s ± 70.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3b: WRITE with high entropy strings

When we also add strings with near to no repetition to the equation, writing becomes twice as slow.
A main contributor here is the memory fragmentation through the Python `str` objects.
They add significantly to the amount of data that needs to be written but even higher is the cost to transfer the strings from the Python objects that are scattered in main memory into a contiguous Arrow array.

```python
df = pq.read_table('../data/str_yellow_tripdata_2016-01.parquet').to_pandas()

%%timeit
buf = io.BytesIO()
df.to_parquet(buf, engine='pyarrow')
# 10.1 s ± 95.8 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3c: WRTIE with low entropy strings

In contrast to the high entropy situation, we see a much slower runtime increase in the categorical case.
Instead of millions of strings, we only need to convert a few thousand Python objects into an Arrow array.
Technically, we first convert the `pd.Categorical` column into a `pyarrow.DictionaryArray` where we also only store the unique values in a single array and have an index array for the size of the whole DataFrame.
Only when we write to Parquet, we materialise the strings into a contiguous stream of binary characters.

```python
df = pq.read_table('../data/yellow_tripdata_2016-01.parquet').to_pandas(categories=["str"])

%%timeit
buf = io.BytesIO()
df.to_parquet(buf, engine='pyarrow')
# 6.56 s ± 88.5 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

The All-Star
------------

In addition to my personal favourite, we should not forget the Swiss army knife CSV as an alternative baseline.
Still, it's the most common data source for data science projects and the universal exchange format everyone and nearly every tool can read.
CSV is a human-readable format but due to its very wide-spread usage, the readers for it are often very efficient and can compete with a lot of binary formats.

Pandas already brings along its own built-in CSV reader. This is probably also pandas' most famous feature.
As our reference data also comes as a collection of CSV files, it is a natural fit to also include a CSV benchmark in the post.

As the high- and low-entropy string columns were artificially generated, we take the generated Parquet files and turn them back into CSVs again.

```python
for csv_file in csv_files:
    # Change the file suffix
    p = Path(csv_file)
    parquet_file = p.parent / f"{p.name[:-3]}parquet"
    str_parquet_file = p.parent / f"str_{p.name[:-3]}parquet"
    str_csv_file = p.parent / f"str_{p.name}"
    cat_parquet_file = p.parent / f"cat_{p.name[:-3]}parquet"
    cat_csv_files = p.parent / f"cat_{p.name}"
    
    df = pd.read_parquet(str_parquet_file)
    df.to_csv(str_csv_file, index=False)
    
    df = pd.read_parquet(cat_parquet_file)
    df.to_csv(cat_csv_files, index=False)
```

Depending on which of the three files we use, we can specify certain options on `pandas.read_csv` to improve the parsing drastically.
Although quite often used by myself, the [documentation of `pandas.read_csv`](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.read_csv.html#pandas-read-csv) is a very regular visit for me due to its vast amount of options.
Depending on your data, some of the options can make a magnitude difference in parsing speed.

We have two datetime columns in our data: `tpep_pickup_datetime` and `tpep_dropoff_datetime`.
We declare that these two are dates together with the `infer_datetime_format=True` flag.
This might not always be the optimal choice depending on your date format but in the case of the NYC taxi trip data, it improves the read performance.
When you want to see whether these flags improve the parsing speed on your data, you can try them on a subset of the CSV by limiting the number of rows read with `nrows` first.

```python
def read_nyc_csv(filename_or_buf, **kwargs):
    df = pd.read_csv(
        filename_or_buf,
        parse_dates=["tpep_pickup_datetime", "tpep_dropoff_datetime"],
        infer_datetime_format=True,
        index_col=False,
        **kwargs,
    )
```

#### Benchmark 1a: READ without strings

As with the Parquet benchmarks, we first load the CSV file into memory to remove the filesystem cache from the equation.

```python
with open('../data/yellow_tripdata_2016-01.csv', 'rb') as f:
    csv_bytes = f.read()
```

To get optimal performance and the `DataFrame` with the right types, we tell `read_csv` that the `store_and_fwd_flag` column should be boolean and also declare `Y` and `N` as the respective boolean values.
Thus we don't need to do any post-processing on the data and let `read_csv` handle the type conversions during read.
The final performance of the CSV reading is much slower than with the Parquet files.
This is acceptable given that CSV is human-readable and Parquet a highly optimised binary format.
On a local disk, this might seem slow but 40MiB/s (or 320Mbit/s) is what still be quite decent when you read these files over network.

```python
%%timeit
read_nyc_csv(
    io.BytesIO(csv_bytes),
    dtype={'store_and_fwd_flag': 'bool'},
    true_values=['Y'],
    false_values=['N']
)
# 41 s ± 190 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 1b: READ with high entropy strings

In the high entropy case, we sadly have to include the filesystem cache in our benchmarks as the combination of the Python I/O stack and `pandas.read_csv` cannot cope with in-memory buffers of more than 2.1GiB.
As the file is read several times and the machine we're running the benchmarks on has quite a decent amount of spare RAM, we assume that we're no reading from disk but that the file is cached in memory, just with the additional indirection of the cache being in the filesystem subsystem.

The CSV files with strings were generated by pandas already with the correct dtypes as shown in the code above.
Thus pandas already auto-detects that `store_and_fwd_flag` is a boolean column and we don't need to indicate that via any flags anymore.

From an absolute perspective, we see more than twice the increase in runtime but relatively seen this is less drastic than for Parquet files.
Although we have the overhead of the Python `str` objects in the final pandas column, we also have the larger amount of input data in a row-wise form.

```python
%%timeit
with open('../data/str_yellow_tripdata_2016-01.csv', 'rb') as f:
    read_nyc_csv(f)
# 52.2 s ± 250 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 1c: READ with low entropy strings

For the low-entropy case, we see roughly the same overhead as for any other column type.
It seems like here the most significant overhead is coming from the size of the increased input data.

```python
with open('../data/cat_yellow_tripdata_2016-01.csv', 'rb') as f:
    csv_bytes = f.read()

%%timeit
read_nyc_csv(io.BytesIO(csv_bytes), dtype={'str': 'category'})
# 44.1 s ± 184 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 2: continuous READ

As with Benchmark 2 in the Parquet case, we have covered the continuous READ here mostly for reference.
For files there is no practical difference to 1a, only that we also measure the disk overhead as we cannot keep the data fully in memory.

```python
files = glob.glob('../data/yellow_tripdata_2016-*.csv')

%%timeit
for f in files:
    read_nyc_csv(
        f,
        dtype={'store_and_fwd_flag': 'bool'},
        true_values=['Y'],
        false_values=['N'],
    )
    # Here you would normally update e.g. your online algorithm.
    # We skip any work here as we only want to measure I/O time.

# 4min 15s ± 2.98 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3a: WRITE without strings

For the write benchmarks, we take the (binary) Parquet files as the reference.
We can also see here the same factor in runtime increase between READ and WRITE as we have seen in the Parquet case.
Due to its already much higher base, the CSV writes are noticeably slow.
Using them as a checkpoint in your data pipeline will be one of the pain points that will quickly be optimised away once a co-worker is annoyed about total runtimes.

```python
df = pd.read_parquet('../data/yellow_tripdata_2016-01.parquet', engine='pyarrow')

%%timeit
df.to_csv();
# 3min 4s ± 262 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3b: WRITE with high entropy strings

Writing high-entropy strings adds twice the same absolute time as it did in the READ case.
This is actually better than we would have assumed.
Given that overall write is 5-6x slower, we also would have assumed that the additional string column is adding nearly a minute to the write.
One of the possible reasons is that during the READ, we needed to allocate a huge number of Python `str` objects whereas now we only have the additional amount of data to write to a file but we don't have to allocate any further Python objects.

```python
df = pd.read_parquet('../data/str_yellow_tripdata_2016-01.parquet')

%%timeit
df.to_csv();
# 3min 19s ± 761 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

#### Benchmark 3c: WRTIE with low entropy strings

For the additional categorical column, we see an increase in runtime between the no-string and high-entropy string case that correlates well with the additional size in the output.
Thus we see no big surprise, just decent performance here.

```python
df = pq.read_table('../data/cat_yellow_tripdata_2016-01.parquet').to_pandas(categories=["str"])

%%timeit
df.to_csv();
# 3min 12s ± 3.12 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

The timings in comparison
-------------------------

In all cases, we have seen that Parquet files are boosting better times than CSVs.
While CSVs provide near perfect portability and human-readability, their readable, row-wise notation comes at a cost.
Parquet files are binary, with a schema and columnar.
Thus they are a natural fit for DataFrames and the CPU.

| Benchmark         | Parquet | CSV        | Parquet Speedup |
|-------------------|---------|------------|-----------------|
| 1a READ           |   1.08s |      41.0s |             38x |
| 1b READ with str  |   3.68s |      52.2s |             14x |
| 1c READ with cat  |   1.45s |      44.1s |             30x |
| 2 cont. READ      |   7.10s | 4min 15.0s |             36x |
| 3a WRITE          |   5.66s | 3min  4.0s |             33x |
| 3b WRITE with str |  10.10s | 3min 19.0s |             20x |
| 3c WRITE with cat |   6.56s | 3min 12.0s |             28x |

What's ahead?
-------------

The main reason for this blog post was to have published baseline measurement.
In the next blog posts I either want to look at the relative performance of client for a certain storage technologies using the same data as in this post or compare the absolute performance of a storage to the timings here.
As a personal motivation, we can then use the timings in the upcoming posts to see whether Arrow brought a performance increase or whether it might be beneficial to make an Arrow client simply for performance's sake.
The benefit of a native Arrow client is not only that it will increase access performance but also enable more simple integration into other systems and languages.
But especially in the case of Python/pandas, an Apache Arrow based clients will most likely show significant decreases in runtime due to the highly tuned routines that come with the Arrow C++ and Python libraries.

Future posts will also include looking at how we can improve on the baselines here.
This can either come through new Arrow and pandas releases or by using slightly different paths.
Arrow nowadays also comes with its own CSV reader that should be more performant although not providing all options that come pandas' built-in one.
Furthermore, I'm actively developing [an implementation of pandas ExtensionArray interface for Apache Arrow](https://github.com/xhochy/fletcher) that should remove the `to_pandas()` overhead that currently contributes nearly half to the read runtime.
At the moment, it is sadly lacking support for writing Parquet and CSV files without going through many hurdles.
Once this is smoothed out, you can expect a follow-up blog post.

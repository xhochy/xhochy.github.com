---
layout: post
title: "Calculating levenshtein distances with fletcher"
feature_image: "/images/marija-zaric-k-yf-LTqOuA-unsplash.jpg"
---

*Levenshtein distance is a typical measure to compare two different strings. It gives you the minimal number of add, remove and replace operations to transition from one string to another.*

Recently I sent out [a tweet asking for uses of strings in `pandas`](https://twitter.com/xhochy/status/1318247155539333120) and [Ian Ozsvald responded with the example of calculating the Levenshtein distance between two questions](https://twitter.com/ianozsvald/status/1318518801453940737) in the [Quora Question Pairs dataset](https://www.kaggle.com/c/quora-question-pairs/data).
Intrigued by the 30 minute runtime on this dataset, I hoped to push the `fletcher` performance below the one minute mark. In the end I got much further.

## Reference benchmarks

To start working with this use case, we first need to recreate the original approach using `pandas.DataFrame.apply`.
This might not be Ian's exact approach back then when he did look at the dataset but is hopefully pretty close.
As a starter, we download the dataset using the `kaggle` cli and convert it to Parquet so that we can load it much faster on repeated runs of the code:

```bash
kaggle competitions download -c quora-question-pairs
unzip quora-question-pairs.zip
```

```python
from pathlib import Path
import pandas as pd

# Save to Parquet to save storage size
if not (Path("data") / "test.parquet").exists():
    df = pd.read_csv(Path("data") / "test.csv")
    df.to_parquet(Path("data") / "test.parquet")
```

Loading this data back into `pandas`, we observe that both string columns together take about half a gigabyte of RAM when using the current default `object` dtype.

```python
df = pd.read_parquet(Path("data") / "test.parquet")
df[["question1", "question2"]].info(memory_usage="deep")

# <class 'pandas.core.frame.DataFrame'>
# RangeIndex: 2345796 entries, 0 to 2345795
# Data columns (total 2 columns):
#  #   Column     Dtype 
# ---  ------     ----- 
#  0   question1  object
#  1   question2  object
# dtypes: object(2)
# memory usage: 527.9 MB
```

To compute the Levenshtein distance, we use the [`jellytext`](https://github.com/jamesturk/jellyfish) package:

```python
import jellyfish

def levenshtein_per_row(x, y):
    """Wrapper function to handle None entries."""
    if x is None or y is None:
        return -1
    else:
        return jellyfish.levenshtein_distance(x, y)

%timeit df.apply(lambda x: levenshtein_per_row(x["question1"], x["question2"]), axis=1)
# 51.4 s ± 1.16 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

Compared to what was mentioned in the tweets, already being below a minute sounds already quite fast which make the challenge just even more interesting.

A simple way to use all the CPUs in my MacBook is to use `dask` / `distributed` to spread the work among multiple cores.
To achieve that, we first split the into even chunks:

```python
# Split the dataframe into several chunks
df = pd.read_parquet(Path("data") / "test.parquet")
dfs = np.array_split(df, 32)
for i, chunk in enumerate(dfs):
    chunk.to_parquet(Path("data") / f"test-{i}.parquet")
```

Afterwards, we start a `LocalCluster` and load the data into RAM.
As a `persist` call in `distributed` will only load the data once a computation reaches this point, we do a small dummy computation to ensure the data loading is excluded from our algorithm benchmark.

```python
cluster = LocalCluster()
client = Client(cluster)
ddf = dd.read_parquet("data/test-*.parquet", engine="pyarrow").persist()
ddf["test_id"].sum().compute(), len(ddf)
```

Afterwards, we run the same code as we have run on the `pandas.DataFrame` but use the magic of `distributed`'s `Future` implementation to compute the result but don't pull anything into the local process.
Thus we should really only measure the execution and no additional communication.

```python
%%timeit
tasks = ddf.apply(lambda x: levenshtein_per_row(x["question1"], x["question2"]), axis=1, meta=(None, 'int64'))
result = client.compute(tasks)
wait(result)
# Cleanup so that it is re-run everytime
result.release()
# 11.9 s ± 7.56 s per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

Overall this means, that with `51s` in the serial and `11.9` in the parallel case, we already get quite a decent performance with `jellyfish`.
As `jellyfish` is backed by an efficient implementation in C, the main things we can improve on is that we omit the overhead going through the `PyObject` structure and that Arrow's string representation is much more cache-friendly.
In the case of strings the `PyObject` overhead is much smaller as in the numeric case as the payload data is much larger compared to the size of the `PyObject` header.

## Implementing Levenshtein distance using fletcher and numba

Instead of loading the data using `pandas` directly, we use `fletcher`'s wrapper `fletcher.read_parquet` to have the string columns available as `pyarrow.Array`-backed columns.

```python
import fletcher as fr

df = fr.read_parquet("data/test.parquet", continuous=True)
df[["question1", "question2"]].info(memory_usage="deep")

# RangeIndex: 2345796 entries, 0 to 2345795
# Data columns (total 2 columns):
#  #   Column     Dtype                      
# ---  ------     -----                      
#  0   question1  fletcher_continuous[string]
#  1   question2  fletcher_continuous[string]
# dtypes: fletcher_continuous[string](2)
# memory usage: 287.4 MB
```

Here we already see the nice fact that using an Arrow-backed string column only uses about half the size of RAM.
This is due to the missing `PyObject` overhead and that Arrow is encoding the data using `UTF-8` whereas Python uses an encoding based on [the size of the largest character as defined in PEP 393](https://www.python.org/dev/peps/pep-0393/).

Implementing Levenshtein distance through the textbook definition would be a bad idea as with every other algorithm.
Instead there is a [nice collection of implementations in wikibooks](https://en.wikibooks.org/wiki/Algorithm_Implementation/Strings/Levenshtein_distance#C) of which we took the following C implementation.
Note that we currently limit us to single-byte characters, i.e. if a multi-byte character occurs, it will occur in the size with the same amount as it has bytes.

```c
int levenshtein(char *s1, char *s2) {
    unsigned int s1len, s2len, x, y, lastdiag, olddiag;
    s1len = strlen(s1);
    s2len = strlen(s2);
    unsigned int column[s1len + 1];
    for (y = 1; y <= s1len; y++)
        column[y] = y;
    for (x = 1; x <= s2len; x++) {
        column[0] = x;
        for (y = 1, lastdiag = x - 1; y <= s1len; y++) {
            olddiag = column[y];
            column[y] = MIN3(column[y] + 1, column[y - 1] + 1, lastdiag + (s1[y-1] == s2[x - 1] ? 0 : 1));
            lastdiag = olddiag;
        }
    }
    return column[s1len];
}
```

We can translate the above per-row function into Python code suitable for numba as follows:

```python
@numba.jit(nogil=True, nopython=True)
def levenshtein_numba_row(s1, s1len, s2, s2len) -> int:
    """Compute the levenshtein distance for a single row."""

    # Ensure the inner loop is smaller
    if s1len > s2len:
        s1, s2 = s2, s1
        s1len, s2len = s2len, s1len

    column = np.arange(s1len + 1)

    for x in range(1, s2len + 1):
        column[0] = x
        lastdiag = x - 1
        for y in range(1, s1len + 1):
            olddiag = column[y]
            column[y] = min(
                column[y] + 1,
                min(column[y - 1] + 1, lastdiag + (0 if s1[y - 1] == s2[x - 1] else 1)),
            )
            lastdiag = olddiag

    return column[s1len]
```

Sadly, as this is only the function that will be applied per-row, we need to write the for-loop that goes over the arrays.
This happens in three separate functions:

 * A numba-jitted for-loop that loops over both arrays when both have no missing values.
 * A numba-jitted for-loop that loops over both arrays but first checks the valid bitmap for missing values before it calls the row-wise function.
 * A user-facing Python (not-numba-jitted) function that unpacks the `pyarrow.Array` instances into their raw buffers, dispatches to the correct for-loop and then reassembles the raw buffers at the end into a `pyarrow.Array` instance.

```python
@numba.jit(nogil=True, nopython=True)
def levenshtein_numba_no_null(
    length, offsets_buffer_a, data_buffer_a, offsets_buffer_b, data_buffer_b, out
):
    """Compute the levenshtein distance for a whole column without nulls."""
    for i in range(length):
        out[i] = levenshtein_numba_row(
            data_buffer_a[offsets_buffer_a[i] :],
            offsets_buffer_a[i + 1] - offsets_buffer_a[i],
            data_buffer_b[offsets_buffer_b[i] :],
            offsets_buffer_b[i + 1] - offsets_buffer_b[i],
        )


@numba.jit(nogil=True, nopython=True)
def levenshtein_numba_nulls(
    length,
    valid,
    offsets_buffer_a,
    data_buffer_a,
    offsets_buffer_b,
    data_buffer_b,
    out,
):
    """Compute the levenshtein distance for a whole column where nulls occur."""
    for i in range(length):
        # Check if one of the entries is null
        byte_offset = i // 8
        bit_offset = i % 8
        mask = np.uint8(1 << bit_offset)
        is_valid = valid[byte_offset] & mask

        if is_valid:
            out[i] = levenshtein_numba_row(
                data_buffer_a[offsets_buffer_a[i] :],
                offsets_buffer_a[i + 1] - offsets_buffer_a[i],
                data_buffer_b[offsets_buffer_b[i] :],
                offsets_buffer_b[i + 1] - offsets_buffer_b[i],
            )


def levenshtein_numba(a, b):
    """
    Compute the levenshtein distance for two fletcher.FletcherContinuousArray columns.

    This will dispatch to the respective methods that can deal with nulls
    or the omission of the validity bitmap.
    """

    if len(a) != len(b):
        raise ValueError("Arrays must be of same length")

    out = np.empty(len(a), dtype=int)

    offsets_buffer_a, data_buffer_a = _extract_string_buffers(a)
    offsets_buffer_b, data_buffer_b = _extract_string_buffers(b)

    if a.null_count == 0 and b.null_count == 0:
        levenshtein_numba_no_null(
            len(a),
            offsets_buffer_a,
            data_buffer_a,
            offsets_buffer_b,
            data_buffer_b,
            out,
        )
        return pa.array(out)
    else:
        valid = _merge_valid_bitmaps(a, b)
        levenshtein_numba_nulls(
            len(a),
            valid,
            offsets_buffer_a,
            data_buffer_a,
            offsets_buffer_b,
            data_buffer_b,
            out,
        )
        buffers = [pa.py_buffer(x) for x in [valid, out]]
        return pa.Array.from_buffers(pa.int64(), len(out), buffers)
```

With all this boilerplate in place, we can now run the function on top of the Arrow data contained in the `fletcher` columns.

```python
%timeit levenshtein = levenshtein_numba(df["question1"].values.data, df["question2"].values.data)
# 24.2 s ± 214 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

One possible optimisation that comes to my mind is that we have an allocation in the loop that we could avoid in most cases.
Thus we can restructure the loops to only call `numpy.empty` if we need a larger allocation:

```python
@numba.jit(nogil=True, nopython=True)
def levenshtein_numba_no_null(
    length, offsets_buffer_a, data_buffer_a, offsets_buffer_b, data_buffer_b, out
):
    """Compute the levenshtein distance for a whole column without nulls."""
    column = np.empty(128)

    for i in range(length):
        s1 = data_buffer_a[offsets_buffer_a[i] :]
        s1len = offsets_buffer_a[i + 1] - offsets_buffer_a[i]
        s2 = data_buffer_b[offsets_buffer_b[i] :]
        s2len = offsets_buffer_b[i + 1] - offsets_buffer_b[i]

        # Ensure the inner loop is smaller
        if s1len > s2len:
            s1, s2 = s2, s1
            s1len, s2len = s2len, s1len

        if len(column) < s1len + 1:
            column = np.empty(s1len + 1)

        for y in range(1, s1len + 1):
            column[y] = y

        for x in range(1, s2len + 1):
            column[0] = x
            lastdiag = x - 1
            for y in range(1, s1len + 1):
                olddiag = column[y]
                column[y] = min(
                    column[y] + 1,
                    min(
                        column[y - 1] + 1,
                        lastdiag + (0 if s1[y - 1] == s2[x - 1] else 1),
                    ),
                )
                lastdiag = olddiag

        out[i] = column[s1len]


@numba.jit(nogil=True, nopython=True)
def levenshtein_numba_nulls(
    length,
    valid,
    offsets_buffer_a,
    data_buffer_a,
    offsets_buffer_b,
    data_buffer_b,
    out,
):
    """Compute the levenshtein distance for a whole column where nulls occur."""
    column = np.empty(128)

    for i in range(length):
        # Check if one of the entries is null
        byte_offset = i // 8
        bit_offset = i % 8
        mask = np.uint8(1 << bit_offset)
        is_valid = valid[byte_offset] & mask

        if is_valid:
            s1 = data_buffer_a[offsets_buffer_a[i] :]
            s1len = offsets_buffer_a[i + 1] - offsets_buffer_a[i]
            s2 = data_buffer_b[offsets_buffer_b[i] :]
            s2len = offsets_buffer_b[i + 1] - offsets_buffer_b[i]

            # Ensure the inner loop is smaller
            if s1len > s2len:
                s1, s2 = s2, s1
                s1len, s2len = s2len, s1len

            if len(column) < s1len + 1:
                column = np.empty(s1len + 1)

            for y in range(1, s1len + 1):
                column[y] = y

            for x in range(1, s2len + 1):
                column[0] = x
                lastdiag = x - 1
                for y in range(1, s1len + 1):
                    olddiag = column[y]
                    column[y] = min(
                        column[y] + 1,
                        min(
                            column[y - 1] + 1,
                            lastdiag + (0 if s1[y - 1] == s2[x - 1] else 1),
                        ),
                    )
                    lastdiag = olddiag

            out[i] = column[s1len]

%timeit levenshtein = levenshtein_numba(df["question1"].values.data, df["question2"].values.data)
# 27.3 s ± 94.5 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

Surprisingly, this is a bit slower than the previous version.
As the loops are a bit more complex, it could be that less optimisation could be applied to make them faster.
Still, one would expect that removing an allocation should make a difference.
But in this case, we are talking about a rather small allocation for a typical `numpy` usage.
For these occasions, `numpy` keeps [a cache of previous allocations that can be (and this case are) reused](https://github.com/numpy/numpy/blob/b01a4732cedf8ffe6def3a877e22dacacc796bd5/numpy/core/src/multiarray/alloc.c#L79-L111).

While this is the one optimisation that we can do on the algorithm, the other thing we can improve in the implementation is the use of multiple CPUs.
In the case of my MacBook, I have four physical cores (8 when you count the hyperthreaded ones) and currently only a single one is utilised.
As the Arrow structures are one block of memory and only a single Python object, working on several rows at a time is not bound by Python's GIL.
This we can also change the implementation of our for-loop functions to use `numba`'s parallelisation using `prange`:

```python
@numba.jit(nogil=True, nopython=True, parallel=True)
def levenshtein_numba_no_null(
    length, offsets_buffer_a, data_buffer_a, offsets_buffer_b, data_buffer_b, out
):
    """Compute the levenshtein distance for a whole column without nulls."""
    for i in prange(length):
        out[i] = levenshtein_numba_row(
            data_buffer_a[offsets_buffer_a[i] :],
            offsets_buffer_a[i + 1] - offsets_buffer_a[i],
            data_buffer_b[offsets_buffer_b[i] :],
            offsets_buffer_b[i + 1] - offsets_buffer_b[i],
        )

…

%timeit levenshtein = levenshtein_numba(df["question1"].values.data, df["question2"].values.data)
# 6.36 s ± 271 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

This gives us a really nice speedup and brings us not fully but in the near range of a 10x speedup to the already well-tuned `jellyfish` implementation.

## Making it less ugly / verbose

Performance-wise, this has been a good showcase of the performance you could expect with an Arrow-backed string column.
Sadly from the amount of code, this has been very verbose.
Most of the code is though for dispatching on the array and not the actual algorithm that works on a per-row basis.
Thus when someone would implement another algorithm on top of an Arrow-backed string column, we would expect them to implement the exact same boilerplate.
Instead of doing that each time, we can also just provide a convenience layer for that in `fletcher` as `numba` is capable of jitting functions that come in as an argument of another function.

Exactly this has been done in [fletcher#208](https://github.com/xhochy/fletcher/pull/208) that is available as part of `fletcher>=0.7`.
Here, we have a helper called `fletcher.algorithms.string.apply_binary_str` that takes two Arrow-backed columns, a `numba`-jitted callable, and an output dtype.
You as an algorithm implementer only need to code the `numba`-jitted callable that takes two strings and their length as an input and returns a fixed-size result (int, bool, float).
The helper `apply_binary_str` takes care of (un)packing the Arrow structures and running the correct for-loops with the correct handling of missing values.
In addition to the Levenshtein implementation in this post, it also deals with `pyarrow.ChunkedArray` instances and dispatches the execution of the array-wise for-loops on batches of chunks.

Furthermore, there is a fifth (optional) argument `parallel=` where you can select whether the algorithm execution should be parallelised or not.
With all this convenience in place, the final Levenshtein call then boils down to only the row-wise function `levenshtein_numba_row` and the call to `apply_binary_str`.

```python
apply_binary_str(
    df["question1"].values.data,
    df["question2"].values.data,
    func=levenshtein_numba_row,
    output_dtype=np.int64,
    parallel=True,
)
```

## Conclusion

With a bit of research, it already turns out that we could compute the Levenshtein distance in roughly 50s on a single core or with parallelisation in 11s on multiple cores of a 13" Intel-based MacBook Pro.
By using `fletcher` and `numba`, we could half the execution time by using a simple Levenshtein implementation in pure `numba`-jitted Python, either on a single or multiple cores.
Additionally, the Arrow-backed columns only need half the RAM of the Python-`object` based columns used currently by default in `pandas`.
And while the current implementation here consisted of quite a significant amount of code, we added a helper function to `fletcher 0.7` that ensures that we can implement such algorithms that take two strings and return a fixed size scalar as a result by only defining a `numba`-jitted function that works on a single row.

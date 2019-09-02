---
layout: post
title: Writing a boolean array for pandas that can deal with missing values
feature_image: "https://picsum.photos/id/110/2560/600"
---

When working with missing data in `pandas`, one often runs into issues as the main way is to convert data into `float` columns.
`pandas` provides efficient/native support for boolean columns through the `numpy.dtype('bool')`.
Sadly, this `dtype` only supports `True/False` as possible values and no possibility for storing missing values.
Additionally, `numpy` uses a whole byte to store the `True/False` information while a single bit would be sufficient.

## The current way in pandas

By intuition, one would simply create a boolean array with missings by calling either `pandas.Series([True, None, False])` or `pandas.Series([True, numpy.nan, False])`.
This sadly creates a `Series` with an `object` dtype.
While already having an `object` dtype means that all operations are run in native Python and not accelerated by `numpy`, the behaviour of reduction functions like `any()` is feeling a bit weird at first.
The actual algorithm implements `Series([x, y, z]).any()` as `x or y or z` which seems reasonable but doesn't match [the docstring of `pandas.Series.any` dealing with missing data](https://pandas.pydata.org/pandas-docs/version/0.25/reference/api/pandas.Series.any.html#pandas.Series.any).
The result of `any()` should always be a boolean, irrespectively if the data contains missing values:

> skipna : bool, default True <br />
> Exclude NA/null values. If the entire row/column is NA and skipna is True, then the result will be False, as for an empty row/column. If skipna is False, then NA are treated as True, because these are not equal to zero.

In the case of `pd.Series([False, np.nan]).any(skipna=False)`, we get the surprising result of `False or np.nan = np.nan`.
The behaviour of `any()` using `None` as a missing value representation it even less intuitive as the operation is not commutative.
I've opened [a discussion about this behaviour](https://github.com/pandas-dev/pandas/issues/27709) upstream and it may be changed soon to match the docstring but this still won't get us a fast solution.

As previously mentioned, for dealing with missing values in `pandas`, you normally have to resort to `float` columns.
The same is true in the case of booleans with missing values.
Using `pd.Series([False, np.nan]).astype(float).any(skipna=False)`, we get `True` as the result.
This is what `pandas` docstring also describes as the correct behaviour of `any()` on arrays with missing values.
While we now have a working solution, we ended up using the wrong data type (`float` instead of `bool`) and we waste even more space for storing a tri-state (true/false/missing).

## Making it more efficient

Some time ago, I have started a prototype(!) implementation of a `pandas.ExternsionArray` implementation backed by [Apache Arrow](https://arrow.apache.org/).
This project is called [fletcher](https://github.com/xhochy/fletcher) and is available via [PyPI](https://pypi.org/project/fletcher/) and [conda-forge](https://github.com/conda-forge/fletcher-feedstock).
Apache Arrow uses two memory buffers to represent a column for booleans with missing values.
It uses one bitmask for storing whether a row is missing or has a value and a second bitmap whether the value is true or false.
This then uses 2 bit instead of 8 bit (`numpy.dtype('bool')`) or even 16-64bit in the case of a float column.

While we don't present a boolean array that is on-par with the functionality you would get with a float-typed column, we want to have a minimally working, efficient boolean column.
`fletcher` in its current form already supports the creation of a column and the proxing of basic pandas functionality that is independent of the data type.
It doesn't support type-specific functionality though.
Thus we need to implement the `any()` and `all()` reduction on this column.
As Apache Arrow does not yet support these operations as a built-in, we need to implement them on our own.

### Testing the implementation

Following good software engineering practices, we are going to implement this using test-driven development.
As we already have a working implementation and would like to implement an alternative one using a more efficient storage, we can use the existing one as a reference for checking the correctness of our new implementation.
This is a perfect case for using [property-based testing](https://hypothesis.works/articles/what-is-property-based-testing/) as we are able to retrieve the correct result for a given input data.
As a helpful tool to implement property-based testing on Python, we are using [`hypothesis`](https://hypothesis.readthedocs.io).

For writing such a unit test, we first need to take care of the creation of random input data.
`pandas.Series.any` takes two relevant arguments in our case, the data itself and a flag `skipna` whether to skip missing values or consider them as part of the algorithm.
The generation of the `skipna` parameter is straight-forward though the `st.booleans()` strategy which either emits `True` or `False`.
For our data parameter, there is no predefined strategy that would generate the expected data.
Thus we need to construct a combined strategy out of `hypothesis` existing strategies.
Using `st.one_of(st.booleans(), st.none())` we can create a strategy that either returns `True` or `False` or `None` as a missing value.
By wrapping this into `st.lists(…)`, we generate a list of these values instead of a single scalar.

```python
import hypothesis.strategies as st
from hypothesis import example, given

@given(data=st.lists(st.one_of(st.booleans(), st.none())), skipna=st.booleans())
```

`hypothesis` is really good at generating edge cases for the given input types but we also want to be explicit in testing them.
Thus we can use its `example` decorator to enforce that the empty list is always passed in as test data.

```python
@example([], False)
@example([], True)
```

Now that the random input generation is solved, we need to take care to run the code and test its properties.
Testing the properties is simple as we can test all properties of the result by comparing it against a reference result.
As mentioned in the first section, for dealing with boolean arrays with missing values in pandas, we need a `float`-typed `Series`.

```python
import pandas as pd
import pyarrow as pa

arrow = pa.array(data, type=pa.bool_())
pandas = pd.Series(data).astype(float)

assert any_op(arrow, skipna) == pandas.any(skipna=skipna)
```

One speciality of the `fletcher` implementation is that it doesn't only work on continuous arrays but also on chunked arrays.
As such, we will have code in the later implementation that reduces the result of `any(…)` on separate memory regions together.
This is an edge case our above defined data generation doesn't cover.
Thus we take the random data and split it in half to get a chunked array and test this also leads to the very same result.

```python
if len(data) > 2:
    arrow = pa.chunked_array(
        [data[: len(data) // 2], data[len(data) // 2 :]], type=pa.bool_()
    )
    assert any_op(arrow, skipna) == pandas.any(skipna=skipna)
```

Putting all pieces together, we get the following unit test:

```python
@given(data=st.lists(st.one_of(st.booleans(), st.none())), skipna=st.booleans())
@example([], False)
@example([], True)
def test_any_op(data, skipna):
    arrow = pa.array(data, type=pa.bool_())
    pandas = pd.Series(data).astype(float)

    assert any_op(arrow, skipna) == pandas.any(skipna=skipna)

    # Split in the middle and check whether this still works
    if len(data) > 2:
        arrow = pa.chunked_array(
            [data[: len(data) // 2], data[len(data) // 2 :]], type=pa.bool_()
        )
        assert any_op(arrow, skipna) == pandas.any(skipna=skipna)
```

### Implementing `any(…)/all(…)` reductions

With the unit test in place, we can now start implementing the functions.
The first major difference to NumPy booleans is that we don't have a bytemask but a bitmask.
Thus means that every byte in memory contains eight values.
As we will be working value-by-value we first need to write some code that given a position selects the correct memory address and from the byte there the correct bit.
As Arrow columns store the value and the missing flag in two separate memory buffers, we can use the same index but different base address to select these values.

```python
byte_offset = i // 8
bit_offset = i % 8
mask = np.uint8(1 << bit_offset)
valid = valid_bits[byte_offset] & mask
value = data[byte_offset] & mask
```

The all implementation is quite straight-forward as adhering to the `pandas` definition of `skipna` means that the value of it is actually irrelevant.
Thus we come to the following solution.
The only important aspect here is that we give `numba` the hint that `valid` and `value` are boolean variables.

```python
@numba.njit(locals={"valid": numba.bool_, "value": numba.bool_})
def _all_op(length, valid_bits, data):
    # This may be specific to Pandas but we return True as long as there is not False in the data.
    for i in range(length):
        …fetch valid and value…
        if valid and not value:
            return False
    return True
```

For the end-user facing operation, we provide a function that takes a `pyarrow.Array` instead of the individual memory buffers.
Unpacking those can be done by calling `arr.buffers()` where in the case of a `BooleanArray` the first is the valid bitmap (this is true for all Arrow arrays) and the second the memory buffer holding the actual values.
Additionally, we don't only support passing in `pyarrow.Array` objects but also `pyarrow.ChunkedArray` objects that are made up of a list of `pyarrow.Array` objects. Here we need to call the `all_op` on a per-array basis and merge the results at the end again.

```python
def all_op(arr, skipna):
    if isinstance(arr, pa.ChunkedArray):
        return all(all_op(chunk, skipna) for chunk in arr.chunks)

    # skipna is not relevant in the Pandas behaviour
    return _all_op(len(arr), *arr.buffers())
```

The `any` operation is slightly more complex as we here also need to take care of the differing behaviour depending on the setting of `skipna`.
Thus the user-facing function dispatches to different jitted-functions depending on whether we want to take missing values into account.


```python
def any_op(arr, skipna):
    if isinstance(arr, pa.ChunkedArray):
        return any(any_op(chunk, skipna) for chunk in arr.chunks)

    if skipna:
        return _any_op_skipna(len(arr), *arr.buffers())
    return _any_op(len(arr), *arr.buffers())
```

In the case of `skipna=True`, the we return `True` if one element satisfies the condition `valid and value`.
For `skipna=False` we interpret missing values as `True` (as defined in `pandas` documentation) and thus the `any_op` is `True` whether there is an entry for which `(valid and value) or (not valid)` holds.

#### A last, small special case

There is sadly one more thing to handle which is that Arrow allows to omit the valid bitmap in the case when there are no nulls (i.e. `null_count == 0`).
This helps to save a bit of memory in the quite-often case of no missing data (in our case this amounts to a saving of half of the used memory) but sadly introduces another branch in our code.
For the all operation, we can use the same code as in the case where we have a valid bitmap but instead of accessing it, we simply assume that every entry is valid.
Therefore the non-null case of the all operation can be implemented using:

```python
@numba.njit(locals={"value": numba.bool_})
def _all_op_nonnull(length, data):
    for i in range(length):
        byte_offset = i // 8
        bit_offset = i % 8
        mask = np.uint8(1 << bit_offset)
        value = data[byte_offset] & mask
        if not value:
            return False
    return True
```

### Performance

While we already can fallback in `pandas` to using float{32,64} columns to correctly work with boolean columns with missings, one thing we also want to achieve is a better performance.
As boolean operations are quite simple, the best impact in performance is the reduced memory usage we get by using Arrow.

To compare the performance, we have written some simple benchmarks using [airspeed velocity](https://asv.readthedocs.io/en/stable/).
In the case of _all_, we have the taken the most complex version where all entries are `True` and thus the whole array must be scanned to compute a valid result.
We do the same most-expensive version for _any_ but here we set all entries to `False`.
To also test the performance difference with having missing data, we added another pair of benchmarks where we set the last entry to None/NAN/missing.
For NumPy this makes no difference but in the Arrow case, the valid bitmap is actually allocated and must then be checked for every entry.


Running the benchmarks on my MacBook with Python 3.7 gives the following results:

```
% asv run --python=same -b 'boolean*'
· Discovering benchmarks
· Running 12 total benchmarks (1 commits * 1 environments * 12 benchmarks)
[  0.00%] ·· Benchmarking existing-py_Users_uwe_miniconda3_envs_fletcher_bin_python
[  4.17%] ··· Running (boolean.BooleanAll.time_fletcher--)......
[ 29.17%] ··· Running (boolean.BooleanAny.time_numpy--)..
[ 54.17%] ··· boolean.BooleanAll.time_fletcher               16.2±0.6ms
[ 58.33%] ··· boolean.BooleanAll.time_fletcher_withna        16.3±0.4ms
[ 62.50%] ··· boolean.BooleanAll.time_numpy                    35.6±1ms
[ 66.67%] ··· boolean.BooleanAll.time_numpy_withna             37.8±2ms
[ 70.83%] ··· boolean.BooleanAny.time_fletcher               15.9±0.1ms
[ 75.00%] ··· boolean.BooleanAny.time_fletcher_withna        16.5±0.8ms
[ 79.17%] ··· boolean.BooleanAny.time_numpy                  35.9±0.6ms
[ 83.33%] ··· boolean.BooleanAny.time_numpy_withna           35.2±0.2ms
[ 87.50%] ··· boolean.BooleanAny.track_size_fletcher            2097152
[ 91.67%] ··· boolean.BooleanAny.track_size_fletcher_withna     4194304
[ 95.83%] ··· boolean.BooleanAny.track_size_numpy              67108864
[100.00%] ··· boolean.BooleanAny.track_size_numpy_withna       67108864
```

From the results, we can see that runtime-wise, we are at least **2.1x** times faster than the NumPy implementation in any case and can achieve an increase of upto 2.32x in speed in the most extreme case.
More remarkable though is the improvement in memory usage.
We have been fair to NumPy/pandas and already used `float32` instead of the typically allocated `float64` type.
Still, for the case with no missing data, we can save a factor of **32** on RAM whereas with missings, we still have a factor of **16** while getting correct results.
The `np.bool` dtype would also have saved us a factor of 4 but as mentioned in the beginning, the results with missing data where inconsistent and thus sometimes incorrect.

### Conclusion

In this blog post, we have shown on how a simple operation on a `pyarrow.Array` or `pyarrow.ChunkedArray` can be implemented using `numba` and `hypothesis` for property based testing.
This was especially helpful as there is already an existing but memory-wasting implementation available.
We could use that implementation to verify that our new implementation has the same behaviour with a different underlying storage.

With the Arrow-based implementation we now have a custom boolean type that uses only 2 bits of storage per row instead of the existing implementation that need between 8 and 64 bit and at the same type achieve a speedup of **2** in the runtime.

The work described here has already been merged into `fletcher` [1](https://github.com/xhochy/fletcher/pull/77), [2](https://github.com/xhochy/fletcher/pull/78), [3](https://github.com/xhochy/fletcher/pull/80) and released as `0.2.0` on [PyPI](https://pypi.org/project/fletcher/0.2.0/) and [conda-forge]().

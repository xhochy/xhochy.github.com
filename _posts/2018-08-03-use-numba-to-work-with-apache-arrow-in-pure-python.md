---
layout: post
title: Use Numba to work with Apache Arrow in pure Python
---

[Apache Arrow](https://arrow.apache.org/) is an in-memory memory format for
columnar data. In more "plain" English, it is a standard on how to store
DataFrames/tables in memory, independent of the programming language. One of
its most prominent uses is for the `@pandas_udf` decorator in [Apache
Spark](https://databricks.com/blog/2017/10/30/introducing-vectorized-udfs-for-pyspark.html)
to move data quickly between Scala and Python/pandas.

Furthermore, it also brought the [reading of Apache Parquet files to the
Python world](https://arrow.apache.org/docs/python/parquet.html). While yet
known for benefiting I/O, there is not yet much analytic implemented on top of
it. As the focus of Arrow is on performance, the core of the Python package is
based on the C++ implementation. At the moment there are already some analytic
kernels implemented. As the implementation of these functions yet only happened
on the C++ level and many Python developers only want to write Python code,
they were unable to extend Arrow with more functionality.

Fast for-loops on numerical data
--------------------------------

One of the main things you learn when you start with scientific computing in
Python is that you should not write for-loops over your data. Instead you are
advised to use the vectorized functions provided by packages like `numpy`. The
major share of computations can be represented as a combination of fast NumPy
operations. But in the end, there are still some that cannot be expressed
efficiently with NumPy. In these cases ones has to resort to slow Python
for-loops.

As an alternative to these for-loops, people often use
[Cython](http://cython.org/) to write compiled code that provides similar
performance to NumPy operations. This provides a good combination of
a Python-like language with the performance benefits of code written in C/C++.
One of the disadvantages of C/C++ and Cython is that you need to compile your
code ahead-of-time which is in stark contrast to the typical just-in-time
interpreted Python code.

Numba for just-in-time compiled, efficient Python code
------------------------------------------------------

The between fast, compiled scientific code and the simple, interpreted nature
of Python is closed by [Numba](http://numba.pydata.org/). Numba is a
just-in-time compiler based on the [LLVM compiler
infrastructure](https://llvm.org/) that inspects math-heavy Python code at
runtime. It will then produce fast, vectorized native machine instructions
that often even beat the performance of NumPy operations slightly.

In contrast to [PyPy](https://pypy.org/), Numba is not a generic Python
just-in-time compiler but is focused on accelerating code that works on
non-Python memory regions like the contents of NumPy arrays. You can apply it
to a function by using the `@numba.jit` decorator. A typical example where
Numba can greatly improve the performance of your code is when you write one of
those for-loops over numerical data. Although you were always told, that they
will be horribly slow in comparison to using NumPy operations, sometimes, they
are unavoidable.

To highlight the benefits of Numba, we take a small example of an for-loop over
a NumPy array. We look at two variants: a simple for-loop and the same code
just-in-time compiled with Numba. You possibly will also find an implementation
in NumPy without Numba for this code but we just use it here for demonstration
purposes.

{% highlight python %}
import numpy as np

arr = np.arange(1000000)
result = np.zeros_like(arr)

def py_adapt(arr, result):
    for i in range(len(arr) - 1):
        result[i] = np.sqrt(arr[i] * arr[i + 1])
{% endhighlight %}

Profiling the pure-Python approach:

{% highlight python %}
%timeit py_adapt(arr, result)
# 3.35 s ± 20.3 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
{% endhighlight %}

Implementing the same but using a Numba decorator this time.

{% highlight python %}
from numba import jit

@jit
def nb_adapt(arr, result):
    for i in range(len(arr) - 1):
        result[i] = np.sqrt(arr[i] * arr[i + 1])
{% endhighlight %}

{% highlight python %}
%timeit nb_adapt(arr, result)
# 5.83 ms ± 91.6 µs per loop (mean ± std. dev. of 7 runs, 1 loop each)
{% endhighlight %}

This small example shows the power of Numba: A simple Python loop can be turned
into efficient numerical code only by the addition of a decorator.

Applying the magic of Numba to Apache Arrow
-------------------------------------------

Numba has built-in support for NumPy arrays and Python's `memoryview` objects.
As Arrow arrays are made up of more than a single memory buffer, they don't
work out of the box with Numba. To integrate them with Numba, we need to
understand how Arrow arrays are structured internally. As they are all
nullable, each array has a valid bitmap where a bit per row indicates whether
we have a null or a valid entry. Depending of the type of the array, we have
one or more memory buffers to store the data.

For the set of primitive types (`int`, `float`, `bool`), there is simply
another buffer that contains the data, just like in a NumPy array. In the
case of strings, we have two additional buffers. There is one buffer that
contains the actual characters of all strings in the array, one string after
the next ones. To find the start and end points of the strings, we also have an
offsets buffer that contains the start index of each string in the characters
array. To reconstruct the string at position `i`, we take its starting point
`s_i` and the one of the next string `s_i+1`. The string then is represented by
the characters in `values[s_i:s_i+1]`.

As NumPy has no native variable length string type, we're going to use this as
an example. We want to build a fast function that returns us the lengths of all
strings in an Arrow StringArray. To represent composite memory structures and
provide operations on them, Numba provides the [`@jitclass`
decorator](https://numba.pydata.org/numba-doc/dev/user/jitclass.html). The
decorator is applied to a standard Python class. For each member of the class,
we need to specify its native type like `numba.int32` for a scalar int or
`numba.float32[:]` for a float array. When the class is used in a
`@jit`-decorated function, objects of it can be used just like an ordinary
Python class but the generated machine code is highly optimised.

Given the above explanation, we can build a `NumbaStringArray` class that
provides us a convenience interface to the underlying Arrow buffers.
In addition, we add a convenience function `NumbaStringArray.make` that
dissects an existing `pyarrow.Array` instance into the buffer and instantiates
the new class.

{% highlight python %}

import numba
import types


@numba.jitclass(
    [
        ("missing", numba.uint8[:]),
        ("offsets", numba.uint32[:]),
        ("data", numba.optional(numba.uint8[:])),
        ("offset", numba.int64),
    ]
)
class NumbaStringArray(object):
    """Wrapper around arrow's StringArray for use in numba functions.

    Usage::

        NumbaStringArray.make(array)
    """

    def __init__(self, missing, offsets, data, offset):
        self.missing = missing
        self.offsets = offsets
        self.data = data
        self.offset = offset

    @property
    def size(self):
        return len(self.offsets) - 1 - self.offset

    def isnull(self, str_idx):
        str_idx += self.offset
        byte_idx = str_idx // 8
        bit_mask = 1 << (str_idx % 8)
        return (self.missing[byte_idx] & bit_mask) == 0

    def byte_length(self, str_idx):
        str_idx += self.offset
        return self.offsets[str_idx + 1] - self.offsets[str_idx]


def _make(cls, sa):
    if not isinstance(sa, pa.StringArray):
        sa = pa.array(sa, pa.string())

    buffers = sa.buffers()
    return cls(
        np.asarray(buffers[0]).view(np.uint8),
        np.asarray(buffers[1]).view(np.uint32),
        np.asarray(buffers[2]).view(np.uint8),
        offset=sa.offset
    )


# @classmethod does not seem to be supported
NumbaStringArray.make = types.MethodType(_make, NumbaStringArray)
{% endhighlight %}

To compare the performance with NumPy, we create arrays with a million strings:

{% highlight python %}
import pyarrow as pa
import numpy as np

arrow_array = pa.array([str(i) for i in range(1000000)])
numpy_array = arrow_array.to_pandas()
{% endhighlight %}

We build then two nearly equal functions that compute the individual string
lengths, one with a NumPy array, the other one on an Arrow array using Numba.
One noticeable difference here is that we can turn on the `nopython` and
`nogil` modes for the Arrow version as it does not have to deal with Python
objects. In contrast, as NumPy has no native variable-length string type, we
have resort to Python object and thus these modes are not possible.

{% highlight python %}
@numba.jit
def numpy_lengths(array):
    result = np.empty_like(array)
    
    for i in range(len(array)):
        result[i] = len(array[i])
        
    return result
        

@numba.jit(nopython=True, nogil=True)
def arrow_lengths(array):
    result = np.empty(array.size)

    for t in range(array.size):
        if array.isnull(t):
            result[t] = 0
        else:
            result[t] = array.byte_length(t)
    
    return result
{% endhighlight %}

{% highlight python %}
%timeit numpy_lengths(numpy_array)
# 363 ms ± 7.88 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
{% endhighlight %}

{% highlight python %}
# Takes 11us, so no real performance impact
numba_array = NumbaStringArray.make(arrow_array)
%timeit arrow_lengths(numba_array)
# 42 ms ± 1.09 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
{% endhighlight %}

This leads us to a near 10x performance increase while still staying in pure
Python. One could probably get better performance by switching to C++ and being
more careful about alignment, vectorisation, … but for a simple first pass,
this is an extremely grateful result.

Use Case
--------

While this is a nice example on how to combine Numba and Apache Arrow, this is
actual code that was taken from [Fletcher](https://github.com/xhochy/fletcher).
There we are in the process of building a pure-Python library that combines
Apache Arrow and Numba to extend `pandas` with the data types are available in
Arrow. While you need some C++ knowledge in the main Arrow project, you can get
started building fast columnar code in pure Python there.

---
layout: post
title: "Fletcher 0.3: A status report on the mission to get pandas hooked on Apache Arrow"
feature_image: "/images/marija-zaric-k-yf-LTqOuA-unsplash.jpg"
---

It has been now nearly two years since the idea came up to use `pandas`' new `ExtensionArray` interface to provide columns in `pandas` that are backed by Apache Arrow.
[`fletcher`](https://github.com/xhochy/fletcher) was started as a prototype project to show how this idea can be brought together.
Since then there has been quite a lot of development in both `pandas` and Apache Arrow.
Still, `fletcher` remains a prototype to show how this could look like as essential functionality is missing to use it productively.
With the two year mark now approaching, I thought it was a good time to give a progress report and tag new intermediate release.

Although I highly warn of any productive use of `fletcher`, I would be curious to find out where the first entry point is that fails for users.
Thus, please try to use it in your project and report the first exception you encounter in [our issue tracker](https://github.com/xhochy/fletcher/issues).
This will the focus of `fletcher` development as it will give us the insight of what the required functionality is to provide a minimal useful library.

Choosing between chunked & continuous arrays as storage backend
---------------------------------------------------------------

Initially `ExtensionArray` instances in `fletcher` were solely backed by `pyarrow.ChunkedArray` instances.
This was chosen as chunked arrays allow for the most flexibility, e.g. concatenating them can be done in constant time.
But with the flexibility on the user side also comes a lot of complexity in implementing algorithms on top of them.
Due to the nature of the chunking, you don't deal with a simple linear index to access any element of an array but you always need to translate between the scalar index that indicates the position in the whole array and the tuple `(chunked_index, index_in_chunk)` that gives you the relative position of an element to its containing chunk.

Thus, we now provide two different extension array implementations.
There now is the more simpler `FletcherContinuousArray` which is backed by a `pyarrow.Array` instance and thus is always a continuous memory segments.
The initial `FlectherArray` which is backed by a `pyarrow.ChunkedArray` is now renamed to `FletcherChunkedArray`.
While `pyarrow.ChunkedArray` allows for more flexibility on how the data is stored, the implementation of algorithms is more complex for it.
As this hinders contributions and also the adoption in downstream libraries, we now provide both implementations with an equal level of support.
We don't provide the more general named class `FlectherArray` anymore as there is not a clear opinion on whether this should point to `FletcherContinuousArray` or `FletcherChunkedArray`.
As usage increases, we might provide such an alias class in future again.

Arithmetic, comparison and reduce operations
--------------------------------------------

For numeric data, `pandas` has added in the last year a test suite that provides a vast amount of tests to check all kind of numeric operations on `ExtensionArray`.
With the help of this suite, we were able to implement these operations on top of `pyarrow.float*` and `pyarrow.int*` types.
The current implementation applies the mask on the input arrays and then delegates to numpy for the computations.
With this, we are on the same performance level as `pandas.IntegerArray`.
In future, we want to use numeric operation that are directly implemented in Apache Arrow C++ and make direct use of the validity bitmap instead.
This will save on memory bandwidth / storage as well will be faster on the actual numeric operations as bitmap checking and operation calculation won't be separated steps anymore.

BooleanArray & StringArray in pandas and its fletcher counterparts
------------------------------------------------------------------

In the newest release, we have an implementation of a boolean array that supports missings and behaves like a `pandas.Series` of float type for `any` and `all`.
There was a [blog post outlining its implementation](https://uwekorn.com/2019/09/02/boolean-array-with-missings.html).
In pandas 1.0, a new `BooleanArray` was released with a slightly different behaviour, we will adapt `fletcher` to this in the next release.
Currently our tests are failing and it looks like an inconsistency in pandas' implementation which we currently investigate.

Besides `BooleanArray`, pandas 1.0 also added `StringArray` which brings in a check that all objects in that column are strings but doesn't improve on performance.
Thus there is still the need for a fast string type like we are implementing in fletcher.
A first step in this direction we now support `.str.cat` as an algorithm on fletcher string columns via `.fr_text.cat`.

Missing kernels / operations / .. in Arrow or fletcher
------------------------------------------------------

One of the main things making `fletcher` not practically usable at the moment are the missing algorithm implementations on top of it.
You can select / slice / store `fletcher` columns but executing operations like `zfill` for strings or `dt.year` on top of its columns is not possible yet.
These operations currently need a cast to an object-typed series making them even slower than their current pandas counterparts.

Such operations are named kernels in the Apache Arrow C++ source code where a rudimentary set of them exists already.
Sadly a lot of common functionality is missing for the basic data types and some of the existing kernels are only implemented for `pyarrow.Array` and not for `ChunkedArray`.
Having good kernels available for `ChunkedArray` in Arrow C++ itself is crucial as applying the kernel to individual chunks often includes non-trivial transformations of intermediate results or indices that were given as an input.

With the pandas integration basics now in place in `fletcher`, we will be able to concentrate on exactly these kernels.
As one of the points of `fletcher` is to explore on how to implement kernels on top of Arrow in the most efficient way with `numba`, we are first trying to implement a kernel in `fletcher` and will only resort to Arrow if the implementation turns out to be too complex or too slow with `numba`.
One of the drawbacks of putting a kernel implementation into Arrow C++ is that we need to wait for a release of it to make it available to end-users.
With implementing them first in `fletcher`, we can make releases on our own and thus release them faster to the user.
Afterwards, when Arrow is then released, we can remove our implementation and point to the most likely (a bit) faster implementation in C++.

spatialpandas as an impact example
----------------------------------

The main goal of fletcher is to make impact on `pandas` and Apache Arrow but we are also very pleased that we have an impact on the ecosystem.
The influence of fletcher can be seen a bit in [`spatialpandas`](https://github.com/holoviz/spatialpandas), an `ExtensionArray` implementation for spatial/geometric operations.
`spatialpandas` is also building on top of `pyarrow` and `numba` to implement certain specific spatial-specific data types and also reuses basic code from `fletcher` for accessing Arrow data.

Next steps
----------

As the next steps in `fletcher`, we will focus to implement more of the `.str` and `.dt` methods.
These are the simple datatypes `fletcher` can have massive improvements over the status quo in `pandas`.
This is because strings are currently implemented as `object` dtype even when using the new `StringDtype` and thus are not comparable in performance to the dtypes that are implement with numpy-native types.
For `date(time)` columns, we can also improve a bit by allowing more-than-nanosecond precision and also the use of 32bit datatype for dates where 64bit aren't needed to represent the most commonly used timespans.

It would also be nice to have more operations on nested types as they are currently unavailable in `pandas` and `fletcher` supports them through the use of Arrow.
But as the kernel implementations for them are much more complex, we are going to focus first on strings and dates.

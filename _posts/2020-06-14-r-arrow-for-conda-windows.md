---
layout: post
title: "Building R Arrow on Windows: A tale of two compilers"
feature_image: "/images/miguel-oros-LmsppeMVX2Y-unsplash.jpg"
---

Windows support for Apache Arrow is pretty good.
There are Python wheels, Python conda packages and a binary build for R on CRAN.
One thing that has been missing though for a long time has been a conda package for R Arrow on Windows.
Thanks to a lot of experimentation and some important suggestions by [Isuru Fernando](https://github.com/isuruf) (Thanks!), we are able to use `conda install -c conda-forge r-arrow` successfully on Windows now.

While it may sound like an easy task if there are already packages in CRAN and Python, there is one important caveat which made the whole process complicated.
Python PyPI and conda-forge packages target Microsoft Visual C/C++ (MSVC) as their main compiler whereas R uses on Windows the Rtools toolchain which is based of MSYS2 and the MinGW compilers.
To support R on Windows in conda-forge, the `r-*` packages are also all built using MinGW as most of the packages won't compile at all with MSVC.
But with the goal of being a good citizen in the conda-forge ecosystem, we wanted to built `r-arrow` against the same `arrow.dll` from the `arrow-cpp` package as the Python package `pyarrow` links to.
This meant that we would have to link an MSVC DLL with a MinGW DLL.
Under certain conditions this works but as the two compilers have a different C++ ABI, this doesn't work for C++ symbols.

In the following, I'm describing my more or less adventurous path to get the `r-arrow` package building.
The final solution is one that I have not seen yet being used by anyone else.
This means that it might also be a solution that could get other packages building on Windows and linking to the same DLLs as their Python counter-parts do.
On the other side, it might also mean that the solution is yet quite unstable and will more likely lead to problems.
This is something time and the issue tracker will tell us.
Thus if you're building an R package for Windows and don't have any problems yet, stick to the more stable and mature Rtools40 toolchain.

### 1: Activate windows on the conda-force CI.

As a first attempt, we wanted to see where things fail if we take the standard R package approach on conda-forge.
Thus we removed the `skip: true  # [win]` line from the recipe and called `conda smithy rerender` and pushed to see what CI said.

Even though we listed `arrow-cpp` as a dependency, the existing package scripts failed to detect it.
Neither the auto-detection code in `configure.win` was able to find the package:

```
Arrow C++ library was not found
Usage: grep [OPTION]... PATTERN [FILE]...
Try 'grep --help' for more information.
```

Nor did the compiler find it on the default include path (MinGW headers and MSVC headers are in different locations in conda installations, so they don't accidentally mix up):

```
g++  -std=gnu++11 -I"D:/bld/r-arrow_1586370250701/_h_env/lib/R/include" -DNDEBUG -I../windows//include -DARROW_STATIC -DPARQUET_STATIC -DARROW_DS_STATIC -DARROW_R_WITH_ARROW -I"D:/bld/r-arrow_1586370250701/_h_env/Lib/R/library/Rcpp/include"   -I"D:/bld/r-arrow_1586370250701/_h_env/lib/R/../../Library/mingw-w64/include"     -O2 -Wall -gdwarf-2 -march=x86-64 -mtune=generic -c array.cpp -o array.o
bash.exe: warning: could not find /tmp, please create!
In file included from array.cpp:18:
./arrow_types.h:198:10: fatal error: arrow/api.h: No such file or directory
 #include <arrow/api.h>
          ^~~~~~~~~~~~~
compilation terminated.
```

### 2: Only build on PR for windows, save CI resources

While we were lucky to also have a local development environment, to check whether things really worked reproducible, we already planned to push quite a lot to CI.
Instead of always running the whole build matrix, we adjusted the recipe so that `r-arrow` on our development branch would only be built for Windows.
We knew it still succeeded on the other systems as all changes we were doing we either in the build scripts that are only executed on Windows or the lines changed in the `meta.yaml` were suffixed by a `# [win]` selector.

```
diff --git a/recipe/meta.yaml b/recipe/meta.yaml
index dcf3617..d2a14bd 100644
--- a/recipe/meta.yaml
+++ b/recipe/meta.yaml
@@ -12,6 +12,7 @@ source:

 build:
   merge_build_host: True  # [win]
+  skip: True  # [not win]
   number: 1
   rpaths:
     - lib/R/lib/
```

To apply this change to the build process, we called `conda smithy rerender` to update the auto-generated CI scripts.


### 3: Adjust configure.win to find the correct paths

The R package of Arrow comes with a long `configure.win` script that tries to detect an existing Arrow C++ installation to link against or otherwise download a pre-built binary.
While we didn't want the latter, the former is also more advanced than needed (and failing in our case).
As we are aware of the exact paths where Arrow C++ resides, we supplied our own (fully hard-coded) `configure.win` script instead.
In addition to a normal R build, this also references the directory where the non-MinGW headers reside and also explicitly tells the linker where to find the conda-built Arrow C++ libraries.

```bash
#!/bin/bash

set -euxo pipefail

echo "PKG_CPPFLAGS=-DNDEBUG -I\"${LIBRARY_PREFIX}/include\" -I\"${PREFIX}/include\" -O2 -D_CRT_SECURE_NO_WARNINGS -DARROW_R_WITH_ARROW" > src/Makevars.win
echo 'PKG_CXXFLAGS=$(CXX_VISIBILITY)' >> src/Makevars.win
echo 'CXX_STD=CXX11' >> src/Makevars.win
echo "PKG_LIBS=-L\"${LIBRARY_PREFIX}/lib\" -larrow_dataset -lparquet -larrow" >> src/Makevars.win
```

In our conda `bld.bat` build script, we manually copy this file over the existing `configure.win` in the package before we call `R CMD install --build r`.

```
diff --git a/recipe/bld.bat b/recipe/bld.bat
index 470dd8b..9e26180 100644
--- a/recipe/bld.bat
+++ b/recipe/bld.bat
@@ -1,2 +1,4 @@
+cp %RECIPE_DIR%/configure.win r
+IF %ERRORLEVEL% NEQ 0 exit 1
 "%R%" CMD INSTALL --debug --build r
 IF %ERRORLEVEL% NEQ 0 exit 1
```

This already helps us to successfully pass the configuration and compilation steps.
But not totally unexpected, it fails to link the R package against the MSVC-built `arrow.dll`.

```
g++ -shared -o arrow.dll tmp.def array.o array_from_vector.o array_to_vector.o arraydata.o arrowExports.o buffer.o chunkedarray.o compression.o compute.o csv.o dataset.o datatype.o expression.o feather.o field.o filesystem.o io.o json.o memorypool.o message.o parquet.o recordbatch.o recordbatchreader.o recordbatchwriter.o schema.o symbols.o table.o threadpool.o -L%BUILD_PREFIX%\Library/lib -larrow_dataset -lparquet -larrow -LD:/bld/r-arrow_1586674369380/_h_env/lib/R/../../Library/mingw-w64/lib/x64 -LD:/bld/r-arrow_1586674369380/_h_env/lib/R/../../Library/mingw-w64/lib -LD:/bld/r-arrow_1586674369380/_h_env/lib/R/bin/x64 -lR
array.o: In function `Array__Slice1(std::shared_ptr<arrow::Array> const&, int)':
D:\bld\r-arrow_1586674369380\work\r\src/array.cpp:28: undefined reference to `__imp__ZNK5arrow5Array5SliceEx'
array.o: In function `Array__Slice2(std::shared_ptr<arrow::Array> const&, int, int)':
D:\bld\r-arrow_1586674369380\work\r\src/array.cpp:34: undefined reference to `__imp__ZNK5arrow5Array5SliceExx'
array.o: In function `Array__null_count(std::shared_ptr<arrow::Array> const&)':
D:\bld\r-arrow_1586674369380\work\r\src/array.cpp:52: undefined reference to `__imp__ZNK5arrow5Array10null_countEv'
array.o: In function `Array__ToString[abi:cxx11](std::shared_ptr<arrow::Array> const&)':
D:\bld\r-arrow_1586674369380\work\r\src/array.cpp:61: undefined reference to `__imp__ZNK5arrow5Array8ToStringB5cxx11Ev'
```

### 4: Use clang instead of MinGW

The problem here is that the C++ ABI of MinGW (which we used here) and MSVC (which was used to build arrow-cpp) is different.
Thus we cannot easily link these libraries together.
Luckily the linkage works on the C-level ABI.
As R has a C-API, we can try to attempt to use a toolchain that supports building recent C++ on Windows (with the necessary bits to that the R dependencies of Arrow need) and linking against the C ABI of MSVC and the R binaries.
For this, Isuru Fernando gave me the important tip to try out clang and its linker `lld`.

As normally R builds packages with the same compiler it was built with, we needed to do some trickery to get it to use `clang`.
At the end of this section, we have much cleaner approach for this but as it is also interesting how we approached this for some readers, we're first having a look at the nasty one.
Therefore we wrote a custom bash script to adjust some paths in R's `Makeconf`.
This is the file where R stores its compiler and build configurations.
We could have also made these calls in the `bld.bat` but as my bash proficiency is much better than the `cmd` one, we took this shortcut.
This then lead to the following `build_win.sh`:

```
#!/bin/bash

set -exuo pipefail

export BUILD_PREFIX_SH=${BUILD_PREFIX//\\//}
export BUILD_PREFIX_SH=/${BUILD_PREFIX_SH//:}

# Patch Makeconf
sed -i -e "s;g++;${BUILD_PREFIX_SH}/Library/bin/clang++.exe;g" $BUILD_PREFIX/Lib/R/etc/x64/Makeconf
sed -i -e "s;nm;${BUILD_PREFIX_SH}/Library/bin/llvm-nm.exe;g" $BUILD_PREFIX/Lib/R/etc/x64/Makeconf
sed -i -e "s;gnu++11;gnu++14;g" $BUILD_PREFIX/Lib/R/etc/x64/Makeconf
```

This also needed some adjustments to the requirements, as we don't use the default R/MinGW compiler anymore.
Additionally, we need to install clang's toolchain.

```
diff --git a/recipe/bld.bat b/recipe/bld.bat
index 9e26180..850df4b 100644
--- a/recipe/bld.bat
+++ b/recipe/bld.bat
@@ -1,3 +1,5 @@
+bash %RECIPE_DIR%/build_win.sh
+IF %ERRORLEVEL% NEQ 0 exit 1
 cp %RECIPE_DIR%/configure.win r
 IF %ERRORLEVEL% NEQ 0 exit 1
 "%R%" CMD INSTALL --debug --build r
diff --git a/recipe/meta.yaml b/recipe/meta.yaml
index d2a14bd..2ecb30f 100644
--- a/recipe/meta.yaml
+++ b/recipe/meta.yaml
@@ -22,6 +22,10 @@ requirements:
   build:
{% raw %}     - {{ compiler('c') }}        # [not win]
     - {{ compiler('cxx') }}      # [not win]
+    - clang       # [win]
+    - clangxx     # [win]
+    - llvm-tools  # [win]
+    - lld         # [win]
     - pkg-config
     - {{ posix }}make
     - {{ posix }}sed{% endraw %}
```

While the logs look promising, it sadly failed to build as it could not find the referenced `clang-cpp.exe`.

```
/D/bld/r-arrow_1586760590742/_h_env/Library/bin/clang-cpp.exe  -std=gnu++14 -I"D:/bld/r-arrow_1586760590742/_h_env/lib/R/include" -DNDEBUG -DNDEBUG -I"%BUILD_PREFIX%\Library/include" -I"%BUILD_PREFIX%/include" -O2 -D_CRT_SECURE_NO_WARNINGS -DARROW_R_WITH_ARROW -fuse-ld=lld -I"D:/bld/r-arrow_1586760590742/_h_env/Lib/R/library/Rcpp/include"   -I"D:/bld/r-arrow_1586760590742/_h_env/lib/R/../../Library/mingw-w64/include"     -O2 -Wall -gdwarf-2 -march=x86-64 -mtune=generic -c array.cpp -o array.o
bash.exe: warning: could not find /tmp, please create!
/bin/sh: /D/bld/r-arrow_1586760590742/_h_env/Library/bin/clang-cpp.exe: No such file or directory
make: *** [array.o] Error 127
```

The issue here is that a base MinGW installation isn't able to resolve all native Windows paths.
To use them, you also need to list `m2-filesystem` as a build dependency.
After that, building with `clang` works fine with one exception that we will cover in the next section.

To enable using Clang with R packages on Windows in a more clean fashion, we wrote an activation package that sets up all the correct environment variables and dependencies.
One can use it using the Jinja2 function `{% raw %}{{ compiler('r_clang') }}{% endraw %}`.
This is part of the [r_clang_activation](https://github.com/conda-forge/r_clang_activation-feedstock) feedstock.
This allow us to remove the `sed` commands on the `Makeconf` and reduces the build requirements to:

```yaml
   build:
{% raw %}     - {{ compiler('c') }}        # [not win]
     - {{ compiler('cxx') }}      # [not win]
     - {{ compiler('r_clang') }}  # [win]
     - pkg-config
     - {{ posix }}make
     - {{ posix }}sed{% endraw %}
```

### 5: RCpp compilation failure with clang

While using `clang` and `lld` enables us to compile and link the C++ part of the R Arrow package, we faced one compilation issue with RCpp.
Here a symbol / macro resolution was different than with the typical compilers RCpp is normally built with.

```
/C/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Library/bin/clang++.exe  -std=gnu++14 -I"C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/lib/R/include" -D
NDEBUG -DNDEBUG -I"%BUILD_PREFIX%\Library/include" -I"%BUILD_PREFIX%/include" -O2 -D_CRT_SECURE_NO_WARNINGS -DARROW_R_WITH_ARROW -fuse-ld=lld -I"C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848
759312/_h_env/Lib/R/library/Rcpp/include"   -I"C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/lib/R/../../Library/mingw-w64/include"     -O2 -Wall -gdwarf-2 -march=x86-64 -mtune=
generic -c recordbatch.cpp -o recordbatch.o
In file included from recordbatch.cpp:18:
In file included from ././arrow_types.h:145:
In file included from C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp.h:57:
C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp/DataFrame.h:82:24: error: no matching function for call to 'abs'
                return abs(INTEGER(rn)[1]);
                       ^~~
recordbatch.cpp:100:47: note: in instantiation of member function 'Rcpp::DataFrame_Impl<PreserveStorage>::nrow' requested here
  return arrow::RecordBatch::Make(schema, tbl.nrow(), std::move(arrays));
                                              ^
C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp/sugar/functions/math.h:42:19: note: candidate function not viable: no known conversion from 'int'
      to 'SEXP' (aka 'SEXPREC *') for 1st argument
VECTORIZED_MATH_1(abs,::fabs)
                  ^
C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp/sugar/block/Vectorized_Math.h:91:9: note: expanded from macro 'VECTORIZED_MATH_1'
        __NAME__( SEXP x){ return __NAME__( NumericVector( x ) ) ; }             \
        ^
C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp/sugar/functions/math.h:42:19: note: candidate template ignored: could not match
      'VectorBase<14, NA, type-parameter-0-1>' against 'int'
C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/Rcpp/include\Rcpp/sugar/functions/math.h:42:19: note: candidate template ignored: could not match
      'VectorBase<13, NA, type-parameter-0-1>' against 'int'
1 error generated.
make: *** [recordbatch.o] Error 1C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/lib/R/etc/x64/Makeconf:215: recipe for target 'recordbatch.o' failed

ERROR: compilation failed for package 'arrow'
* removing 'C:/Users/Administrator/miniconda3/conda-bld/r-arrow_1586848759312/_h_env/Lib/R/library/arrow'
```

The fix for it was to prepend the `std::` namespace to the called function.
This [has already been merged upstream](https://github.com/RcppCore/Rcpp/pull/1069).

### 6: DLL naming

With the package built, it was now time to test whether it actually works.
Sadly loading it via `library(arrow)` directly led to a confusing error.

```
error: unable to load shared object
'…/libs/x64/arrow.dll':

  LoadLibrary failure:  %1 is not a valid Win32 application.
```

Using one of the Windows equivalent of `ldd` `dumpbin /DEPENDENTS` did not reveal any missing library.
After thinking about its output, it revealed a serious naming issue.
The Arrow C++ library and the Arrow R library were both named `arrow.dll`.
On unix-like sytems, this collision is normally avoided by linking libraries with a relative `RPATH` to each other.
Sadly, this concept doesn't exist on Windows and the lookup mechanism searches through the directories on `PATH`.
This meant that the R Arrow library `arrow.dll` was trying to load itself as a dependency.

To break this cycle, we had to rename one of the two.
As our main goal was to work with the identical Arrow C++ library as the Python package uses, we could only rename the R one.
This had the caveat though that the tooling `r-arrow` uses is all built upon the fact that the DLL of the package has the same name as the package.
Luckily R provides the possibility to hook into the last step of the library installation using a custom `install.libs.R` script.
We wrote one that installs the newly-built `arrow.dll` to the target location as `lib_arrow.dll`.

```r
src_dir <- file.path(R_PACKAGE_SOURCE, "src", fsep = "/")
dest_dir <- file.path(R_PACKAGE_DIR, paste0("libs", R_ARCH), fsep="/")

dir.create(file.path(R_PACKAGE_DIR, paste0("libs", R_ARCH), fsep="/"), recursive = TRUE, showWarnings = FALSE)
file.copy(file.path(src_dir, "arrow.dll", fsep = "/"), file.path(dest_dir, "lib_arrow.dll", fsep = "/"))
```

As this is a non-default library name, we still need to patch two things, the `useDynLib` command and the package load hook on the C side, to make this work:

```sh
# Rename arrow.dll to lib_arrow.dll to avoid conflicts with the arrow-cpp arrow.dll
sed -i -e 's/R_init_arrow/R_init_lib_arrow/g' r/src/arrowExports.cpp
sed -i -e 's/useDynLib(arrow/useDynLib(lib_arrow/g' r/NAMESPACE
```

### 7: Symbol visibility

With this fixed, we can load the library but executing a function still doesn't work.
Calling an R function from the `arrow` function leads to missing objects.
These missing objects are the references to the C++ functions in the binding.

```
> library('arrow'); data(mtcars); write_parquet(mtcars, 'test.parquet')

Attaching package: 'arrow'

The following object is masked from 'package:utils':

    timestamp

Error in shared_ptr_is_null(xp) : 
  object '_arrow_shared_ptr_is_null' not found
Calls: write_parquet -> <Anonymous> -> shared_ptr -> shared_ptr_is_null
Execution halted
```

The issue here is that the symbol visibility is different between the MSVC and the MinGW world.
In the MinGW world, every function of a dynamic library can be used by default from the outside.
Whereas on the MSVC side, only function declared with `__declspec(dllexport)` are visible as exported symbols.
As we have used a MSVC-flavoured `clang`, all the symbols in the `lib_arrow.dll` are also hidden from the outside.
Due to the fact that we are using a non-standard compiler setup for the R world, there was also no fully pre-made solution.
Still, there exists a macro named `attribute_visible` that is defined on other platforms where symbols need to be exported.
On Windows, it is empty though.
To make things work, we override the use of this macro with the needed attributed `__declspec(dllexport)` in the RCpp codebase.
We did not yet supply an upstream patch here as the clean solution would have to be to added to the R, not the RCpp, codebase.
We are only going to do this once we have seen that the MSVC-flavoured-clang inside a MinGW approach actually works stable.
Otherwise it might be an edge case no maintainer wants to support.

```
sed -i -e 's/attribute_visible/__declspec(dllexport)/g' $BUILD_PREFIX/Lib/R/library/Rcpp/include/RcppCommon.h
```.

Finally, with this last patch, we can compile, link and run the `r-arrow` code.
This means we now have a fully functional package.

## Conclusion

In constrast to normal Windows packaging, this was a bit different adventure.
Normally, the issue is that packages don't compile because a specific POSIX API they are using is not implemented.
This was the first time, I have had to tackle two compiler toolchains that don't work together at the C++ ABI level.
It was exciting that at the end, we got to a working version that might also be used for other packages.

I really like that you can use `clang` and its implementation of modern C++ on Windows while targeting MSVC as the ABI.
If you have a package that builds on Linux but doesn't work on Windows, you should definitely give this setup a try.
If your package is working though fine as-is currently, stick to the RTools40 toolchain.
This is much better tested, more mature and you will get less bugs but more support than for the experimental things described in the blog post.

Considering the `r-arrow` package, please give it a try and report issues at https://github.com/conda-forge/r-arrow-feedstock.
I'm happy that it is running but I haven't spend more time looking into whether performance differs from the RTools40 build.
It would be nice if someone had a look at this and made some real-world measurements.

*Title picture: Photo by Photo by [Miguel Orós](https://unsplash.com/@miki202og?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText).

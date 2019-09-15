---
layout: post
title: How we build Apache Arrow's manylinux wheels
feature_image: "https://picsum.photos/id/133/2560/600"
---

Apache Arrow is provided for Python users through two package managers, `pip` and `conda`.
The first mechanism, providing binary, pip-installable Python wheels is [currently unmaintained as highlighted on the mailing list](https://lists.apache.org/thread.html/128a2bec285ad45aa4189ebb39a15b39dcf6d91c4ab0278ff4f7cdea@%3Cdev.arrow.apache.org%3E).
There has been shoutouts for help, e.g. [on Twitter](https://twitter.com/ApacheArrow/status/1163919996214501377) that we need new contributors who look after the builds.
We sadly cannot point to all issues that arise, mostly issues come up slowly when people use new releases.
Thus we need people that look after the wheel builds and have an understanding what is done to provide these binaries to the end-user.
As having worked quite some time on the Linux wheel, I thought the best thing to handover would be to give an introduction to the current build process.

The main reason I gave up the maintenance is that I have personally no need more for the wheel based builds.
As with the other core Arrow Python contributors, I use solely `conda` for development of Apache Arrow and nowadays also use `conda` solely for packaging at my main job.
In my previous company, we were using solely `pip` for the Python dependency management and a switch to conda was not happening in the foreseeable future so having Arrow wheels was the only option to get Arrow into production.

Nowadays, I'm happily working solely with `conda` as my package and environment manager.
For a developer of a package that not only has Python but also native dependencies, this is a huge relief (will go into detail later).
As there are some misconceptions what `conda` is and what using `conda` means, I recommend to read [Jake VanderPlas' blog post about conda's myths and misconceptions](https://jakevdp.github.io/blog/2016/08/25/conda-myths-and-misconceptions/).
While Apache Arrow doesn't have active maintenance of its wheels builds currently, other projects like [RAPIDS opted to don't publish any pip packages at all anymore](https://medium.com/rapids-ai/rapids-0-7-release-drops-pip-packages-47fc966e9472)

## What are wheels?

Wheels are one of the two typical ways Python packages are distributed on [PyPI](https://pypi.org/) nowadays.
The other popular choice is to provide source distributions (`sdist`) where the source code is simply bundled and an installation script (`python setup.py install`) is executed after the package was downloaded.
Wheels on the other side are a so-called `built-package format`.
This means that the contents of the archive that is downloaded is simply the files that will be copied as-is into the target environment.
Instead of running an installer script, the installation process of wheels is simply unarchiving files.

The definition of wheels is laid out in [PEP 427](https://www.python.org/dev/peps/pep-0427/).
Instead of `built-package format`, wheels are often also called binary Python packages.
While for a pure Python packages, this may seem confusing (Python code is always human readable and not binary), it makes sense for packages that contain native (e.g. C++) code.
As no installer script is executed, all code must already be provided in a runnable form, i.e. no compilation process can happen on installation.

One of the main benefits of wheels is the that you only the to install the runtime requirements of a package, not the build-time requirements.
Whereas for a pure Python package, this mean that you don't need to have setuptools, for packages with native code, this means that you don't need to have a compiler stack and the corresponding build tools for the native code (e.g. `cmake`) installed.

## What is manylinux1 and why do we need it?

As binary packages don't only have Python dependencies but also dependencies to other binary libraries on the operating system we must ensure that we don't install packages where these dependencies are not met.
As `pip` is a *Python* package manager, it is not taking care of the installation of binary, non-Python, OS-specific libraries.
Thus the first thing in building a Python package containing native code is to package all non-Python dependencies also into this package.

Still there are some binary dependencies that you cannot or shouldn't package alongside your binary Python package.
This includes the OS runtime, a libc implementation and at least the graphics subsystem of your distribution (read `X server and friends`).

In the case of Windows and OS X it is simple to define a baseline on which libraries are provided by the OS and which should be part of the Python package.
This is due the quite limited set of versions Windows and OS X have and that newer OS versions support still binaries that were built for an older version.
Thus for e.g. an OSX package, it is sufficient to say `this Python wheel can be install on OSX 10.6 and later` to specify its binary needs.

The picture is totally different when looking into the support for binary, portable Python packages on Linux.
Here the sheer number of Linux distributions and the choice of combinations of Linux kernel, libc implementation (GNU vs musl vs …), versions thereof and various other options mean that there is no single layer that is present on all Linux distributions.
Still, for the major share of modern Linux distributions, it is possible to define a minimal baseline of what Linux kernel, which `libc` and which `libstdc++` should be present.

This baseline layer is defined in [PEP 513 – manylinux1](https://www.python.org/dev/peps/pep-0513/) and [PEP 571 – manylinux 2010](https://www.python.org/dev/peps/pep-0571/).
Both use CentOS as the base distribution of what is available as base libraries.
This doesn't mean that everything which can be installed on CentOS is available but the version of `libc`, `libstdc++`, and some other libraries that are available on CentOS 5 or 6 can be expected to be present on any system that claims to support `manylinux1` or `manylinux2010`.

Already from the names, one can see that binaries built with the `manylinux2010` name are ones that run on all major distributions that were released after 2010.
It doesn't include all available Linux distributions.
The distributions versions that were released earlier with a previous version of the GNU `libc` aren't supported by the binaries and you will get a missing symbol error on startup.
Furthermore, Linux distributions that aren't using the GNU `libc` are also not supported, the most prominent here is Alpine Linux as it uses the `musl libc`.
For these distributions there is currently no `manylinuxX`-like standard and users have to fallback to compiling packages from source.


## How to build a manylinux1 wheel?

Given that we now have an understanding of the idea of `manylinux1`, how does one build a wheel for it?
To keep it simple, we take `pandas` as an example.
While `pandas` has native code for performance improvements, it has no other external binary dependencies than `numpy`.
`numpy` is special as it is also a Python package and comes with a (backward-)stable ABI (application binary interface).
Also `numpy` is special as we build code against `numpy` headers but don't link to it.
Rather `pandas` embeds C-functions that 
Thus `pandas` doesn't have a direct _link_ dependency on `numpy`, making it free from external binary dependencies.
This is a detail you normally wouldn't need to know but is making this example much simpler.

To build the wheel, we clone the `pandas` GIT repository and start the `quay.io/pypa/manylinux2010_x86_64` docker container.
In the docker command, we have mounted the pandas source code in `/io` and thus run all commands in there.
As the first step, we need to install the build-time requirements of `pandas`, `cython` and `numpy`.
We install `numpy==1.16.4` at build time and through `numpy`'s forward-compatibility, we support running all `numpy` version starting from 1.16.
Then we call `python setup.py bdist_wheel` to start the actual wheel build.

```
# git clone https://github.com/pandas-dev/pandas.git
# docker run -ti -v $(pwd):/io quay.io/pypa/manylinux2010_x86_64 /bin/bash
[root@f0536b7fedce /]# cd /io/
[root@f0536b7fedce io]# /opt/python/cp37-cp37m/bin/python -m pip install cython numpy==1.16.4
Collecting cython
  Downloading https://files.pythonhosted.org/packages/f1/d3/03a01bcf424eb86d3e9d818e2082ced2d512001af89183fca6f550c32bc2/Cython-0.29.13-cp37-cp37m-manylinux1_x86_64.whl (2.1MB)
     |████████████████████████████████| 2.1MB 2.1MB/s
Collecting numpy==1.16.4
  Downloading https://files.pythonhosted.org/packages/fc/d1/45be1144b03b6b1e24f9a924f23f66b4ad030d834ad31fb9e5581bd328af/numpy-1.16.4-cp37-cp37m-manylinux1_x86_64.whl (17.3MB)
     |████████████████████████████████| 17.3MB 6.3MB/s
Installing collected packages: cython, numpy
Successfully installed cython-0.29.13 numpy-1.16.4
[root@f0536b7fedce io]# /opt/python/cp37-cp37m/bin/python setup.py bdist_wheel
<…>

```

Up until now, this is like every other wheel build, simply we're running it in a special docker container.
To make it a `manylinux1` wheel, we need to check for the `manylinux1` compatibility and fix possible issues.
For this, we can use the `auditwheel repair` command.
It checks whether only the supported symbols from `libc` and `libstdc++` are used and the package is self-contained, i.e. it doesn't link to any non-whitelisted shared library outside of the wheel.
If a packages uses newer symbols from `libc` or `libstdc++` that are not yet present in the `manylinux1` version of these libraries, it errors out.
Mostly likely you have used your own compiler chain here that doesn't link against the old versions of these libraries.
`auditwheel` cannot make any tricks here to make your binary compatible.

In the case of shared libraries that are neither in the `manylinux1` whitelist nor packaged in the wheel, `auditwheel repair` is capable though of handling this situation.
It will take the library as-is and rename it from `libfoo.so` to `libfoo-<hash>.so` to make it unique to the level that the library name is different for every build but the same for the identical version of this library.
It then packages it also into the wheel so that it is shipped along with the binaries requiring it.
As the location of the library has changed, `auditwheel` uses the `patchelf` tool under the hood to change the library link in the binary from an absolute system path to a path relative to the binary as the wheel will be extracted to any location on installation.

```
[root@f0536b7fedce io]# auditwheel repair --plat manylinux1_x86_64 dist/pandas-0.25.0+240.g0d0daa846-cp37-cp37m-linux_x86_64.whl
INFO:auditwheel.main_repair:Repairing pandas-0.25.0+240.g0d0daa846-cp37-cp37m-linux_x86_64.whl
INFO:auditwheel.wheeltools:Previous filename tags: linux_x86_64
INFO:auditwheel.wheeltools:New filename tags: manylinux1_x86_64
INFO:auditwheel.wheeltools:Previous WHEEL info tags: cp37-cp37m-linux_x86_64
INFO:auditwheel.wheeltools:New WHEEL info tags: cp37-cp37m-manylinux1_x86_64
INFO:auditwheel.main_repair:
Fixed-up wheel written to /io/wheelhouse/pandas-0.25.0+240.g0d0daa846-cp37-cp37m-manylinux1_x86_64.whl
```

The resulting wheel `pandas-0.25.0+240.g0d0daa846-cp37-cp37m-manylinux1_x86_64.whl` is now ready for distribution and can be run on any Linux distribution that meets the minimal requirements for `manylinux1`.

## Why is it so complicated for Apache Arrow?

As seen in the above `pandas` case, building a `manylinux1` compatible wheel can be a straight-forward process.
Sadly for Arrow, this isn't the case.
The main reason for this is that in contrast to `pandas`, Arrow has a huge set of third-party binary dependencies.
As these are non-Python dependencies, there is not package for them in `pip`.

While not all dependencies of Arrow C++ are hard dependencies (nearly most of them are optional), we want to build the wheels with all features enabled and thus also need to build all dependencies.
You can get a good overview of the dependencies by looking at the conda requirements for the [C++ library](https://github.com/apache/arrow/blob/327057e99ee7280ac8a11846258aec7bcdccc01d/ci/conda_env_cpp.yml) and additionally [for the Gandiva](https://github.com/apache/arrow/blob/327057e99ee7280ac8a11846258aec7bcdccc01d/ci/conda_env_gandiva.yml).
Gandiva is here separated as it brings in a heavy LLVM dependency.
While being only a single dependency, it needs more time to build than the remaining other dependencies combined.

As for the old CentOS 5-based `manylinux1` container no binaries for these libraries are available, we also maintain [a large set of scripts to build all dependencies](https://github.com/apache/arrow/tree/327057e99ee7280ac8a11846258aec7bcdccc01d/python/manylinux2010/scripts).
Additionally, for the `manylinux1` based build, we can use a modern GCC 7/8 whereas for CentOS 5 we are limited to GCC 4.8 and might also need to adjust upstream packages to work with this old compiler version.
In contrast to typical distribution packages, we don't build dynamically-linked shared libraries but also adjust all the build scripts to emit static libraries.
This has the benefit for us that at the end we only have to deal with various `libarrow_*.so` and not also deal with bundling and relinking all third-party dependencies in the packaging scripts.

Building static libraries also has the benefit that we can better control which symbols are exported and visible to other binaries that will be loaded into the same process.
This is important when a user has a different module that also gets loaded into the Python process that also depends on one of our dependencies but uses a different.
For more background on this, we will see what happens when this is not correctly done in the section about the crashes we saw in combination with Tensorflow.

## Annoying end users issue #1: "Undefined symbol in turbodbc_arrow_support.so"

There are several issues that pop up with end users when they install the `pyarrow` wheel and continue with their normal `pip` / Python workflow.
Some of these issues can be fixed on the Arrow side, others will always stay as a hurdle.
I have picked the two most prominent issues with the `manylinux1` wheels.
The first one is sadly due to build with old compilers and old standards and is an inconvenience that cannot be fixed on the Arrow side whereas the second one was introduced by the `manylinux1` build environment but which we could luckily work around on the Arrow.

The first issue is one that is often encountered by users of `turbodbc`, an ODBC implementation written in C++ for Python that also supports Arrow as a input/output format.
The issues happens when a user has installed `pyarrow` via the `manylinux1` wheel and then installs `turbodbc` via `pip` **from source** and has a Linux distribution that was build using the `cxx11` ABI.
The error message is always in the like of `undefined symbol: _ZN5arrow6StatusC1ENS_10StatusCodeERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE` on either building or importing `turbodbc`.
You can see this as an example as [a user-reported issue about "Undefined symbol in turbodbc_arrow_support.so"](https://github.com/blue-yonder/turbodbc/issues/218).

To understand what goes wrong here, we need to have a look at the introduction of C++11 to `libstdc++`.
With the introduction of C++11, some implementation of existing classes in `libstdc++` had to be changed to conform with the newly defined standard.
One of these classes was `std::string` / `std::basic_string` where the new standard forbid to implement these classes with copy-on-write optimizations.
These new implementations broke the ABI of `libstdc++` and thus would mean that libraries built with other versions of `libstdc++` wouldn't run on systems with the new version.
This situation also happened previously with GCC 3.
While there they had chosen to change the SONAME from `libstdc++.so.5` to `libstdc++.so.6`, this approach wasn't used this time as it lead to various issues like in some situations both versions with clashing implementations where loaded in the same program.
Instead [dual ABI approach](https://gcc.gnu.org/onlinedocs/gcc-5.2.0/libstdc++/manual/manual/using_dual_abi.html) was used that meant that `libstdc++` came with the two implementations `std::basic_string` and `std::__cxx11::basic_string`.
The C++ developer wouldn't have to change its code based on which of the two ABIs was used.
Whether `std::string` referred to a specialization of `std::basic_string` or `std::__cxx11::basic_string` was handled automatically by the compiler depending on the setting of the `_GLIBCXX_USE_CXX11_ABI` macro.
*For manylinux1, this macro was always set to `_GLIBCXX_USE_CXX11_ABI=0` as GCC 4.8 was before the introduction of the new ABI.* 

On newer Linux distributions, we have `_GLIBCXX_USE_CXX11_ABI=1` and thus all functions use the new `std::__cxx11::basic_string` implementation.
When we compile code that uses a function from Arrow that expects a string as a parameter, we only see the code definition in the header, not the compiled function in the shared library.
Thus the compile step passes successfully.
Depending on your linker settings, you will get an error when the turbodbc shared library is linked to the Arrow shared libraries or not as resolving missing symbols at runtime is needed for the Python symbols already and thus such errors are ignored.

In the above error message `undefined symbol: _ZN5arrow6StatusC1ENS_10StatusCodeERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE` we have exactly this case.
When this symbol is demangeled using `c++filt`, one can see that one of the parameters is of type `std::__cxx11::basic_string<char, …`.
This symbol is not provided by the `manylinux1` version of `libarrow.so`, it only has the `std::basic_string` version included.

```
# Symbol as in turbodbc_arrow_support.so
_ZN5arrow6StatusC1ENS_10StatusCodeERKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
# Demangled:
arrow::Status::Status(arrow::StatusCode, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)
# Symbol in libarrow.so
_ZN5arrow6StatusC1ENS_10StatusCodeERKNSt12basic_stringIcSt11char_traitsIcESaIcEEE
# Demangled:
arrow::Status::Status(arrow::StatusCode, std::basic_string<char, std::char_traits<char>, std::allocator<char> > const&)
```

The only ways to work around this as an end-user is to build `turbodbc` and its dependencies with `-D_GLIBCXX_USE_CXX11_ABI=0` or build `pyarrow` from source on your own instead of using the wheel.
This is issue is not only affecting Arrow but also every other C++11 project that is distributed as a `manylinux1` wheel.

## Annoying end users issue #2: "Segmentation fault after `import tensorflow`"

Our most prominent issue with `manylinux` wheels has been the interaction between the `pyarrow` and the `tensorflow` wheel.
Instead of a prominent error message, we only had a mysterious segmentation fault whose root cause was long unknown and we could only find (hacky) workaround before we fixed the issue in general.

One of the mysterious crashes is described at [ARROW-5130](https://issues.apache.org/jira/browse/ARROW-5130). Here a simple `import pyarrow; import tensorflow` led to a segmentation fault.
Surprisingly `import tensorflow; import pyarrow` did succeed.
Sadly the backtrace of the segmentation did not provide much information at first sight:

```
Program terminated with signal SIGSEGV, Segmentation fault.
#0 0x0000000000000000 in ?? ()
(gdb) bt
#0 0x0000000000000000 in ?? ()
#1 0x00007f529ee04410 in pthread_once () at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_once.S:103
#2 0x00007f5229a74efa in void std::call_once<void (&)()>(std::once_flag&, void (&)()) () from /usr/local/lib/python2.7/dist-packages/tensorflow/python/../libtensorflow_framework.so
#3 0x00007f5229a74f3e in tensorflow::port::TestCPUFeature(tensorflow::port::CPUFeature) () from /usr/local/lib/python2.7/dist-packages/tensorflow/python/../libtensorflow_framework.so
#4 0x00007f522978b561 in tensorflow::port::(anonymous namespace)::CheckFeatureOrDie(tensorflow::port::CPUFeature, std::string const&) ()
from /usr/local/lib/python2.7/dist-packages/tensorflow/python/../libtensorflow_framework.so
…
```

As the approach "first import tensorflow" worked nicely, this was the initial solution to this was ["ARROW-1960: Pre-emptively import TensorFlow if it is available"](https://issues.apache.org/jira/browse/ARROW-1960).
This only fixed the symptom, we didn't really understand the cause of the crash and why it was dependent on the import order.

One of the main ingredients here was that Tensorflow at that stage was a `manylinux1` wheel in its name but failed the `auditwheel` check as it was compiled on a newer Ubuntu system with different versions of `libc` and `libstdc++`.
At the moment `import pyarrow; import tensorflow` is working simply because nowadays both provide `manylinux2010` compliant wheels.
Additionally, we have fixed the underlying issue that caused the crashes when both wheels were build with different toolchains.


The problem occurred in the combination of `tensorflow==1.10` and `pyarrow==0.12.1` and was fixed with `pyarrow==0.14.0`.
The commit that fixed the problem in Arrow was [`ARROW-5130: [C++][Python] Limit exporting of std::* symbols`](https://github.com/apache/arrow/commit/0289af23a780ee0947c97707b82bad4df2f6e30f).
At first sight changing the visibility of symbols should not normally affect the behavior of a program, only whether there are linker errors due to unresolved symbols in the build.
Thus we have a look at the exported symbols, in this case specifically for `std::__once_call` as this was mentioned explicitly in the commit and also appears as `std::call_once` in the above backtrace.

```
nm libarrow.so.12 | grep __once_call
0000000000000000 B _ZSt11__once_call
0000000000000008 B _ZSt15__once_callable
00000000001d8980 W _ZSt16__once_call_implISt12_Bind_simpleIFPFvvEvEEEvv
00000000001bb400 W _ZSt16__once_call_implISt12_Bind_simpleIFSt7_Mem_fnIMNSt13__future_base11_State_baseEFvRSt8functionIFSt10unique_ptrINS2_12_Result_baseENS6_8_DeleterEEvEERbEEPS3_St17reference_wrapperISA_ESH_IbEEEEvv
```

When we look at `tensorflow==1.10`, we see a slightly larger number of symbols but only a single overlapping one, namely `_ZSt16__once_call_implISt12_Bind_simpleIFPFvvEvEEEvv`. When we unmangle this symbol, we get `void std::__once_call_impl<std::_Bind_simple<void (*())()> >()`.
This symbol is marked with type `W`.
According to the [nm documentation](https://sourceware.org/binutils/docs/binutils/nm.html), we have a weak, exported symbol.
This is a symbol that can be linked many times into a program.
If it is only linked weakly, the first symbol will be used but if there is a non-weak (normal) instance of the symbol, this will be preferred over the others.
Thus if `pyarrow` is loaded first, its implementation will be used, otherwise the one from tensorflow.

```
nm tensorflow/libtensorflow_framework.so | grep __once_call
                 U _ZSt11__once_call
                 U _ZSt15__once_callable
0000000000672a50 W _ZSt16__once_call_implISt12_Bind_simpleIFPFvvEvEEEvv
000000000067c3e0 t _ZSt16__once_call_implISt12_Bind_simpleIFZN10tensorflow13profile_utils8CpuUtils34GetCpuUtilsHelperSingletonInstanceEvEUlvE_vEEEvv
0000000000672a10 t _ZSt16__once_call_implISt12_Bind_simpleIFZN10tensorflow4port26InfoAboutUnusedCPUFeaturesEvEUlvE_vEEEvv
0000000000a7f4a0 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re23RE24InitERKNS1_11StringPieceERKNS2_7OptionsEEUlvE_vEEEvv
0000000000a7b170 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re24Prog10first_byteEvEUlPS2_E_S3_EEEvv
0000000000a57990 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re24Prog6GetDFAENS2_9MatchKindEEUlPS2_E0_S4_EEEvv
0000000000a579f0 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re24Prog6GetDFAENS2_9MatchKindEEUlPS2_E1_S4_EEEvv
0000000000a57920 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re24Prog6GetDFAENS2_9MatchKindEEUlPS2_E_S4_EEEvv
0000000000a86cf0 t _ZSt16__once_call_implISt12_Bind_simpleIFZN3re26Regexp6IncrefEvEUlvE_vEEEvv
0000000000a812a0 t _ZSt16__once_call_implISt12_Bind_simpleIFZNK3re23RE211ReverseProgEvEUlPKS2_E_S4_EEEvv
0000000000a7f3f0 t _ZSt16__once_call_implISt12_Bind_simpleIFZNK3re23RE219CapturingGroupNamesEvEUlPKS2_E_S4_EEEvv
0000000000a7f390 t _ZSt16__once_call_implISt12_Bind_simpleIFZNK3re23RE220NamedCapturingGroupsEvEUlPKS2_E_S4_EEEvv
0000000000a7f460 t _ZSt16__once_call_implISt12_Bind_simpleIFZNK3re23RE223NumberOfCapturingGroupsEvEUlPKS2_E_S4_EEEvv
```

When we now take a look at the symbols from `pyarrow==0.14.0`, we can see that the type of the symbol has changed to `t` which is in the local text/code section.
This mean that `pyarrow` will make use of it but won't export it globally to other libraries, i.e. `tensorflow` will not pick it up.

```
nm pyarrow/libarrow.so.14 | grep __once_call
0000000000001858 b _ZSt11__once_call
0000000000001860 b _ZSt15__once_callable
0000000000848030 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPKN6google8protobuf14FileDescriptorEES5_EEEvv
00000000007e2510 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPKN6google8protobuf15FieldDescriptorEES5_EEEvv
0000000000847fd0 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPKN6google8protobuf20FileDescriptorTablesEES5_EEEvv
00000000007e24e0 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPKN6google8protobuf8internal22AssignDescriptorsTableEEPS4_EEEvv
0000000000848060 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPN6google8protobuf8internal14LazyDescriptorEES5_EEEvv
0000000000848000 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvPSt4pairIPKN6google8protobuf20FileDescriptorTablesEPKNS3_14SourceCodeInfoEEESB_EEEvv
000000000040ac60 t _ZSt16__once_call_implISt12_Bind_simpleIFPFvvEvEEEvv
0000000000402f30 t _ZSt16__once_call_implISt12_Bind_simpleIFSt7_Mem_fnIMNSt13__future_base11_State_baseEFvRSt8functionIFSt10unique_ptrINS2_12_Result_baseENS6_8_DeleterEEvEERbEEPS3_St17reference_wrapperISA_ESH_IbEEEEvv
0000000000410ae0 t _ZSt16__once_call_implISt12_Bind_simpleIFZNK14arrow_vendored4date9time_zone4initEvEUlvE_vEEEvv
```

While I'm unable to point to why this one symbol is crashing the interaction between the both libraries, we can still confirm that it is really the source of the problem by creating a minimal example.

The first code snippet we need is a minimal C++ library that makes use of `std::call_once`. For the simplicity, we will use this code to mirror `pyarrow` as well as `tensorflow` in the above case.
As `pyarrow` was built in the manylinux1 image and tensorflow on Ubuntu 14.04, we also build the C++ library on the respective systems with `g++ -pthread -fno-rtti -fPIC -std=c++11 -shared threading.cpp -o libthread-manylinux1.so`.

```c++
#include <iostream>
#include <mutex>
#include <thread>

std::once_flag flag;

void function() {
  std::cout << "Function called" << std::endl;
}

void do_call() {
  std::call_once(flag, function);
}

extern "C" void do_action() {
  std::thread thread1(do_call);
  std::thread thread2(do_call);
  thread1.join();
  thread2.join();
}
```

To replicate Python, we write a C program that loads the dynamic libraries at runtime.
Having a C main program helps here as we don't want to load any C++ symbols before the libraries are loaded to replicate the initial problem.
We compile and run the following code with `gcc -fPIC -ggdb load-both.c -ldl && ./a.out`:

```c
#include <dlfcn.h>

typedef void (*func_t)();

int main() {
  void* manylinux1_handle = dlopen("./libthread-manylinux1.so", RTLD_NOW);
  func_t func = dlsym(manylinux1_handle, "do_action");
  func();

  void* ubuntu_handle = dlopen("./libthread-ubuntu.so", RTLD_NOW);
  func = dlsym(ubuntu_handle, "do_action");
  func();
}
```

With the above order of first loading the manylinux1 variant and then the Ubuntu one, we can reproduce the segmentation fault.
If we switch the order, it runs smoothly just like in the bug reports.
This nicely shows the cause of the `pyarrow`<->`tensorflow` crash that happened with the incompatible wheels.
*Sadly, from this point, I don't have any more in-depth knowledge about the inner workings of the STL symbols but I hope this is enough for the experts to dig in.*

***TL;DR***: The `pyarrow`/`tensorflow` issue boils down to:

* `tensorflow` wasn't actually `manylinux1` compliant although it was tagged as such
* `pyarrow` and `tensorflow` both provide an implementation of `std::call_once<void(*)()>`. Sadly these implementations are incompatible and thus lead to a crash.

It was fixed by (one of the two would have been enough):

* `tensorflow` and `pyarrow` both now provide `manylinux2010` compliant wheels built with the same toolchain
* `pyarrow` is no longer exporting the offending symbols
* `pyarrow` can work with the symbol exported by `tensorflow`, still it would be best when `tensorflow` would also not export the symbol.

## Where to go from here

The most simple next step for us Apache Arrow maintainers to move forward would be to only provide conda packages as these are the most reliable way for our Python users to install Arrow.
As sadly a lot of people depend on `pip` as a sole Python package manager and may not even have the chance to switch to conda, there will be continued interest in the wheels.
Thus as a start, I wanted to document the build process and the main hurdles that we came along with the usage of these wheels so that it will be easier for new maintainers to step in.

For the manylinux side of things, there are also things happening.
While it took a multi-year effort to finalize the `manylinux1` successor `manylinxu2010`, work is underway to release a new [`manylinux2014` standard/toolchain with PEP 599](https://www.python.org/dev/peps/pep-0599/).
This will bring an even more modern toolchain as the base for building redistributable Linux wheels.
To resolve the issue that drafting a PEP and testing a toolchain and coordinating the rollout of a new wheel tag in this is case is much simpler in future, there are currently discussions happening on this in [PEP 600](https://www.python.org/dev/peps/pep-0600/).

Another interesting approach is [`conda-press`](https://github.com/regro/conda-press), a tool that takes conda packages and turns them into Python wheels.
It is especially interesting as it also can produce fat wheels, meaning that it can also package all non-Python dependencies of a package into the wheel.
In combination with `auditwheel` and `delocate`, this might be a good alternative on how we can use the stable process of producing `pyarrow` conda packages and still keep releasing wheels without the hassle of a separate toolchain.
This may well make it into a future blog post of mine.
…

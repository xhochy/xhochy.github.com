---
layout: post
title: Apache Arrow on the Apple M1
feature_image: "/images/natalie-grainger-7atL8cXOvXs-unsplash.jpg"
---

In [the previous blog post](https://uwekorn.com/2021/01/04/first-two-weeks-with-the-m1.html) I explained how I got a well-working setup on my M1 MacBook. With that in place, I mostly worked on my main work setup running. But as a core Apache Arrow developer, I was also very eager to spend the extra mile and get Arrow (the C++ and Python part) working on the M1. As outlined in the previous post, I used `conda-forge` as the source for all dependencies and Arrow itself to build binary packages.

If you are eager to use `pyarrow` on the M1, you can install it just as every other conda package using

```
conda install -c conda-forge pyarrow
```

If you still need to setup `conda` on the M1 [conda-forge already has native installers for it](https://github.com/conda-forge/miniforge/blob/master/README.md).

## Building Arrow using cross-compilation on CI

Building Arrow C++ for the new architecture also meant building all its dependencies for the new architecture. Luckily as clang and the macOS SDK support cross-compiling, we could already start to build the conda-forge packages before the first machines were shipped. Most of the changes were straight-forward as [outlined in the conda-forge post about `osx-arm64`](https://conda-forge.org/blog/posts/2020-10-29-macos-arm64/) as most dependencies used either CMake or autotools for their build system.

We only had issues with packages that called themselves during the build phase. Here we couldn't use the builds for `osx-arm64` as Rosetta 2 only supports running Intel code on Arm64 devices, not the other way around. Thus, in this case, we had to build the Intel and the ARM version in CI and point to the correct executables to be used during the cross-compilation build.

An example of a dependency that called itself is `grpc-cpp`. This compiles a `protoc` plugin and then calls that later in the process to compile some gRPC-specific protobuf files. `conda-build` by itself already supports cross-compilation by having a `build` (what you need to compile) and a `host` (the dependencies you link to) section in its requirements. As we are going to build gRPC twice, we needed to duplicate its host dependencies also in the build section. Using `conda-build`'s selectors we can limit that though to only the cases where we are doing a cross-compilation.


```diff
--- a/recipe/meta.yaml
+++ b/recipe/meta.yaml
@@ -24,6 +25,12 @@ requirements:
     # `protoc` is also used for building
     - libprotobuf
     - ninja
+    # We need all host deps also in build for cross-compiling
+    - abseil-cpp  # [build_platform != target_platform]
+    - c-ares      # [build_platform != target_platform]
+    - re2         # [build_platform != target_platform]
+    - openssl     # [build_platform != target_platform]
+    - zlib        # [build_platform != target_platform]
   host:
     - abseil-cpp
     - c-ares
```

Similarly, we changed the `build.sh` to build gRPC for the current architecture when `$CONDA_BUILD_CROSS_COMPILATION` is set. The compiler packages on conda-forge define for that case special environment variables `$CC_FOR_BUILD` and `$CXX_FOR_BUILD` that contain the compilers for the architecture the build is running on. Sadly with the CMake setup we have in the conda-forge setup, gRPC's CMake setup doesn't detect that we are cross-compiling and chooses the wrong protobuf compiler. Thus we supplied the following patch that hardcodes the variables for the case we are using in our build scripts. With that it was sufficient to pass `-DProtobuf_PROTOC_EXECUTABLE=$BUILD_PREFIX/bin/protoc` to the CMake command line to get the build passing. The gRPC build for the local architecture was detected automatically as we have installed it into the `$BUILD_PREFIX`.


```diff
--- cmake/protobuf.cmake	2020-09-11 09:52:16.000000000 +0200
+++ cmake/protobuf.cmake	2020-09-11 09:53:28.000000000 +0200
@@ -75,21 +75,8 @@
       set(_gRPC_PROTOBUF_PROTOC_LIBRARIES ${PROTOBUF_PROTOC_LIBRARIES})
       set(_gRPC_PROTOBUF_WELLKNOWN_INCLUDE_DIR ${PROTOBUF_INCLUDE_DIRS})
     endif()
-    if(TARGET protobuf::protoc)
-      set(_gRPC_PROTOBUF_PROTOC protobuf::protoc)
-      if(CMAKE_CROSSCOMPILING)
-        find_program(_gRPC_PROTOBUF_PROTOC_EXECUTABLE protoc)
-      else()
-        set(_gRPC_PROTOBUF_PROTOC_EXECUTABLE $<TARGET_FILE:protobuf::protoc>)
-      endif()
-    else()
-      set(_gRPC_PROTOBUF_PROTOC ${PROTOBUF_PROTOC_EXECUTABLE})
-      if(CMAKE_CROSSCOMPILING)
-        find_program(_gRPC_PROTOBUF_PROTOC_EXECUTABLE protoc)
-      else()
-        set(_gRPC_PROTOBUF_PROTOC_EXECUTABLE ${PROTOBUF_PROTOC_EXECUTABLE})
-      endif()
-    endif()
+    set(_gRPC_PROTOBUF_PROTOC_EXECUTABLE ${Protobuf_PROTOC_EXECUTABLE})
+    set(_gRPC_PROTOBUF_PROTOC ${Protobuf_PROTOC_EXECUTABLE})
     set(_gRPC_FIND_PROTOBUF "if(NOT Protobuf_FOUND AND NOT PROTOBUF_FOUND)\n  find_package(Protobuf ${gRPC_PROTOBUF_PACKAGE_TYPE})\nendif()")
   endif()
 endif()
```

With all the dependencies built, it was now time to adjust the [`arrow-cpp-feedstock`, the repository that defines the conda-forge build for `arrow-cpp` and `pyarrow`](https://github.com/conda-forge/arrow-cpp-feedstock). The fleet of conda-forge bots already [made an automatic PR](https://github.com/conda-forge/arrow-cpp-feedstock/pull/195) once all the dependencies were built. Sadly this didn't work out of the box due to Arrow's rather complicated setup. The first thing we needed to do was to also supply the `build`-dependencies needed for cross-compilation to the `pyarrow` and not only the `arrow-cpp` output.

```diff
--- a/recipe/meta.yaml
+++ b/recipe/meta.yaml
@@ -245,6 +245,10 @@ outputs:
         {% raw %}{{ "- arrow-cuda" if cuda_enabled else "" }}{% endraw %}
     requirements:
       build:
+        - python                                 # [build_platform != target_platform]
+        - cross-python_{{ target_platform }}     # [build_platform != target_platform]
+        - cython                                 # [build_platform != target_platform]
+        - numpy                                  # [build_platform != target_platform]
         - cmake 3.16.*
         - ninja
         - make  # [unix]
```

As at this point LLVM 10 wasn't built for `osx-arm64`, we also disabled Gandiva for now. Meanwhile, this has been built for the new architecture, so we will probably test and reenable it soon. Also, we needed to adjust the line in the `CMakeLists.txt` for Arrow Flight to have it find the right gRPC protobuf plugin. Otherwise, it would always have used the one for the target architecture that wouldn't run on CI.

```diff
--- a/recipe/build-arrow.sh
+++ b/recipe/build-arrow.sh
@@ -34,6 +34,14 @@ else
     EXTRA_CMAKE_ARGS=" ${EXTRA_CMAKE_ARGS} -DARROW_CUDA=OFF"
 fi
 
+if [[ "${target_platform}" == "osx-arm64" ]]; then
+    # We need llvm 11+ support in Arrow for this
+    EXTRA_CMAKE_ARGS=" ${EXTRA_CMAKE_ARGS} -DARROW_GANDIVA=OFF"
+    sed -i "s;\${GRPC_CPP_PLUGIN};${BUILD_PREFIX}/bin/grpc_cpp_plugin;g" ../src/arrow/flight/CMakeLists.txt
+else
+    EXTRA_CMAKE_ARGS=" ${EXTRA_CMAKE_ARGS} -DARROW_GANDIVA=ON"
+fi
+
 cmake \
     -DARROW_
```

Due to some issues with the build and failing tests reported by other uses, we also disabled the most performant allocators backed by jemalloc and mimalloc for now. As they use things that are heavily depending on system internals, they needed more adjustments and testing as they would otherwise prohibit the successful import of the `pyarrow` Python module.

At this point, we have built `arrow-cpp` and `pyarrow` packages but don't know whether they work as we don't have a machine to test on. Still, this was a major boost in getting things running as we would otherwise have spent a week or more waiting for everything to be built before we could test anything once I got my hands on an M1.

## CMake Issues

Once my MacBook with an M1 processor arrived, I also came to the point where everything was running so smoothly that I could finally invest some time into building Arrow natively on the M1. This directly runs into the issue that the `CMAKE_SYSTEM_PROCESSOR` is slightly different on macOS with the system being detected as supporting AVX2. As this is an x86-only instruction set, the compilation failed with errors about invalid assembly instructions. The underlying issue here was `CMAKE_SYSTEM_PROCESSOR` was `arm64` whereas the other ARM systems we have yet built Arrow on reported it either as `ARM64` or `aarch64`. We could resolve this in [ARROW-10873](https://issues.apache.org/jira/browse/ARROW-10873) / [PR#8893](https://patch-diff.githubusercontent.com/raw/apache/arrow/pull/8893) with a simple patch and got Arrow to build natively, too.

```diff
--- a/cpp/cmake_modules/SetupCxxFlags.cmake
+++ b/cpp/cmake_modules/SetupCxxFlags.cmake
@@ -24,7 +24,7 @@ include(CheckCXXSourceCompiles)
 message(STATUS "System processor: ${CMAKE_SYSTEM_PROCESSOR}")
 
 if(NOT DEFINED ARROW_CPU_FLAG)
-  if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|ARM64")
+  if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|ARM64|arm64")
     set(ARROW_CPU_FLAG "armv8")
   elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "armv7")
     set(ARROW_CPU_FLAG "armv7")
```

## Codesigning Issues with gtest

Once Arrow C++ was built locally, running the tests sadly lead to all of them failing. Typically when everything fails, one would expect a `Segmentation fault` as the error message but in that case, we only got `Killed.`. Even weirder, when trying to debug a test, the call to `lldb -- testcase` also got `Killed.`. Not the test broke but also the debugger didn't start if we supplied an Arrow test as the binary. This is something that I have never seen on any platform before. Even more surprising was that you could start `lldb` after you have restarted your system and the test would run. But once you have run the test in `lldb` or if you have run the test after a system restart without `lldb`, `lldb` itself would get a `Killed.` again. *As weird as it sounds, this was reproducible after a lot of restarts and a good night of sleep.*

Even more troubling to me was that linking also failed once you have executed a test and did a clean build (`rm -rf build`).

```

[1/330] Linking CXX shared library release/libarrow_testing.300.0.0.dylib
FAILED: release/libarrow_testing.300.0.0.dylib
: && /Users/uwe/mambaforge/envs/arrow-dev-1/bin/arm64-apple-darwin20.0.0-clang++ -ftree-vectorize -fPIC -fPIE -fstack-protector-strong -O2 -pipe -stdlib=libc++ -fvisibility-inlines-hidden -std=c++14 -fmessage-length=0 -isystem /Users/uwe/mambaforge/envs/arrow-dev-1/include -fno-omit-frame-pointer -g -Qunused-arguments -fcolor-diagnostics -O3 -DNDEBUG  -Wall -Wno-unknown-warning-option -Wno-pass-failed -march=armv8-a  -O3 -DNDEBUG -arch arm64 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk -dynamiclib -Wl,-headerpad_max_install_names -Wl,-pie -Wl,-headerpad_max_install_names -Wl,-dead_strip_dylibs -Wl,-rpath,/Users/uwe/mambaforge/envs/arrow-dev-1/lib -L/Users/uwe/mambaforge/envs/arrow-dev-1/lib  -undefined dynamic_lookup   -framework CoreFoundation -compatibility_version 300.0.0 -current_version 300.0.0 -o release/libarrow_testing.300.0.0.dylib -install_name @rpath/libarrow_testing.300.dylib src/arrow/CMakeFiles/arrow_testing_objlib.dir/io/test_common.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/ipc/test_common.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/json_integration.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/json_internal.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/gtest_util.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/random.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/generator.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/testing/util.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/csv/test_common.cc.o src/arrow/CMakeFiles/arrow_testing_objlib.dir/filesystem/test_util.cc.o  release/libarrow.300.0.0.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libgtest.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libssl.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libcrypto.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libbrotlienc.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libbrotlidec.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libbrotlicommon.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/liborc.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libprotobuf.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-config.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-transfer.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-identity-management.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-cognito-identity.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-sts.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-s3.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-cpp-sdk-core.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-c-event-stream.1.0.0.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-checksums.1.0.0.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libaws-c-common.1.0.0.dylib  -pthread  -lpthread  -framework CoreFoundation  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libutf8proc.dylib  /Users/uwe/mambaforge/envs/arrow-dev-1/lib/libre2.dylib && :
ld: warning: -pie being ignored. It is only used when linking a main executable
clang-11: error: unable to execute command: Killed: 9
clang-11: error: linker command failed due to signal (use -v to see invocation)
```

But at least this pointed us in the direction that there is an issue with one of the dependencies of `libarrow_testing` that wasn't present in any of the other Arrow libraries. Thus I was able to [build a minimal reproducible example](https://github.com/xhochy/gtest-fail) that had a small dummy unit test with `gtest` that showed the same behaviour. This was quite helpful as we could point to the fact that there must be something broken with the `gtest` package on conda-forge. We also knew quite quickly that the possible source of the issue is related to code signing. With the ARM chips, macOS now requires every executable to be code signed when executed. No signature is not acceptable but a dummy signature, e.g. using `ldid -S <binary>` as we do in the conda tooling is ok. This signature needs to match the binary though and needs to be updated once the binary is updated. This is typically the case when conda installs an environment and replaces hardcoded paths in binaries. Therefore the newest conda release also has support to update the signature when installing a binary file where it replaces a prefix.

When a binary using an invalid signature is loaded, the system reports that in the kernel log (you can monitor that via `sudo log stream --predicate 'sender = "kernel"'`) and kills the program. In the log we could see that `libgtest.dylib` was the issue. Sadly code signing it did not resolve the `Killed.` tests. After a lot of searching the internet on code signing issues, we came around [a bug report that highlighted an issue that binaries stored in a specific inode of the filesystem cannot be executed if this inode previously contained an executable that failed verification](https://openradar.appspot.com/FB8914243).

This bug report was in line with the observation that rebooting did help in some cases as this flushed the verification table and all inodes were equal again. Also, we could verify that we trigger that bug by changing the code signing from "just" calling `ldid -S` to the following.

```
cp $CONDA_PREFIX/lib/libgtest.dylib /tmp/
ldid -S /tmp/libgtest.dylib
mv /tmp/libgtest.dylib $CONDA_PREFIX/lib/libgtest.dylib
```

Due to the first copy, we ensure that the new file is in a different inode as the failing one and we were able to execute the test.

Still, we had to find out why our `libgtest.dylib` had an invalid signature as we took care that all steps in the build chain update the signature. So we have one that didn't do it. The first observation here was that `libgtest.dylib` has no hardcoded paths that need to be replaced on installation. Thus this safety net wasn't triggered. As we were quite sure that we produce a valid signature in the build using the linker in our toolchain, the issue must be hidden somewhere after the build and before installation. We then also found it in the CI logs where `conda-build` was changing RPATHs and friends to be relative instead of absolute using `install_name_tool` as shown in the logs:

```
Fixing linking of libgtest.dylib in /Users/runner/miniforge3/conda-bld/gtest_1607939533240/…/lib/libgtest.dylib
New link location is lib/libgtest.dylib
/Applications/Xcode_12.2.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/install_name_tool: warning: changes being made to the file will invalidate the code signature in: /Users/runner/miniforge3/conda-bld/gtest_1607939533240/…/lib/libgtest.dylib
```

We now have fixed this by using a patched version of `install_name_tool` that does the changes after a copy and also fixes the signature on the newly generated files. With this adjusted, all tests run and we only got a single failure we will look into later in this post.

## mimalloc failed on load

While Arrow C++ works fine without the custom allocators, these are important reasons why some of the operations in Arrow are much faster to their STL-equivalent like `std::vector`. We do have different constraints in Arrow and thus we can make slightly different optimizations that speed up our typical operations. By using a different allocator, we can improve e.g. in the case where we build up arrays of yet unknown size. For that, you need good reallocation support. `mimalloc` and `jemalloc` both support that among their various other improvements.

While having one of them working on a platform would be sufficient, we still look here into building both to have a choice. Sadly, we directly hit a roadblock as `mimalloc` doesn't compile on the M1. This is due to the use of the `-march=native` flag for ARM machines. While this gives the best performance for the machine you are currently on, this is a nuisance for packager that build redistributable binaries. There, you want to build binaries that run on all platforms. As your CI systems that build the binaries are often quite new and powerful, they typically also support instruction sets like AVX2 that not all of your users have. Thus building with `-march=native` yields binaries that wouldn't run on their machine due to unsupported instructions. In the case of the M1 setup, we luckily don't even ship binaries with that flag enabled as the compiler rejects `-march=native` for the `arm64` target on macOS. Thus we have submitted [a PR that simply doesn't use that flag on Apple Silicon](https://github.com/microsoft/mimalloc/pull/344).

Once we have fixed the compilation issue, we could build Arrow with `mimalloc` support. Sadly this led to an instantaneous `Segmentation fault`. This fault could also be reproduced in a debug build of mimalloc's unit test:

```
% lldb ./mimalloc-test-api
(lldb) target create "./mimalloc-test-api"
Current executable set to '/Users/uwe/Development/mimalloc/build/mimalloc-test-api' (arm64).
(lldb) run
Process 17703 launched: '/Users/uwe/Development/mimalloc/build/mimalloc-test-api' (arm64)
mimalloc-test-api was compiled with optimization - stepping may behave oddly; variables may not be available.
Process 17703 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x000000010001f264 mimalloc-test-api`_mi_process_init [inlined] mi_tls_slot(slot=0) at mimalloc-internal.h:711:9 [opt]
   708 	#elif defined(__aarch64__)
   709 	  void** tcb; UNUSED(ofs);
   710 	  __asm__ volatile ("mrs %0, tpidr_el0" : "=r" (tcb));
-> 711 	  res = tcb[slot];
   712 	#endif
   713 	  return res;
   714 	}
Target 0: (mimalloc-test-api) stopped.
```

After a bit of suggestion, research and simply trying things out, we realized that the `tpidr_el0` register was always zero. This is a special AArch64 CPU register that is used to store a pointer to thread-local memory. On Linux, this is always set to a location where a chunk of memory has been allocated precisely for the current running thread. On macOS and as well on iOS as some other reports suggested, this wasn't the case. As one can see in the [AArch64 documentation about thread registers](https://developer.arm.com/documentation/ddi0500/j/System-Control/AArch64-register-summary/AArch64-thread-registers), there is an alternative read-only variant `tpidrro_el0` available for user-space (EL0) programs. This is actually the one that is populated by Apple operating systems. Sadly Linux systems don't set that in my tests so we need to special case here for the different operating systems as done in [the PR that fixes the test for Apple Silicon](https://github.com/microsoft/mimalloc/pull/346).

```diff
diff --git a/include/mimalloc-internal.h b/include/mimalloc-internal.h
index e3e78e4..df700d3 100644
--- a/include/mimalloc-internal.h
+++ b/include/mimalloc-internal.h
@@ -707,7 +707,11 @@ static inline void* mi_tls_slot(size_t slot) mi_attr_noexcept {
   res = tcb[slot];
 #elif defined(__aarch64__)
   void** tcb; UNUSED(ofs);
+#if defined(__MACH__)
+  __asm__ volatile ("mrs %0, tpidrro_el0" : "=r" (tcb));
+#else
   __asm__ volatile ("mrs %0, tpidr_el0" : "=r" (tcb));
+#endif
   res = tcb[slot];
 #endif
   return res;
@@ -730,7 +734,11 @@ static inline void mi_tls_slot_set(size_t slot, void* value) mi_attr_noexcept {
   tcb[slot] = value;
 #elif defined(__aarch64__)
   void** tcb; UNUSED(ofs);
+#if defined(__MACH__)
+  __asm__ volatile ("mrs %0, tpidrro_el0" : "=r" (tcb));
+#else
   __asm__ volatile ("mrs %0, tpidr_el0" : "=r" (tcb));
+#endif
   tcb[slot] = value;
 #endif
 }
```

For the curious that want to understand the above assembly, you need to have the knowledge about the existence of special registers that are used/populated by the system as in the above linked-page and you need to be aware of the `mrs` instruction. This is instruction is called [`move from system register`](https://developer.arm.com/documentation/dui0802/a/A64-General-Instructions/MRS). It moves the content from a system register into a general-purpose register. In the above code, it moves the content of the system register into the register which was selected by the compiler for the `tcb` variable.

Using the above results, we can incorporate this into the `arrow-cpp-feedstock` on conda-forge. Therefore, we need to let the `arrow-cpp` build run until the `mimalloc` sources are extracted and can then modify the `mimalloc` source code that comes with Arrow using `sed` commands.

```bash
if [[ "${target_platform}" == "osx-arm64" ]]; then
    ninja mimalloc_ep-prefix/src/mimalloc_ep-stamp/mimalloc_ep-patch
    sed -ie 's/list(APPEND mi_cflags -march=mative)//g' mimalloc_ep-prefix/src/mimalloc_ep/CMakeLists.txt
    # Use the correct register for thread-local storage
    sed -ie 's/tpidr_el0/tpidrro_el0/g' mimalloc_ep-prefix/src/mimalloc_ep/include/mimalloc-internal.h
fi
```

## jemalloc doesn't build

As we have `mimalloc` building and running successfully, the next task was to build and test `jemalloc`. For that, we first checked out the sources separately and build the tests. For this we have built `jemalloc` with debug and optional safety checks enabled as we care in this case about correctness and not about performance.

```
mamba create -n jemalloc autoconf c-compiler cxx-compiler
conda activate jemalloc
./configure --enable-debug --enable-opt-safety-checks
make -j8 all tests
make check
```

All tests have passed succesfully which should mean that we should be able to use it inside of Arrow. Sadly the build for it inside the Arrow C++ tree did fail with the following message:

```
checking build system type... x86_64-apple-darwin13.4.0
checking host system type...
-- stderr output is:
Invalid configuration `arm64-apple-darwin20.0.0': machine `arm64-apple' not recognized
configure: error: /bin/sh build-aux/config.sub arm64-apple-darwin20.0.0 failed
```

This is a common error we have seen in many `autotools`-based packages when building them for `osx-arm64`. The issue is here that the base information that `configure` has about the possible architectures it should run on doesn't include the new architecture. For this issue, we already have the common workaround that we have a small package called `gnuconfig` that ships with the latest version of this information. Normally the ARM OSX migration bot already detects its necessity but as `jemalloc` is not the main artefact, we need to add it manually. Also, as Arrow C++ unpacks the `jemalloc` sources during the build, we need to run the build until the sources are extracted but before `configure` is called to fix the inputs. This leads to the following addition in `build-arrow.sh`:

```bash
if [[ "${target_platform}" == "osx-arm64" ]]; then
     ninja jemalloc_ep-prefix/src/jemalloc_ep-stamp/jemalloc_ep-patch
     cp $BUILD_PREFIX/share/gnuconfig/config.* jemalloc_ep-prefix/src/jemalloc_ep/build-aux/
fi
```

With this, the cross-compilation build succeeds but it doesn't run on the M1. There it fails with the following message:

```
<jemalloc>: Unsupported system page size
…/conda_test_runner.sh: line 3: 47058 Segmentation fault: 11  "…/bin/python" -s "…/test_tmp/run_test.py"
```

By running the `jemalloc` configure directly on the M1, we see that the system page size is set to 14 and thus we can use that value in the cross-compilation step to make an informed (and correct) decision.

```bash
sed -ie 's;"--with-jemalloc-prefix\=je_arrow_";"--with-jemalloc-prefix\=je_arrow_" "--with-lg-page\=14";g' ../cmake_modules/ThirdpartyToolchain.cmake
```

With these fixes, now both allocators works on the M1.

## Arrow C++ test failure

As the conda package builds and the basic functionality also seems to work, it was time to also check that every corner of Arrow is working on the M1. Therefore, we have run all the Arrow C++ tests including the benchmarks. The tests all did pass, in the benchmarks we though got a failure in the parquet-encoding-benchmark with the following issue:

```
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x00000001009a5340 libparquet.300.dylib`arrow::internal::BaseSetBitRunReader<false>::LoadFullWord(this=0x000000016fdfce30) at bit_run_reader.h:264:5
   261 	    if (Reverse) {
   262 	      bitmap_ -= 8;
   263 	    }
-> 264 	    memcpy(&word, bitmap_, 8);
   265 	    if (!Reverse) {
   266 	      bitmap_ += 8;
   267 	    }
Target 0: (parquet-encoding-benchmark) stopped.
(lldb) p word
(uint64_t) $0 = 6171905360
(lldb) p bitmap_
(const uint8_t *) $1 = 0x0000000000000000
```

Here we see that we want to work on a null-pointer bitmap. This clearly fails. Astonishingly, this didn't happen yet on any other platform, probably the random generator is set up using a different seed. After having a look into the traceback, the issue is that we have generated an input array with no nulls. In this case, it is ok in the Arrow structure to omit the validity bitmap (the one that tells you whether a row is null or not). To fix this, we have special-cased the presence of the validity bitmap in the function that is calling the above failing `LoadFullWord` in [PR#9097](https://github.com/apache/arrow/pull/9097).

```diff
--- a/cpp/src/parquet/encoding.cc
+++ b/cpp/src/parquet/encoding.cc
@@ -110,12 +110,16 @@ class PlainEncoder : public EncoderImpl, virtual public TypedEncoder<DType> {
 
   void PutSpaced(const T* src, int num_values, const uint8_t* valid_bits,
                  int64_t valid_bits_offset) override {
-    PARQUET_ASSIGN_OR_THROW(auto buffer, ::arrow::AllocateBuffer(num_values * sizeof(T),
-                                                                 this->memory_pool()));
-    T* data = reinterpret_cast<T*>(buffer->mutable_data());
-    int num_valid_values = ::arrow::util::internal::SpacedCompress<T>(
-        src, num_values, valid_bits, valid_bits_offset, data);
-    Put(data, num_valid_values);
+    if (valid_bits != NULLPTR) {
+      PARQUET_ASSIGN_OR_THROW(auto buffer, ::arrow::AllocateBuffer(num_values * sizeof(T),
+                                                                   this->memory_pool()));
+      T* data = reinterpret_cast<T*>(buffer->mutable_data());
+      int num_valid_values = ::arrow::util::internal::SpacedCompress<T>(
+          src, num_values, valid_bits, valid_bits_offset, data);
+      Put(data, num_valid_values);
+    } else {
+      Put(src, num_values);
+    }
   }
```

During development, we also discovered that we trigger a compiler warning as we are not using the `cpu_info` object that is instantiated in some cases to do runtime dispatching for specific instruction sets. Currently, we only have special code paths for AVX2 and AVX512, both of which are unavailable on ARM64. Thus we also need to omit the construction of the `cpu_info` object itself on that platform using the following change in [PR
#9098](https://github.com/apache/arrow/pull/9098).

```diff
--- a/cpp/src/arrow/compute/function.cc
+++ b/cpp/src/arrow/compute/function.cc
@@ -81,7 +81,9 @@ Result<const KernelType*> DispatchExactImpl(const Function& func,
   }
 
   // Dispatch as the CPU feature
+#if defined(ARROW_HAVE_RUNTIME_AVX512) || defined(ARROW_HAVE_RUNTIME_AVX2)
   auto cpu_info = arrow::internal::CpuInfo::GetInstance();
+#endif
```

## Benchmarks

While I intended to only write about how I got Arrow C++/Python to run on the M1, people already asked me for benchmarks while I was working on this blog post. Thus I also will run the benchmarks on my Intel MacBook Pro and my ARM64 MacBook Pro and do a comprehensive comparison.

Personally, for me, the biggest interest lies in the Parquet loading performance. Thus the first look is at the New York Taxi trip dataset loading performance, my favourite. Here we simply test the loading performance for January 2016 for the yellow cab trips stored in Parquet. For comparison, I have used my 2019 MacBook Pro. We have built `pyarrow` using the conda-forge default `CXXFLAGS` but used `-march=native -mtune=native` for performance optimization on the Intel CPU so that it can unleash its full potential. As the M1 is currently the single ARM64 CPU targeted for macOS on conda-forge, here we already have that implicitly set.

```python
# Intel(R) Core(TM) i5-8259U CPU @ 2.30GHz
%timeit pd.read_parquet("yellow_tripdata_2016-01.parquet")
1.04 s ± 15.1 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)

# Apple M1
%timeit pd.read_parquet("yellow_tripdata_2016-01.parquet")
375 ms ± 2.56 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

This already shows an impressive 3x speedup. This partly comes from the better per-core performance from the M1 but also that it has 8 real (4 high-performance, 4 energy saving) cores instead of the 4 physical with hyperthreading on the Intel machine.

As in the above benchmark, we have been using Arrow master on commit `8896b42ed0205fc392036ec18c3052e78f93aa5e`. To get a view on the performance difference between the two machines for other Arrow functionality, we have run some of Arrow C++'s benchmarks using the following command on both machines. In contrast to the loading of a full Parquet file, these benchmarks only measure single core performance.

```
archery benchmark run \
    --suite-filter 'parquet-arrow-reader-writer-benchmark|arrow-csv-parser-benchmark|arrow-compute-vector-sort-benchmark|arrow-builder-benchmark' \
    --output intel.json \
    .
```

We have uploaded the results for [the Apple M1](https://gist.github.com/xhochy/b81bd84159dd05967e3956546438af6d) and [the Intel i5](https://gist.github.com/cf34bc0051d39d2bd467e631334611a9) and also run [a diff report using `archery benchmark diff`](https://gist.github.com/xhochy/60538f43b5d8fc4dce46ff8a76d28a9e). Here we can see that the performance for CSV parsing is similar on both machines, sometimes the M1 is even slower. Otherwise, we see really nice speedups in all benchmarks. Parquet reading (`BM_Read*`) is 2-5x faster depending on the data type, writing is about 50% faster and general-purpose computations on Arrow Tables are somewhere between 50% and an impressive 7.6x speedup.

## Conclusion

Getting Apache Arrow C++ & Python to work on the M1 was easier than expected but still involved some surprises. I would have expected that I would need to patch some of the dependencies but I would never have thought to have to go down to the system CPU register level. Still, due to the ability to cross-compile most things before I had a machine, it was a lot faster to get the final build going. Overall, the performance improvements are quite impressive given that this was only a build to "get it working". We have not yet done any performance optimizations that are specific to the hardware as we have done for the various levels of possible instructions sets that are available for x86 CPUs. Thus we would expect to get an even better performance when we have a look at the specifics of the M1.

At the end a word of caution: We are comparing two Laptop CPUs here that come in the same casing. The M1 being a lot faster than the Intel counterpart housed in them doesn't mean that it's faster than any other CPU. There are other options available for workstations and server, some of them even in Laptops. Overall though, the M1 is a really nice piece in this explicit setup, offering more performance with less emitted heat (and thus more battery life).

*Title picture: Photo by [Natalie Grainger](https://unsplash.com/@missnjc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*


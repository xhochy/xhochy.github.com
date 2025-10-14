---
layout: post
title: >
  2025-38: A week in conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

To continue to give you a glance into my conda-forge work, we're continuing with the second week of reporting. This week was a bit leaner on my activities here as I spend my time on preparing my [PyData Paris 2025](https://uwekorn.com/talks.html). 

Still, the talk also provided the need for one PR on conda-forge: I migrated the `polars-feedstock` to [make it `cargo-auditable`](https://github.com/conda-forge/polars-feedstock/pull/342). This has been useful as an example in my talk as polars statically links a huge number of Rust dependencies. Thus, it is a good case to highlight the existence of Phantom Dependencies (dependencies you have in your environment that your package manager doesn't list/is aware of).

Later this week, I continued working on the [`jaxlib v0.7.1`](https://github.com/conda-forge/jaxlib-feedstock/pull/321) PR. Here, we needed to move to `clang` as the default compiler on all platforms. This meant that we also needed to adjust the [`bazel-toolchain` feedstock](https://github.com/conda-forge/bazel-toolchain-feedstock/pull/25) to also handle this. Furthermore, to work around some build issues, we had to update the bundled `gloo` version and the `abseil-cpp` dependency that we pull in as a `conda` dependency. Once these dependencies were updated, we got a linker error with `abseil-cpp` on Linux. The linker error was for a missing symbol named `_ZN4absl12lts_202505124CordC1INSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEETnNSt9enable_ifIXsr3std7is_sameIT_S8_EE5valueEiE4typeELi0EEEOSA_`. The symbol actually existed in the source code, but was encoded in the binary as `_ZN4absl12lts_202505124CordC1INSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEELi0EEEOT_`. The noticeable difference here is that the first contains the condition `std::is_same`, whereas the latter doesn't. As this only happens on Linux, the obvious culprit here is the difference in compilers. `abseil-cpp` itself is built using conda-forge's standard compiler on Linux (GCC), whereas we moved to clang.
Looking at `clang`'s issue tracker, there is [llvm-project#85656](https://github.com/llvm/llvm-project/issues/85656).
This reveals that adding `-fclang-abi-compat=17`  as a compiler/linker flag solves the issues by falling back to the old symbol naming behaviour.

As part of the move to clang, we replaced not only GCC but also `nvcc` for compiling CUDA device code. This sadly led to one compilation error for `__glibcxx_requires_subscript`, which was appearing in device code, but had no respective device implementation:

```
In file included from /home/ubuntu/.pixi/envs/conda-build/conda-bld/jaxlib_1758038148038/_build_env/bin/../lib/gcc/x86_64-conda-linux-gnu/15.1.0/../../../gcc/x86_64-conda-linux-gnu/15.1.0/include/c++/functional:67:
/home/ubuntu/.pixi/envs/conda-build/conda-bld/jaxlib_1758038148038/_build_env/bin/../lib/gcc/x86_64-conda-linux-gnu/15.1.0/../../../gcc/x86_64-conda-linux-gnu/15.1.0/include/c++/array:210:2: error: reference to __host__ function '__glibcxx_assert_fail' in __host__ __device__ function
  210 |         __glibcxx_requires_subscript(__n);
      |         ^
/home/ubuntu/.pixi/envs/conda-build/conda-bld/jaxlib_1758038148038/_build_env/bin/../lib/gcc/x86_64-conda-linux-gnu/15.1.0/../../../gcc/x86_64-conda-linux-gnu/15.1.0/include/c++/debug/assertions.h:39:3: note: expanded from macro '__glibcxx_requires_subscript'
   39 |   __glibcxx_assert(_N < this->size())
      |   ^
/home/ubuntu/.pixi/envs/conda-build/conda-bld/jaxlib_1758038148038/_build_env/bin/../lib/gcc/x86_64-conda-linux-gnu/15.1.0/../../../gcc/x86_64-conda-linux-gnu/15.1.0/include/c++//x86_64-conda-linux-gnu/bits/c++config.h:658:12: note: expanded from macro '__glibcxx_assert'
  658 |       std::__glibcxx_assert_fail();                                     \
      |            ^
```
In GCC's and Clang's issue trackers we find similar-looking resources: [gcc#115740](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=115740), [llvm#95183](https://github.com/llvm/llvm-project/issues/95183), and [llvm#49727](https://github.com/llvm/llvm-project/issues/49727). The fix for that has been merged in [llvm#136133](https://github.com/llvm/llvm-project/pull/136133). I have tried to apply that fix in [clangdev-feedstock#383](https://github.com/conda-forge/clangdev-feedstock/pull/383), but sadly that was insufficient as raised in [clangdev-feedstock#384](https://github.com/conda-forge/clangdev-feedstock/issues/384). The correct fix then landed in [clangdev-feedstock#385](https://github.com/conda-forge/clangdev-feedstock/pull/385):

While my plan is to get to more in the Python 3.14 migration in the coming weeks, this week the work solely focused on kicking off the build for [python 3.14.0rc3](https://github.com/conda-forge/python-feedstock/pull/814).

Sadly, I ran into a bit of a mess with the AWS C stack this week. I needed to issue [aws-sdk-cpp-feedstock#970](https://github.com/conda-forge/aws-sdk-cpp-feedstock/pull/970) manually as [aws-sdk-cpp-feedstock#969](https://github.com/conda-forge/aws-sdk-cpp-feedstock/pull/969) was closed, but the bot did not retry it. Instead, the bot issued PRs to `arrow-cpp-feedstock` directly (see [arrow-cpp-feedstock#1854](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1854)). This led, in general, to a messy situation with PRs in the arrow-cpp repository.
At least [`[main] Rebuild for aws-c* (Sep '25)`](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1855) seemed to have worked fine, but all arrow versions there were already in the archive (i.e. all except the latest) had some download issues, and the PRs needed to be restarted. Restarting some failed jobs in [[20.x] Rebuild for aws-c* (Sep '25)](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1856), [[19.x] Rebuild for aws-c* (Sep '25)](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1857), and [[18.x] Rebuild for aws-c* (Sep '25)](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1858) made them pass again. Afterwards, we needed to rebase the PRs ([main](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1860), [20.x](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1852), [19.x](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1853), [18.x](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1854)) for `aws-crt-cpp 0.34.3` migration.

This week, there were also numerous small tasks I engaged in:
- The build for [`zstandard v0.25.0`](https://github.com/conda-forge/zstandard-feedstock/pull/69) was failing due to a missing dependency on `packaging` during the build. While fixing this, I also converted it to a v1 recipe.
- There was a contribution that added [ppc64le support for `cargo-auditable`](https://github.com/conda-forge/cargo-auditable-feedstock/pull/4). The contributor first used emulation, but switched to cross-compilation on request, which made the builds much faster.
- `@isuruf` fixed the [libxml2 migration for `msitools`](https://github.com/conda-forge/msitools-feedstock/pull/12#event-19709650378), and during my review, I realised that we were still using the old `gettext` outputs in the feedstock. Thus, I submitted [`Use new gettext outputs; convert to rattler`](https://github.com/conda-forge/msitools-feedstock/pull/14). Afterwards, I also bumped the version to [`msitools 0.106`](https://github.com/conda-forge/msitools-feedstock/pull/15).
- The [`libxml2 rebuild for librsvg`](https://github.com/conda-forge/librsvg-feedstock/pull/128) was failing with an overlinking error.
- Supported people on [dynare-preprocessor-feedstock#7](https://github.com/conda-forge/dynare-preprocessor-feedstock/pull/7) in their effort to build for `osx-64`
- Commented on [fmt-feedstock#67](https://github.com/conda-forge/fmt-feedstock/pull/67#issuecomment-3305642482) that we should not wait too long for things to start
- [vega-lite-cli-feedstock#41](https://github.com/conda-forge/vega-lite-cli-feedstock/pull/41) did not build as there was an outdated `os_version` section in `conda-forge.yml`. Thus, I took the opportunity and moved it to a v1 recipe. The sames fixes were also required to get [vega-cli-feedstock#44](https://github.com/conda-forge/vega-cli-feedstock/pull/44) to work.
- This week, it was also time for [a rust-nightly update again](https://github.com/conda-forge/rust-feedstock/pull/255). At the same time, we also merged [rust v1.90.0](https://github.com/conda-forge/rust-feedstock/pull/256) and then manually created the [activation upto to main 1.90; dev 1.92](https://github.com/conda-forge/rust-activation-feedstock/pull/84) to enable feedstocks to use the new compilers.

And finally, the list of pull requests where I only did a review and merged them, but no real interaction:
- [Rebuild for libxml2 - universal-ctags-feedstock#240](https://github.com/conda-forge/universal-ctags-feedstock/pull/240)
- [Rebuild for libxml2 - libxmlsec-feedstock#40](https://github.com/conda-forge/libxmlsec1-feedstock/pull/41)
- [Close out migration for vtk951](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7785)
- [Rebuild for r_base 4.5 - xgboost-feedstock#256](https://github.com/conda-forge/xgboost-feedstock/pull/256)
- [snowflake-connector-python v3.17.3](https://github.com/conda-forge/snowflake-connector-python-feedstock/pull/196)
- [miniforge-images: Bump ubuntu from noble-20250805 to noble-20250910 in /ubuntu](https://github.com/conda-forge/miniforge-images/pull/159)
- [aws-sdk-cpp: Rebuild for aws-c* (Sep '25)](https://github.com/conda-forge/aws-sdk-cpp-feedstock/pull/968)
- [Migration for aws-crt-cpp 0.34.4 (incl. automerge)](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7788)
- [Migration to s2n 1.5.26 (incl. automerge)](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7786)
- [docker-images: Bump actions/checkout from 4.2.2 to 5.0.0 in the github-actions group](https://github.com/conda-forge/docker-images/pull/312) (follow-up: [Fix comment on pinned action](https://github.com/conda-forge/docker-images/pull/315))
- Trigger CI on [nodejs v24.8.0](https://github.com/conda-forge/nodejs-feedstock/pull/412)
- Manually issued [nodejs v22.19.0](https://github.com/conda-forge/nodejs-feedstock/pull/413)
- Manually issued [nodejs v20.19.5](https://github.com/conda-forge/nodejs-feedstock/pull/414)
- Merged [ibm_db-feedstock#83](https://github.com/conda-forge/ibm_db-feedstock/pull/83) and issued a bot-rerun on [ibm_db#84](https://github.com/conda-forge/ibm_db-feedstock/pull/84).
- [symbolic v12.16.3](https://github.com/conda-forge/symbolic-feedstock/pull/7)

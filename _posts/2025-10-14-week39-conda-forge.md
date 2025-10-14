---
layout: post
title: >
  2025-39: A week in conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

In the third week of reporting on my conda-forge work, you will see how the large number of contributions happens quickly. As we're getting closer to the Python 3.14 release, I spent some time bringing that forward.

The week started with a [rust-dev update](https://github.com/conda-forge/rust-feedstock/pull/257). These updates are based on a script that regenerates a section of the `meta.yaml`, but the PRs themselves are not yet automated. In the same weekly cleaning session, I manually closed the [close aws_c_202509 migration](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7802) and [aws_crt_cpp0343 migration](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7803).

Last week, the `libxml2` migration progressed. Unfortunately, this also meant some breakage in a few places. I spent some time debugging [lxml-feedstock#105](https://github.com/conda-forge/lxml-feedstock/pull/105), where we need to pin to `libxml2-16 2.14.x`. This is much stricter than the `run_exports` of the `libxml2` packages enforces. In the end, this was not an unstable ABI but a change in the build that occurred in the latest minor release. While the PR for the `libxml2` for `ibm_db` passed, it sadly resulted in failing packages as it picked the system `libxml2` and not the new conda-forge version. Thus, we marked these builds as broken in [admin-requests#1672](https://github.com/conda-forge/admin-requests/pull/1672). We fixed the linkages in [ibm_db-feedstock#87](https://github.com/conda-forge/ibm_db-feedstock/pull/87) by pinning to the old version.

 As I'm a maintainer (also user) of `pcre2-feedstock`, I spent a bit of time reviewing the PRs that were still open on the `pcre2 10.46` migration:
 - Pinged the maintainer on [deepsearch-glm-feedstock#8](https://github.com/conda-forge/deepsearch-glm-feedstock/pull/8), [lighttpd-feedstock#45](https://github.com/conda-forge/lighttpd-feedstock/pull/45), [ugrep-feedstock#15](https://github.com/conda-forge/ugrep-feedstock/pull/15)
 - Triggered a `bot-rerun` on [htcondor-feedstock#316](https://github.com/conda-forge/htcondor-feedstock/pull/316) so that the PR is rebased on the latest main. The new PR [htcondor-feedstock#325](https://github.com/conda-forge/htcondor-feedstock/pull/325) was then swiftly merged by the maintainers.
 - Similarly, [r-rjava-feedstock#48](https://github.com/conda-forge/r-rjava-feedstock/pull/48) needed a `bot-rerun` and first the merge of [r-base-feedstock#383](https://github.com/conda-forge/r-base-feedstock/pull/386) so that the older R release also could be installed with the new `pcre2` 
 - Ignored the status of [julia-feedstock#297](https://github.com/conda-forge/julia-feedstock/pull/297), as the whole feedstock seems abandoned
 - Fixing [grep-feedstock#17](https://github.com/conda-forge/grep-feedstock/pull/17) was a bit trickier, as it needed the new release and still kept failing, as the cross-compiled tests don't work because of a path issue. Additionally, some behaviour changed recently on macOS so more integrated `gnulib` tests needed to be skipped. This was fixed by backporting the fixes from [gnulib-ccb60ee9](https://github.com/coreutils/gnulib/commit/ccb60ee9ab0205cfd23066bd520d1a0c9c7a0a76) and [gnulib-b49212bd](https://github.com/coreutils/gnulib/commit/b49212bd6ce6182d95af45d490d4de9f84bcc223)
 - On [nginx-feedstock#96](https://github.com/conda-forge/nginx-feedstock/pull/96), I made some progress by fixing the missing dependency on `libxcrypt` on newer Linux versions. Still, `configure` fails with an error on `libxslt`
* [uwsgi#100](https://github.com/conda-forge/uwsgi-feedstock/pull/100) fails due to bytecode mismatches in LTO code. Thus, it needs a rebuild of Python with the latest version of GCC. As Python 3.13 was already rebuilt, I made [python-feedstock#815](https://github.com/conda-forge/python-feedstock/pull/815), [python-feedstock#816](https://github.com/conda-forge/python-feedstock/pull/816), and [python-feedstock#817](https://github.com/conda-forge/python-feedstock/pull/817) for Python 3.10-3.12.

Meanwhile, I made minor fixes in [librsvg-feedstock#129](https://github.com/conda-forge/librsvg-feedstock/pull/129) and reviewed [admin-requests#1674](https://github.com/conda-forge/admin-requests/pull/1674) and [sqlglotrs-feedstock#33](https://github.com/conda-forge/sqlglotrs-feedstock/pull/33). As some colleagues required specific versions of Netbird, I contributed that via [staged-recipes#31096](https://github.com/conda-forge/staged-recipes/pull/31096) and added [`linux-aarch64`, `linux-ppc64le`, and `osx-arm64`](https://github.com/conda-forge/netbird-feedstock/pull/1) immediately afterwards.

As `jaxlib 0.7.1` was merged, it was time to continue work on [`jaxlib v0.7.2`](https://github.com/conda-forge/jaxlib-feedstock/pull/324). `@h-vetinari` already started working on this, so it wasn't a fresh start. As part of the new version, I updated the `absl` third-party dependency by adding new `log` targets. We could also drop the GCC 15 fixes we did for 0.7.1, as they were already integrated into this release. Sadly, the compilation now takes longer on `osx-64`, and to get it below the six-hour limit, I changed the oneDNN build from the bundled version to linking against the conda-forge package.
Meanwhile, we also attempted to migrate to CUDA 13. This sadly failed, as not even the latest Clang 21 release can build CUDA 13 device code. Still, I used the chance to contribute Clang 21 support to [google-ml-infra/rules_ml_toolchain#88](https://github.com/google-ml-infra/rules_ml_toolchain/pull/88).

There were again several (trivial) merges I did this week using my maintainer duties or `conda-forge/core` powers:
- [conda-forge-pinning-feedstock#7772](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7772)
* [conda-forge-pinning-feedstock#7736](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7736)
* [conda-forge.github.io#2603](https://github.com/conda-forge/conda-forge.github.io/pull/2603)
* [snowflake-connector-python-feedstock#197](https://github.com/conda-forge/snowflake-connector-python-feedstock/pull/197)
* [cargo-deny-feedstock#4](https://github.com/conda-forge/cargo-deny-feedstock/pull/4) (and enabled automerge)
* [aws-sdk-cpp-feedstock#971](https://github.com/conda-forge/aws-sdk-cpp-feedstock/pull/971)
* [admin-requests#1664](https://github.com/conda-forge/admin-requests/pull/1664)
* [libgit2-feedstock#86](https://github.com/conda-forge/libgit2-feedstock/pull/86)
* Closed [aws-sdk-cpp#972](https://github.com/conda-forge/aws-sdk-cpp-feedstock/pull/972) as rebuilds are costly
* Merged [staged-recipes#31099](https://github.com/conda-forge/staged-recipes/pull/31099) for a colleague
* [fisher-feedstock#32](https://github.com/conda-forge/fisher-feedstock/pull/32)
* [seqan-library-feedstock#12](https://github.com/conda-forge/seqan-library-feedstock/pull/12)
* [gdk-pixbuf-feedstock#57](https://github.com/conda-forge/gdk-pixbuf-feedstock/pull/57)
* [flock-feedstock#7](https://github.com/conda-forge/flock-feedstock/pull/7)
* [tree-sitter-sql-feedstock#6](https://github.com/conda-forge/tree-sitter-sql-feedstock/pull/6)
* [conda-forge-pinning-feedstock#7809](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7809)
* Triggered cirun and then merged [nodejs-feedstock#415](https://github.com/conda-forge/nodejs-feedstock/pull/415)
* [admin-requests#1673](https://github.com/conda-forge/admin-requests/pull/1673)
* [conda-forge-pinning-feedstock#7793](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7793)
* [psqlodbc-feedstock#15](https://github.com/conda-forge/psqlodbc-feedstock/pull/15)
* Due to network issues on the `apache.org` side, we needed to retrigger CI before we could merge [arrow-cpp-feedstock#1861](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1861), [arrow-cpp-feedstock#1863](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1863), [arrow-cpp-feedstock#1862](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1862), and [arrow-cpp-feedstock#1864](https://github.com/conda-forge/arrow-cpp-feedstock/pull/1864)
## Python 3.14
This week, I started my mini-sprint to bring the Python 3.14 migration forward so we will have a good release-day availability.

As a first step, I manually opened [pymsbuild-feedstock#25](https://github.com/conda-forge/pymsbuild-feedstock/pull/25), as the bot did not open a migration PR for it, but already had ones for packages depending on it. Similarly, [conda-feedstock#274](https://github.com/conda-forge/conda-feedstock/pull/274) failed because it needs `mamba` to migrate first.  We could close [futures-feedstock#17]( https://github.com/conda-forge/futures-feedstock/pull/17) , as `futures` is a Python 2.x-only package. I requested its archival in [admin-requests#1671](https://github.com/conda-forge/admin-requests/pull/1671). 

Then there was a round of PRs that could be merged, because they either were part of an abandoned feedstock or the maintainer had already approved it: [cchardet-feedstock#24](https://github.com/conda-forge/cchardet-feedstock/pull/24), [flt-feedstock#14](https://github.com/conda-forge/flt-feedstock/pull/14), and [fake-factory-feedstock#26](https://github.com/conda-forge/fake-factory-feedstock/pull/26)

For [fill-voids-feedstock#6](https://github.com/conda-forge/fill-voids-feedstock/pull/6) and [fisher-feedstock#30](https://github.com/conda-forge/fisher-feedstock/pull/30), I triggered a bot rerun because they looked good but had conflicts that prevented their direct merge.

Some PRs could easily be fixed by adding `setuptools` to `host`. Since Python 3.13, conda-forge no longer ships it as an implicit dependency of `pip` and thus it often breaks the build for some packages on these newer Python releases. I applied it to the following feedstocks and pinged the maintainers once the CI was green: [fastbpe-feedstock#18](https://github.com/conda-forge/fastbpe-feedstock/pull/18), [fastpath-feedstock#13](https://github.com/conda-forge/fastpath-feedstock/pull/13), and [future_fstrings-feedstock#21](https://github.com/conda-forge/future_fstrings-feedstock/pull/21).

Finally, there is a long list of feedstocks where I reviewed the PR and pinged the maintainers as the builds looked fine:
- [audioread-feedstock#33](https://github.com/conda-forge/audioread-feedstock/pull/33)
- [clarabel-feedstock#16](https://github.com/conda-forge/clarabel-feedstock/pull/16)
- [ephem-feedstock#40](https://github.com/conda-forge/ephem-feedstock/pull/40)
- [pycifrw-feedstock#32](https://github.com/conda-forge/pycifrw-feedstock/pull/32)
- [rebound-feedstock#116](https://github.com/conda-forge/rebound-feedstock/pull/116)
- [pathlob2-feedstock#24](https://github.com/conda-forge/pathlib2-feedstock/pull/24)
- [coreforecast-feedstock#23](https://github.com/conda-forge/coreforecast-feedstock/pull/23)
- [ezc3d-feedstock#97](https://github.com/conda-forge/ezc3d-feedstock/pull/97)
- [gstools-cython-feedstock#6](https://github.com/conda-forge/gstools-cython-feedstock/pull/6)
- [bitarray-feedstock#111](https://github.com/conda-forge/bitarray-feedstock/pull/111)
- [memray-feedstock#52](https://github.com/conda-forge/memray-feedstock/pull/52)
- [cassandra-feedstock#26](https://github.com/conda-forge/cassandra-feedstock/pull/26)
- [cyvlfeat-feedstock#22](https://github.com/conda-forge/cyvlfeat-feedstock/pull/22)
- [encor-feedstock#7](https://github.com/conda-forge/encor-feedstock/pull/7)
- [ezdxf-feedstock#80](https://github.com/conda-forge/ezdxf-feedstock/pull/80)
- [fastbencode-feedstock#7](https://github.com/conda-forge/fastbencode-feedstock/pull/7)
- [fastdtw-feedstock#18](https://github.com/conda-forge/fastdtw-feedstock/pull/18)
- [fastscapelib-f2py-feedstock#30](https://github.com/conda-forge/fastscapelib-f2py-feedstock/pull/30)
- [filepattern-feedstock#19](https://github.com/conda-forge/filepattern-feedstock/pull/19)
- [fimex-feedstock#91](https://github.com/conda-forge/fimex-feedstock/pull/91)
- [focal-stats-feedstock#14](https://github.com/conda-forge/focal-stats-feedstock/pull/14)
- [fortls-feedstock#41](https://github.com/conda-forge/fortls-feedstock/pull/41)

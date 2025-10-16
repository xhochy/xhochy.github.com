---
layout: post
title: >
  2025-40: A week in conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

This week was the week of PyData Paris and a public holiday in Germany. At the same time, my usual day-to-day work had to continue, so my conda-forge interactions were limited to simple (but many) reviews, fixes and merges. Larger changes have to wait for the next week(s).

As there was no maintainer response for a week, I merged [grep-feedstock#17](https://github.com/conda-forge/grep-feedstock/pull/17) and [lxml-feedstock#105](https://github.com/conda-forge/lxml-feedstock/pull/105). As [r-base-feedstock#386](https://github.com/conda-forge/r-base-feedstock/pull/386) was merged, we could trigger a re-run (and merge) on [r-rjava-feedstock#50](https://github.com/conda-forge/r-rjava-feedstock/pull/50). After the latest merge, the `nodejs` build on `osx-arm64` no longer fit into the six-hour timeframe on Azure Pipelines, so I started to look into a workaround in [nodejs-feedstock#416](https://github.com/conda-forge/nodejs-feedstock/pull/416). As the initial ideas didn't lead to a good result, this will need to be reviewed again later. In the `nodejs` landscape, [parceljs-feedstock#54](https://github.com/conda-forge/parceljs-feedstock/pull/54) was not automerged because an outdated `os_version` broke the build. I cleaned up the feedstock and converted it to a v1 recipe. [scorecard-feedstock#5](https://github.com/conda-forge/scorecard-feedstock/pull/5) failed to build with invalid license issues. Here, we needed to manually download the license of a third-party dependency instead of automatically fetching it via `go-licenses`. On the positive side, [conda-forge-pinning-feedstock#7822](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7822) was a straightforward merge.

The work on [`jaxlib 0.7.2`](https://github.com/conda-forge/jaxlib-feedstock/pull/324) continued with the problem that the GCC headers can be different between `build_platform` and `target_platform`; thus, we needed [bazel-toolchain-feedstock#26](https://github.com/conda-forge/bazel-toolchain-feedstock/pull/26) to handle this.
## Python 3.14
For the Python 3.14 migration, it was also quantity-over-quality this week as I wanted to ping some maintainers to merge PRs or at least get the clock started on PRs where I could use my `conda-forge/core` powers to merge after a week of no response.

During PyData Paris, the currently biggest blocker [pandas-feedstock#233](https://github.com/conda-forge/pandas-feedstock/pull/233) was merged after upstream released a Python 3.14-compatible version. I pinged the maintainer on [fastparquet-feedstock#78](https://github.com/conda-forge/fastparquet-feedstock/pull/78) directly, as this was the next one to get more nodes in the rebuild graph going.
Sadly, not everything is fixable. For example, [vowpalwabbit-feedstock#92](https://github.com/conda-forge/vowpalwabbit-feedstock/pull/92) failed because it doesn't build as it is not compatible with `fmt>=11`. However, moving the pin manually to `fmt=10` only leads to the following error. Luckily, some easy ones can be directly merged, like [tabmat-feedstock#56](https://github.com/conda-forge/tabmat-feedstock/pull/56).

On [pytorch-feedstock#420](https://github.com/conda-forge/pytorch-cpu-feedstock/pull/420), I commented with a link to the upstream issue that we will need to wait for PyTorch 2.10. Still, it might also be possible here that we can backport some patches to the 2.9 release, as this might be simpler/faster than getting the whole of 2.10 building on conda-forge infrastructure.

In the list of larger nodes in the Python 3.14 graph, we also had `pywinpty`. As the PR was failing on the current version, I first tried to upgrade to the latest one in [pywinpty-feedstock#64](https://github.com/conda-forge/pywinpty-feedstock/pull/64). Still, once I found out that this would require nuget and Windows Terminal to be build from source, I moved on an fixed [pywinpty-feedstock#65](https://github.com/conda-forge/pywinpty-feedstock/pull/65) by setting `PYO3_USE_ABI3_FORWARD_COMPATIBILITY=1` so that `maturin` builds for the new Python version.

[numcodecs-feedstock#119](https://github.com/conda-forge/numcodecs-feedstock/pull/119) initially failed because of a missing Python 3.14 build for `coverage`. After a rerender of the feedstock, the build now passes, and thus I requested a review from the maintainers. This was also the case for [pyfftw-feedstock#70](https://github.com/conda-forge/pyfftw-feedstock/pull/70). Similarly, I also asked a rerender for [framel-feedstock#44](https://github.com/conda-forge/framel-feedstock/pull/44). Sadly, in this case, Windows kept failing with MinGW compiler issues. These were solved by removing some old workarounds from `conda_build_config.yaml`. The build then passed, but framel's unit tests failed, and I moved on for now. In [srsly-feedstock#56](https://github.com/conda-forge/srsly-feedstock/pull/56) , the rerender revealed an upper pin in the package for `python<=3.13`. We concluded here to wait for a new release. As the `thrift-feedstock` looked a bit abandoned, I revived the version update in [thrift-feedstock#42](https://github.com/conda-forge/thrift-feedstock/pull/42) and converted this to a v1 recipe to tackle the Python 3.14 migration afterwards.

There were several more small fixes I did, that I'm only summarising as bullets here:
- Rebased [numexpr-feedstock#75](https://github.com/conda-forge/numexpr-feedstock/pull/75) and [bottleneck-feedstock#61](https://github.com/conda-forge/bottleneck-feedstock/pull/61) as both had new releases. In both cases, the build then passed.
- Requested a rerender on [argon2-cffi-bindings-feedstock#12](https://github.com/conda-forge/argon2-cffi-bindings-feedstock/pull/12); the builds then passed. 
- After the rerender on [mariadb-feedstock#19](https://github.com/conda-forge/mariadb-feedstock/pull/19), the builds sadly failed on Windows. This required linking to `zlib` to get them to pass.
- Created [sip-feedstock#100](https://github.com/conda-forge/sip-feedstock/pull/100) to build `sip` for Python 3.14 for this (older) release to move [pyqt-feedstock#159](https://github.com/conda-forge/pyqt-feedstock/pull/159) forward
- Updated [`onnx` to 1.19.0](https://github.com/conda-forge/onnx-feedstock/pull/134) first before attempting the Python 3.14 migration in [onnx-feedstock#135](https://github.com/conda-forge/onnx-feedstock/pull/135). This also needed an older protobuf version rebuild in [protobuf-feedstock#255](https://github.com/conda-forge/protobuf-feedstock/pull/255)
- Similarly to `onnx`, [grpc-cpp-feedstock#414](https://github.com/conda-forge/grpc-cpp-feedstock/pull/414) also needed the older `protobuf` version to be rebuilt first before it could move on. At the same time, we had to exclude some more end2end tests from CI as the runtime exceed our Azure timelimit.
- [kornia-rs-feedstock#22](https://github.com/conda-forge/kornia-rs-feedstock/pull/22) passed, but it had an explicit skip for `py>=313`. After removing this, we needed to set `maturin` into ABI3 compatability mode to produce a wheel for Python 3.14
- [conda-feedstock#274](https://github.com/conda-forge/conda-feedstock/pull/274) needed a rebase after a PR to main was merged, but it is still running into a cyclic dependency with `conda-libmamba-solver`
- Requested rerender for [python-rapidjon-feedstock#70](https://github.com/conda-forge/python-rapidjson-feedstock/pull/70) as build were failing but logs were no longer visible. Sadly the Python 3.14 builds here fail for Python 3.14 in relation to some refcount changes (similar to what was required to be fixed in `pyarrow`)
- [pyreadline-feedstock#17](https://github.com/conda-forge/pyreadline-feedstock/pull/17) required the addition of `setuptools` to `host` to become green
- In contrast to most `jaxlib` work, [jaxlib-feedstock#325](https://github.com/conda-forge/jaxlib-feedstock/pull/325) passed directly and we merged only building for Python 3.14 and reverted that with a `[ci skip]` on `main`.
- [python-duckdb-feedstock#130](https://github.com/conda-forge/python-duckdb-feedstock/pull/130) failed with deprecation warnings that triggered test failures. After updating to a new version in [python-duckdb-feedstock#129](https://github.com/conda-forge/python-duckdb-feedstock/pull/129), the previous PR also passed.
- We could close [gz-gui-feedstock#71](https://github.com/conda-forge/gz-gui-feedstock/pull/71) simply as this is an `abi3` package and doesn't need a rebuild.

Since the one week reaction period passed, I merged the following feedstock due to inactivity using my `conda-forge/core` powers:
- [fastdtw-feedstock#18](https://github.com/conda-forge/fastdtw-feedstock/pull/18)
- [tensorboard-data-server-feedstock#23](https://github.com/conda-forge/tensorboard-data-server-feedstock/pull/23)
- [xtensor-python-feedstock#98](https://github.com/conda-forge/xtensor-python-feedstock/pull/98)
- [setproctitle-feedstock#38](https://github.com/conda-forge/setproctitle-feedstock/pull/38)
- [gunicorn-feedstock#43](https://github.com/conda-forge/gunicorn-feedstock/pull/43)
- [python-maria-trie-feedstock#42](https://github.com/conda-forge/python-marisa-trie-feedstock/pull/42)
- [audioread-feedstock#33](https://github.com/conda-forge/audioread-feedstock/pull/33)
- [pycryptodome-feedstock#65](https://github.com/conda-forge/pycryptodome-feedstock/pull/65)
- [xrootd-feedstock#109](https://github.com/conda-forge/xrootd-feedstock/pull/109)

These feedstocks were merged because `@ocepaf` gave me the 👍 to merge green Python 3.14 PRs where he is a maintainer:
- [pyqtwebkit-feedstock#25](https://github.com/conda-forge/pyqtwebkit-feedstock/pull/25)
- [pyrsistent-feedstock#47](https://github.com/conda-forge/pyrsistent-feedstock/pull/47)
- [numexpr-feedstock#75](https://github.com/conda-forge/numexpr-feedstock/pull/75)

There were also several PRs where the CI passed, so that I pinged the maintainers to make them aware:
- [case_formats_io-feedstock#14](https://github.com/conda-forge/casa_formats_io-feedstock/pull/14)
- [doublemetaphone-feedstock#18](https://github.com/conda-forge/doublemetaphone-feedstock/pull/18)
- [openmm-feedstock#164](https://github.com/conda-forge/openmm-feedstock/pull/164)
- [mdtraj-feedstock#82](https://github.com/conda-forge/mdtraj-feedstock/pull/82)
- [wandb-feedstock#163](https://github.com/conda-forge/wandb-feedstock/pull/163)
- [dbus-python-feedstock#27](https://github.com/conda-forge/dbus-python-feedstock/pull/27)
- [cppe-feedstock#28](https://github.com/conda-forge/cppe-feedstock/pull/28)
- [kahip-feedstock#25](https://github.com/conda-forge/kahip-feedstock/pull/25)
- [scikit-sparse-feedstock#53](https://github.com/conda-forge/scikit-sparse-feedstock/pull/53)
- [simpleitk-feedstock#46](https://github.com/conda-forge/simpleitk-feedstock/pull/46)
- [smqtk-dataprovider-feedstock#4](https://github.com/conda-forge/smqtk-dataprovider-feedstock/pull/4)
- [traits-feedstock#46](https://github.com/conda-forge/traits-feedstock/pull/46)
- [pybtex-docutils-feedstock#22](https://github.com/conda-forge/pybtex-docutils-feedstock/pull/22)
- [pycocotools-feedstock#43](https://github.com/conda-forge/pycocotools-feedstock/pull/43)
- [pygraphviz-feedstock#47](https://github.com/conda-forge/pygraphviz-feedstock/pull/47)
- [psyplot-feedstock#33](https://github.com/conda-forge/psyplot-feedstock/pull/33)
- [assist-feedstock#18](https://github.com/conda-forge/assist-feedstock/pull/18)
- [cmarkgfm-feedstock#36](https://github.com/conda-forge/cmarkgfm-feedstock/pull/36)
- [partsegcore-compiled-backend-feedstock#28](https://github.com/conda-forge/partsegcore-compiled-backend-feedstock/pull/28)
- [memory-allocator-feedstock#20](https://github.com/conda-forge/memory-allocator-feedstock/pull/20)
- [pemja-feedstock#20](https://github.com/conda-forge/pemja-feedstock/pull/20)
- [mmh3-feedstock#27](https://github.com/conda-forge/mmh3-feedstock/pull/27)
- [njoy2016-feedstock#20](https://github.com/conda-forge/njoy2016-feedstock/pull/20)
- [opentsne-feedstock#54](https://github.com/conda-forge/opentsne-feedstock/pull/54)
- [opentracing-feedstock#20](https://github.com/conda-forge/opentracing-feedstock/pull/20)
- [promise-feedstock#26](https://github.com/conda-forge/promise-feedstock/pull/26)
- [pycbf-feedstock#18](https://github.com/conda-forge/pycbf-feedstock/pull/18)
- [pylsqpack-feedstock#9](https://github.com/conda-forge/pylsqpack-feedstock/pull/9)
- [pymdi-feedstock#87](https://github.com/conda-forge/pymdi-feedstock/pull/87)
- [python-box-feedstock#57](https://github.com/conda-forge/python-box-feedstock/pull/57)
- [pyvinecopulib-feedstock#13](https://github.com/conda-forge/pyvinecopulib-feedstock/pull/13)
- [qutip-feedstock#109](https://github.com/conda-forge/qutip-feedstock/pull/109)
- [rpm-tools-feedstock#45](https://github.com/conda-forge/rpm-tools-feedstock/pull/45)

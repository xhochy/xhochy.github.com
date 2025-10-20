---
layout: post
title: >
  2025-41: A week in conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

This week was the Python 3.14 release, and it will also be the final week where I log my conda-forge work. While it is interesting for me to see what I do work, it is also additional work that keeps me from contributing to conda-forge itself. Still, as a follow-up, I will write at least one more post that reviews the kind of work I have done in the last weeks and how that could be made more efficient. 

It started this time with a simple merge of updates for pre-commit hooks in [conda-forge-repodata-patches-feedstock#1091](https://github.com/conda-forge/conda-forge-repodata-patches-feedstock/pull/1091). [go-feedstock#300](https://github.com/conda-forge/go-feedstock/pull/300) and [onnx-feedstock#136](https://github.com/conda-forge/onnx-feedstock/pull/136) were similarly trivial merges. In addition, I merged [deepsearch-glm-feedstock#8](https://github.com/conda-forge/deepsearch-glm-feedstock/pull/8) and [lighttpd-feedstock#45](https://github.com/conda-forge/lighttpd-feedstock/pull/45) because the maintainers didn't react on a ping for over a week. A new release of `pdm` also required a rebase of the patch we keep in `conda-forge` in [pdm-feedstock#155](https://github.com/conda-forge/pdm-feedstock/pull/155). While I was at it, I took the opportunity and converted the feedstock to a v1 recipe.

A colleague of mine wanted to update the nodejs version for `code-server`, but [code-server-feedstock#134](https://github.com/conda-forge/code-server-feedstock/pull/134) failed as the feedstock was still using a manually pinned docker image. Later this week, we also needed to trigger a rerun of the CI jobs in [code-server-feedstock#135](https://github.com/conda-forge/code-server-feedstock/pull/135). This failed because of a virtual package issue in rattler-build for cross-compiled packaged. This was fixed in the meantime using `rattler-build 0.48.1`, and thus the rerun succeeded.

In another "support case" for my colleagues, I added  `types-aioboto3` to conda-forge via [staged-recipes#31182](https://github.com/conda-forge/staged-recipes/pull/31182). Some other typing dependencies were outdated and thus, I have converted the recipe to the v1 format and added myself as a maintainer in [types-s3transfer-feedstock#11](https://github.com/conda-forge/types-s3transfer-feedstock/pull/11) and [types-aiobotocore-feedstock#13](https://github.com/conda-forge/types-aiobotocore-feedstock/pull/13). Once these were merged, I backfilled the feedstocks with the releases that meanwhile happend and activated automerge on them.

This week a lot of new releases came in that needed some work. There was a new DuckDB release. The Python part was dealt with in [python-duckdb-feedstock#131](https://github.com/conda-forge/python-duckdb-feedstock/pull/131) whereas for the C++ part, we first finished the work on getting 1.4.0 built in [duckdb-split-feedstock#42](https://github.com/conda-forge/duckdb-split-feedstock/pull/42) before we rebased the PR for 1.4.1 in [duckdb-split-feedstock#43](https://github.com/conda-forge/duckdb-split-feedstock/pull/43). This week was also the release of LLVM 21.1.3. I'm trying to also look a bit after those as I'm not so experienced with them yet, but appreciate the existence of a well-maintained compiler toolchain. I set up the tracking issue in [clang-compiler-activation-feedstock#172](https://github.com/conda-forge/clang-compiler-activation-feedstock/pull/172) and merged [llvmdev-feedstock#349](https://github.com/conda-forge/llvmdev-feedstock/pull/349). I couldn't continue on my own, though, as all the outstanding PRs were dependent on [libcxx-feedstock#250](https://github.com/conda-forge/libcxx-feedstock/pull/250) to be merged. I'm not a maintainer on that feedstock, though.

Not only was Python 3.14.0 released this week, but there was also a long list of maintenance releases for the older Python versions. I opened PRs in the python-feedstock for [3.13.8](https://github.com/conda-forge/python-feedstock/pull/819), [3.12.12](https://github.com/conda-forge/python-feedstock/pull/815), [3.11.14](https://github.com/conda-forge/python-feedstock/pull/816), and [3.10.19](https://github.com/conda-forge/python-feedstock/pull/817).

Lately, there have been numerous `mark as broken` PRs from [sagemaker-code-editor-feedstock](https://github.com/conda-forge/sagemaker-code-editor-feedstock), e.g. [admin-requests#1701](https://github.com/conda-forge/admin-requests/pull/1701). This led me to review the feedstock and discover serious issues, which I summarised in [sagemaker-code-editor-feedstock#193](https://github.com/conda-forge/sagemaker-code-editor-feedstock/issues/193). Hopefully, they will address these soon so that the packages there have the usual code quality.
## Python 3.14
With the Python 3.14 release coming up this week, I did a last push to get some packages built before the "finish line".  The most major one here was that we manually marked `pyarrow` as done so that the migrator progresses to its dependencies in [conda-forge-pinning-feedstock#7830](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7830).

To document the progress we made this time with Python 3.14 support on release day, I wrote a blog post in [conda-forge.github.io#2618](https://github.com/conda-forge/conda-forge.github.io/pull/2618) that we published on Thursday, two days after the release. As with all rushed things, there were some cosmetic changes afterwards. We did add [a link to migration status early in the text](https://github.com/conda-forge/conda-forge.github.io/pull/2620) and [the link to the release announcement was missing](https://github.com/conda-forge/conda-forge.github.io/pull/2621).

I spent some time debugging the `uwsgi` builds in [uwsgi-feedstock#99](https://github.com/conda-forge/uwsgi-feedstock/pull/99), but gave up because I seemed to be doing something wrong there. At least, I got `jaxlib`-dependent packages unblocked by fixing [jax-feedstock#179](https://github.com/conda-forge/jax-feedstock/pull/179). [sqlalchemy-feedstock#191](https://github.com/conda-forge/sqlalchemy-feedstock/pull/191) could be merged as this is one of the feedstock, and I was allowed to merge the migration even though I'm not a maintainer. [py-rattler-build-feedstock#7](https://github.com/conda-forge/py-rattler-build-feedstock/pull/7) could be closed as it is an ABI3 package and doesn't need a rebuild.

Pinged the maintainers on [fisher-feedstock#33](https://github.com/conda-forge/fisher-feedstock/pull/33) and [fastbencode-feedstock#9](https://github.com/conda-forge/fastbencode-feedstock/pull/9). Also, this week, I was polishing the reports from weeks before. This meant that I also came across some PRs where the maintainers did not react for over a week. I have merged these accordingly. 
- [numcodecs-feedstock#119](https://github.com/conda-forge/numcodecs-feedstock/pull/119)
- [mariadb-feedstock#19](https://github.com/conda-forge/mariadb-feedstock/pull/19)
- [thrift-feedstock#42](https://github.com/conda-forge/thrift-feedstock/pull/42) and then rebased [thrift-feedstock#43](https://github.com/conda-forge/thrift-feedstock/pull/43)
- [fastbpe-feedstock#18](https://github.com/conda-forge/fastbpe-feedstock/pull/18)
- [fastpath-feedstock#13](https://github.com/conda-forge/fastpath-feedstock/pull/13)
- [future_fstrings-feedstock#21](https://github.com/conda-forge/future_fstrings-feedstock/pull/21)
- [cassandra-feedstock#26](https://github.com/conda-forge/cassandra-feedstock/pull/26)
- [cyvlfeat-feedstock#22](https://github.com/conda-forge/cyvlfeat-feedstock/pull/22)
- [encor-feedstock#7](https://github.com/conda-forge/encor-feedstock/pull/7)
- [ezdxf-feedstock#80](https://github.com/conda-forge/ezdxf-feedstock/pull/80)
- [focal-stats-feedstock#14](https://github.com/conda-forge/focal-stats-feedstock/pull/14)
- [fortls-feedstock#41](https://github.com/conda-forge/fortls-feedstock/pull/41)
- [pyreadline-feedstock#17](https://github.com/conda-forge/pyreadline-feedstock/pull/17)
- [doublemetaphone-feedstock#18](https://github.com/conda-forge/doublemetaphone-feedstock/pull/18)

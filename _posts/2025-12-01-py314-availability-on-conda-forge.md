---
layout: post
title: >
  Python 3.14 Availability on conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

As I was intrigued by [`@hugovk`'s BlueSky post on usable Python 3.14 packages on PyPI](https://bsky.app/profile/hugovk.dev/post/3m3376x6egc2r), I wanted to see how the situation is on conda-forge.
Here, we have a different setup where central tooling pushes even more towards availability.

While for wheel builds `cibuildwheel` will already nudge the majority of upstream maintainers to build wheels before the release,
on conda-forge, we have an even more intense nudge by the `regro-cf-autotick-bot` opening PRs `Rebuild for Python 3.14` on packages where all dependencies already support the new release.
This makes it even more noticeable for maintainers that they can build their package for Python 3.14.

In contrast to PyPI, the people responsible for the Python 3.14 builds are different.
In the PyPI case, the workload is on the upstream maintainers of a package to add Python 3.14 builds.
In conda-forge, you have feedstock maintainers (who sometimes include upstream maintainers) and eager contributors who would like to get a working Python 3.14 environment as soon as possible.
If people are looking at more than a single artefact, they will gain a better understanding of how to resolve failing builds for a new Python version.

## When do packages get "available" for a new Python?
As outlined in the [conda-forge blog post about the Python 3.14 release](https://conda-forge.org/blog/2025/10/09/python-314/), builds for the new Python version began once the first release candidate was available.
These packages are not yet immediately available to the end-user as the `python` package itself is not released to the main channel, but rather in the label `conda-forge/label/python_rc`.
By putting the Python release (actually, we let the package depend on a metapackage in that channel) behind a label, we can ensure that end-users don't accidentally get a Python 3.14 environment.
Once the main Python release has happened, all packages built for Python 3.14 become immediately available, as there is now a `python` package without a dependency on this magic label.

Packages that are designated as `noarch: python` or those built as ABI3 packages on conda-forge don't require any modifications or rebuilds to become usable.
There, the old releases are already suited for Python 3.14 and can be installed once a `python` build is available.
For a new Python version, we only need to rebuild packages that have a dependency on the minor version of Python, so-called arch-specific packages.

## Availability in the first 100days since a release
To assess how quickly Python packages become available in conda-forge, we want to examine the speed at which we have built binary packages in the first 100 days since the final Python release.
We will also investigate how many packages were actually built before the final release, so that we can determine whether we can leverage the availability of release candidates.

If we have a look at binary Python packages that were built once a `python` build was available for the currently (conda-forge) supported Python versions, we get the following plot:

![Python 3.14 binary package availability](/images/py314/package_availability_combined_100days.png)

If we compare the Python 3.14 line to the rest, we see that growth began much earlier than for the other Python versions and that, at its current state, it has also progressed much further before the 100-day mark than any other Python release has yet achieved.
There are also three pivotal points you can see:

1. About 40 days before the final release mark, the migration really started. Some crucial packages were finally migrated to Python 3.14, allowing us to migrate a significant number of packages.
2. About five days before the final release, `pandas` had been merged, and then a lot of other downstreams were available for migration.
3. About 25 days after the Python 3.14 final release. There have been days when I spent time reviewing numerous open PRs and merged many of them. These were ready and could be merged, but the maintainer didn't have the time, or the feedstock itself was stale, and it needed someone from core to actually review and merge them.

If you examine the Python versions over time, we can see that with new Python releases, we were able to utilise the availability of release candidates much better and could start building binary packages much earlier.
At the same time, we can also see that the later Python releases actually had fewer packages built at the 100-day mark.
This is not actually due to the community being less active, but there are two additional reasons that enabled us to migrate packages for a Python version early, without actually building them for the specific Python release.
We will examine these reasons in the next paragraphs.

## Migrating packages for new, unavailable Pythons
Building Python-version-specific binary packages requires us to have the Python minor release available;
however, there are also other mechanisms we can use to build a Python package that may contain binaries to ensure compatibility with future Python releases.
Many packages on conda-forge are actually accidentally Python version-specific, but can be converted with some trickery into `noarch: python packages`, which are then usable for future Python releases.
If we examine packages that were binary packages in Python 3.10 and later migrated to `noarch: python`, we can see that in later Python minor releases, packages were even faster available.

![Python 3.14 binary and noarch package availability](/images/py314/package_availability_with_noarch_combined_100days.png)

Marking a package as `noarch: python` works if the package doesn't contain any binaries.
If the package actually contains binaries that are specific to Python, there's still a trick that a lot of those can be ABI3, which means they're compatible in binary form with many Python minor releases, as long as they use the [limited ABI](https://docs.python.org/3/c-api/stable.html#limited-c-api).
Python 3.14 continues to be ABI3 compatible, and thus we can utilise all the packages that were built for ABI3 immediately with the availability of Python 3.14.

![Python 3.14 binary, ABI3 and noarch package availability](/images/py314/package_availability_with_noarch_abi3_combined_100days.png)

If we count in ABI3 packages and `noarch: python` packages that were once binary packages, we can see that the Python 3.14 release was the fastest ever in availability in conda-forge's history.
With these things in mind, we hope to have an even faster Python 3.15 availability.

Disclaimer: I mentioned a BlueSky post in the beginning. This has only inspired me, but sadly, the numbers here are not comparable to those because they look at the general marking of packages being available as Python 3.14. Whereas we're looking in this post about the availability of binary packages for Python 3.14. In the BlueSky post, packages marked as `noarch: python` wouldn't be counted towards availability, but only those where you explicitly add Python 3.14 support.

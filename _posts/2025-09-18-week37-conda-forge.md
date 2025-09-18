---
layout: post
title: >
  2025-37: A week in conda-forge
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---


While I spent the majority of my time on QuantCo-internal (often strategic, non-code) work, my [GitHub profile](https://github.com/xhochy) still hovers around 8,000 yearly contributions.
Part of this is through my internal work, but the majority of the counts are coming from work in conda-forge.
This is a place where I feel I have gamed the metric.
As many people will tell you, the number of PRs, commits, etc. doesn't tell you much about a developer's actual pace.

To give an insight into how this count is achieved, I have documented a week's worth of conda-forge interactions in this blog post.
As the time I can allocate varies drastically per week, I will try to write several reports in the future to gauge my activity a bit.
By writing these reports, I also hope to gain a better understanding of my contributions and how I can make my work a bit more efficient.

The week started off with me seeing that the [`aws-c-* Sep'25` Migration](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7749) I created at the end of last week wasn't auto-merging.
This is a trivial update, and we normally auto-merge all bot PRs on the `aws-c-*` packages.
Thus, I retroactively enabled `automerge` with [conda-forge-pinning-feedstock#7749](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7749).
Still, I had to manually merge [s2n-feedstock#158](https://github.com/conda-forge/s2n-feedstock/pull/158), [aws-c-event-stream-feedstock#220](https://github.com/conda-forge/aws-c-event-stream-feedstock/pull/220), [aws-c-http-feedstock#268](https://github.com/conda-forge/aws-c-http-feedstock/pull/268) and [aws-c-io-feedstock#374](https://github.com/conda-forge/aws-c-io-feedstock/pull/374) as these were already issued. 
Later this week, `aws-c-io 0.21.5` was automatically built and the migration was issued at [conda-forge-pinning-feedstock#7754](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7754).
As we want to `automerge` all `aws-c-*` migrations, I have added that to the migration file.
The migration then finished in [conda-forge-pinning-feedstock#7767](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7767).
And then another started for `aws-c-io 0.22.0`: [conda-forge-pinning-feedstock#7766](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7766).
As I (again) forgot `automerge: true`, I retroactively added this to [conda-forge-pinning-feedstock#7768](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7768). This migration was then closed with [conda-forge-pinning-feedstock#7774](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7774).
Meanwhile, `aws-crt-cpp` also got two new releases, where I [closed the first migrator](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7763) and only started [the second one](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7770#event-19649908088).

While I was on summer vacation, there was also a new major Go release: [go v1.25.1](https://github.com/conda-forge/go-feedstock/pull/298).
This brought a new set of breakages with it.
For example, last week, I added a skip to the `autocgo` test to check whether CGO was disabled once there was no compiler on the path anymore.
As we bake in the full compiler path, this is expected to fail.
Furthermore, Go now requires [macOS 12.x as the baseline](https://github.com/conda-forge/go-feedstock/pull/298/commits/1b43de948a0a8b1d5352c571ec6a7d98cfa4c293).
This required some trial and error to get the `osx-64` pipeline running with the newer SDK.
Once the main PR in `go-feedstock` is merged, we need to wait some hours (roughly three) so that the new go builds are available and we can update/merge [go-activation v1.25.1](https://github.com/conda-forge/go-activation-feedstock/pull/80).

Last week was also the release of [`glib 2.86.0`](https://github.com/conda-forge/glib-feedstock/pull/222).
This seems to be built fine except for one test ([gio/tests/socket-listner.c](https://gitlab.gnome.org/GNOME/glib/-/blob/main/gio/tests/socket-listener.c)) that was added in `2.85.x`.
As only even-numbered releases in the Gnome world are stable releases, this did not show up yet in conda-forge, as we don't build the odd-numbered, unstable releases.
The fix was adding an explicit linkage to `libdl`, and I have created an upstream report at https://gitlab.gnome.org/GNOME/glib/-/issues/3780.
At first, I was unable to create a merge request because my account wasn't allowed to fork something, but the developers in the issue explained to me that I needed to add an SSH key.
Once that was added (and some time passed), I opened https://gitlab.gnome.org/GNOME/glib/-/merge_requests/4799, which was quickly merged.

The [`mlx 0.29.0`](https://github.com/conda-forge/mlx-feedstock/pull/88) build was broken as it required a newer version of `nanobind`. I took the chance and also converted the feedstock to a v1 recipe and loosened the `nanobind` pin. I also had to restart the CI on [`mlx-lm v0.27.1`](https://github.com/conda-forge/mlx-lm-feedstock/pull/84) to get the build passing once the new `mlx` version was available.

[`nomad v1.10.5`](https://github.com/conda-forge/nomad-feedstock/pull/49#event-19623482700) failed to build with an error that other packages also report with Go 1.25; thus, I manually downgraded the Go version to 1.24.
Still, this version was quite picky about having the latest version of Go 1.24, namely 1.24.7.
Thus, I have submitted [go-feedstock#299](https://github.com/conda-forge/go-feedstock/pull/299) and [go-activation-feedstock#81](https://github.com/conda-forge/go-activation-feedstock/pull/81) to enable that version.
Furthermore, to fix the build, it was necessary to move the JavaScript part from `yarn` to `npm`, and we needed to update the `go-license` command in the recipe as the licenses of some new dependencies couldn't be automatically extracted.

On [`staged-recipes`](https://github.com/conda-forge/staged-recipes), I reviewed a colleague's submissions ([staged-recipes#30994](https://github.com/conda-forge/staged-recipes/pull/30994), [staged-recipes#30993](https://github.com/conda-forge/staged-recipes/pull/30993)), and I have contributed some simple recipes to help other colleagues: [`types-Flask-cors`](https://github.com/conda-forge/staged-recipes/pull/31015), [`pydups-stubs`](https://github.com/conda-forge/staged-recipes/pull/31014), and [`elevenlabs`](https://github.com/conda-forge/staged-recipes/pull/31013).
This week, there was sadly no time to review any community contributions.

This week was also a bit of time to look into the [`jaxlib v0.7.1`](https://github.com/conda-forge/jaxlib-feedstock/pull/321) PR.
The first hurdle here was that it needed a rebase of the patches on top of main again, and then the XLA patches needed to be rebased on top of the new XLA commit that the newer `jaxlib` is built upon.
Thereafter, compilation could start but errored out by not finding the `@proto_bazel_features` dependency.
As it turned out, this is an aliased version of the `bazel_features` package explicitly for `protobuf`.
Adding as a new dependency in XLA's `workspace3.bzl` gets the build a bit further, but I immediately ran into the problem that the `rules_java` version that was resolved by putting the latest release of `rules_java 7.x` at the top of `workspace3.bzl`.
Additionally, some more `abseil` headers had to be added to their system lib definition, and the build progressed into the compilation stage.
As part of the release, it also seems that they changed their version number calculation, and we need to add `--bazel_options=--repo_env=ML_WHEEL_TYPE=release` to mark this as a release build.
Sadly, this was not sufficient to build the whole release, as it seems that we now need to use `clang` on all platforms.
Thus, work on this PR will continue in the next week.

### Reviews

Someone else fixed the [`termgraph-feedstock`](https://github.com/conda-forge/termgraph-feedstock/pull/19) , where I'm the maintainer, and converted it to a v1 recipe.
I'm very happy that someone is taking over some of my maintenance.
There was also a PR on [`acme-feedstock`](https://github.com/conda-forge/acme-feedstock/pull/44) waiting for over a week, where I used my `@conda-forge/core` powers and merged it.

## Python 3.14

I could also spend some small cycles on the Python 3.14 migration.
To start the one-week core grace period for merges on unmaintained feedstocks, I pinged the maintainers on [tensorboard-data-server-feedstock#23](https://github.com/conda-forge/tensorboard-data-server-feedstock/pull/23), [https://github.com/conda-forge/watchfiles-feedstock/pull/37](https://github.com/conda-forge/watchfiles-feedstock/pull/37), and [https://github.com/conda-forge/portalocker-feedstock/pull/82](https://github.com/conda-forge/portalocker-feedstock/pull/82).
For two of those, the pings led to merges from the maintainer.
For the other one, I will probably merge myself.
On [https://github.com/conda-forge/openpyxl-feedstock/pull/61](https://github.com/conda-forge/openpyxl-feedstock/pull/61), I merged myself immediately as `@ocepaf`, as he has given us permission to merge the Python 3.14 PRs on the feedstock he maintains.

*Title picture: Photo by [Hannah Gibbs](https://unsplash.com/photos/man-pounding-hammer-on-hot-iron-rod-BINLgyrG_fI) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*


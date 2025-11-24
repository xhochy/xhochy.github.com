---
layout: post
title: >
  Learnings from five weeks of conda-forge logging
feature_image: "/images/hannah-gibbs-BINLgyrG_fI-unsplash.jpg"
---

During the last five weeks 
([2025-37](https://uwekorn.com/2025/09/18/week37-conda-forge.html),
[2025-38](https://uwekorn.com/2025/10/12/week38-conda-forge.html),
[2025-39](https://uwekorn.com/2025/10/14/week39-conda-forge.html),
[2025-40](https://uwekorn.com/2025/10/16/week40-conda-forge.html),
and [2025-41](https://uwekorn.com/2025/10/20/week41-conda-forge.html))
I logged my conda-forge activity as a blog post.
Instead of only reporting on it, I also wanted to have a look at what I have been doing there and whether things could be automated more.

One of the major points of my conda-forge output is the green contribution graph I have on GitHub.
This is partly nice to look at, as it has impressive numbers, but at the same time, it is also a clear sign of missing automation.
If I can manually make so many contributions, there must be a lot of them, despite a very small cognitive load.
These ones are most likely to be better automated than I spending my time on them.

One of those examples where a lot of manual work is done are the migrations for the `aws-c-*` stack.
These are the C packages that make up the `aws-cpp-sdk`.
With the current automation, we can start migrations quite easily, which automerge for a single package.
But often, the migration makes only sense if we update multiple of them at once.
Otherwise, the migration of a single `aws-c-*` package will get stuck if another is not updated simultaneously.
This typically resulted in some manual work on my part in the past. Additionally, the number of PRs when rebuilding for single packages is producing quite a bit of noise. 

To streamline the `aws-c-*` migrations, I now have a script [`make_aws_migration.py`](https://github.com/xhochy/cf-tooling/blob/eaaaf8f27bcbe578ff4102faa253b6b6429ad81b/make_aws_migration.py) that checks which AWS packages can be upgraded in the stack and submits a PR that does exactly that. To minimise the noise this generates, I will manually run this once or twice a month. You can see an example of this in [Migrate for aws-c-* Nov'11](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/7919).

The Go compiler on conda-forge was another source that included unnecessary manual work on my part. If patches applied successfully, it is fine to [`automerge` the PRs there](https://github.com/conda-forge/go-feedstock/commit/495eafc983e9a119c8cddf67d0b2a51cd8c4887e). PRs on `go-activation` also needed to be rerun once the `go` builds were done. Instead, we can use [`check_solvable: true`](https://github.com/conda-forge/go-activation-feedstock/commit/41a1a8c4ed7ba61bda89a0076e818f0cb5464aca) to only issue the version update PR once the underlying `go` builds are available.

Another typical maintenance task on Go and on NodeJS is that we not only want updates for the latest release, but also for older release versions. As the [conda-forge automation](https://github.com/regro/cf-scripts) only supports updating the latest release on a feedstock, this has been a manual task. In the long term, it would be beneficial if the automation also updated versions on maintenance branches. As an intermediate step, I wrote (partially AI-coded) scripts ([one for Go](https://github.com/xhochy/cf-tooling/blob/3244f93c5522e6f53c25ceb8675967b0325c9044/update_go_releases.py) and [one for Node.js](https://github.com/xhochy/cf-tooling/blob/3244f93c5522e6f53c25ceb8675967b0325c9044/update_nodejs_releases.py)) that check for version updates and issue pull requests. This could also be used for Python, but here, many more maintainers are actively subscribed to the release announcements, and thus, my scripts won't be much faster/simpler here.

Overall, there seems to be no further patterns that I could automate. There are many small breakages that a contributor's (and sometimes really "my") attention is required. This includes things like packages that are already on `automerge` but fail due to an unexpected change. Here, my contributions won't be overlooked, but I often find myself spending a significant amount of time scrolling and clicking until I locate the actual error in the log. This is something to work on in the future.


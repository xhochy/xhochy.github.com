---
layout: post
title: The first two weeks with the Apple M1
feature_image: "/images/natalie-grainger-7atL8cXOvXs-unsplash.jpg"
---

Apple recently published new computers that contain their new M1 processors. I was quite excited about them because of the promises made by various benchmarks regarding performance and energy consumption but also because it is also a new platform. Most things won't work there and some assumption on how we work today have to change if you want to use them.

I have now spent about two weeks working with the M1 and finally made the switch to it being my primary work device. The initial motivation to get one was actually to have it before other colleagues get one so that I could ease out the pains of working on a new platform of a vendor of whom we already bought quite some devices from. At this point I'm certain my less low-level-savvy colleagues will also be very productive on this machine. I cannot say that they will be more productive than on x86-powered machines but it will come near to it at least.

Personally, I really like the device because it isn't a small evolution like the typical 10+x% of better CPU performance you would get every year but it really is notably faster without the massive battery life cost one would expect. Additionally, it doesn't seem to get hot, you can now put your laptop on your lap while compiling on all cores without having the feeling a firestorm is happening above your knees.

## (Python) Dependencies are missing (not that much)

With a new platform, we face the issue that every piece of software needs to be ported to it. Luckily, the M1 is not the first CPU with an ARM64 instruction set, it only is the first CPU in a laptop that also runs macOS. This means that a lot of software either has been ported to an iPhone or to Linux-based servers running on ARM64. While these are not the same of macOS-on-ARM64, it relieves a bit of the pain.

### Homebrew 

Still, all these dependencies I need to work on my projects need to be build. For the system-wide packages, I'm using [homebrew](https://brew.sh/) as the package and for the environments I work in, I use `conda` as the package manager with all public dependencies coming from [conda-forge](https://conda-forge.org/).

At the time of writing, the official Homebrew installation script was not supporting the installation on ARM64 but recommended to use Rosetta 2. This is probably for the next months the easiest path to choose if you don't want to be bothered with compile-issues on the new platform. I want to be bothered though, so I installed it from the tarball. If you want to do the same, [Sam Soffes has a nice short post explaining how to get two homebrews, one for x86, one for ARM64 installed on the same machine](https://soffes.blog/homebrew-on-apple-silicon).

For the packages I have been installing from Homebrew, most of the things could be compiled from source and then worked flawlessly. There is [a Github issue that covers a long list about what does work on the M1](https://github.com/Homebrew/brew/issues/7857). The only time I needed to do a workaround was when I was installing Go. There I needed to build from HEAD using [a formula from an open PR](https://github.com/Homebrew/homebrew-core/pull/66166) as at that time, there was no M1 compatible `gobootstrap` available and it needed to be run using Rosetta 2. Now, [as mentioned at the end of the issue thread](https://github.com/Homebrew/homebrew-core/pull/66166#issuecomment-748163588) you can use the beta of Go 1.16 to build it completely natively on the M1.

With that nearly everything I use in my day-to-day setup installed via Homebrew works except my favourite text editor `neovim`. The [upstream issue](https://github.com/neovim/neovim/pull/12624) is closed but I still did not manage to build it. I'll have a look at that some time in the future as currently using old-school `vim` suffices for me (for now).

### conda(-forge)

The other, for me more important, package manager that needed porting to ARM64-based macOS is `conda`. I'm using this to install packages in my per-project-based environments for C++/Python/R code. Here a lot of work has been done porting packages to there called `osx-arm64` infrastructure using cross-compiling. Isuru Fernando wrote [a blog post about how packages are built there on the conda-forge blog](https://conda-forge.org/blog/posts/2020-10-29-macos-arm64/). This meant that a lot of packages were already built before the first devices were shipped.

A lot of packages doesn't still cover though what I need in my daily work. Thus using the M1 for actively working on things meant that I discovered a lot of packages that needed to be rebuilt for the new architecture. Through several PRs to https://github.com/conda-forge/conda-forge-pinning-feedstock/blob/master/recipe/migrations/osx_arm64.txt I instructed the `conda-forge` bots to issue PRs to these packages and their dependencies. For a first round this included:

* `pre-commit` for running pre-commit hooks
* `conda-smithy` to work on recipes for conda-forge, esp. run `conda smithy rerender` locally
* `boa` to use `mamba` inside of `conda-build` using `conda mambabuild recipe`
* `asv` to run benchmarks
* `poetry` which also needed [a backport of an older keyring](https://github.com/conda-forge/keyring-feedstock/pull/53)
* `rust_osx-arm64` / `rust-activation` to natively build Rust packages on the M1, before that only cross-compilation worked
* `grayskull` to add new recipes to `conda-forge`
* `cross-python` to debug cross-compiling Python packages. Here I used the reverse direction we use in CI: I compiled from osx-arm64 for osx64 (Intel)
* Various other utils:
    * `python-xxhash`
    * `uritools`
    * `pytest-cov`
    * `simplejson`
    * `file`
    * `perl`
    * `black`

One thing that needed a bit more love was `numba` as it doesn't yet support the latest release of LLVM, version 11. We did workaround that by also building [LLVM 10 for `osx-arm64`](https://github.com/conda-forge/llvmdev-feedstock/pull/111). Once that was built, we faced the issue that the `llvmlite` was built using handcrafted (i.e. not generated) Makefiles. These did not support cross-compiling. Instead of adding cross-compiling support to them, we provided [a patch to llvmlite to use CMake on all systems](https://github.com/numba/llvmlite/pull/677). This was already used for Windows and neatly supports cross-compiling out of the box. Numba itself then build fine for `osx-arm64` but the test suite was failing with [hashing issues](https://github.com/numba/numba/issues/6589#issuecomment-748595174) and [build failures due to a wrong architecture](https://github.com/numba/numba/issues/6589#issuecomment-748595315). These issues were not directly in Numba but in the conda-forge build for Python. We fixed the hashing issue by [providing Python's `config.site` the correct values](https://github.com/conda-forge/python-feedstock/pull/433) to let it select the default/preferred hash implementation. The build issues were fixed by [removing the hard-coded x86_64 values in the sysconfigdata](https://github.com/conda-forge/python-feedstock/pull/434). After updating to these new Python builds, the tests ran fine.

### Docker

One of the main "pain points" that people will have with the new M1 machines will be that they won't be able to run x86 Docker images with the same speed as on their x86-based laptops. This is something that most like won't change in the future and needs an adoption to the workflow of people.

When my MacBook arrived, there was no Docker support available at all. This is the worst possible situation for everyone. Still, this would have been manageable with some tweaks to the setup. While working on getting things running natively [Docker released a technical preview for the M1](https://www.docker.com/blog/download-and-try-the-tech-preview-of-docker-desktop-for-m1/).
With this technical preview you can work on the M1 actually as on the Intel MacBooks.

Using Docker on the ARM64 device has one notable difference, ARM64 Linux containers run as fast as stuff does on macOS whereas x86_64 containers run through emulation roughly 10x slower. On an Intel device this is the same, just with the architecture switched.

While currently most production system today run on x86 machines like the developers of these systems work on, this will change in future. We see more and more ARM64 server being used as well as there are a lot of predictions that we will see RISC-V as a competing architecture for x86 and ARM64 popping up in the future. With this upcoming heterogeneity, it will be unlikely that your development machine is the same as all your production servers. Thus facing the issue that your standard Docker containers will not by default run at full speed is an inevitable thing that will come up into future anyways.

To circumvent that you do have to run your production containers with the emulation slowdown, there are two popular approaches.

1. You can build your containers for multiple architectures and always use the one that matches the native architecture of your machines. If all your dependencies are available for all your used architectures, you should be able to get near identical containers with native performance and the availability to use the same tooling / scripts / setup as before. Here you though need to trust your tests in your CI pipeline to detect architecture specific differences / issues before you deploy to production.
2. In the case where you need to develop on a specific architecture (e.g. because you want to tweak some assembly or have a failure on one of the arches), you can use Docker's `-H/--host` option. With that you can work with the local docker client but execute the container itself on a remote server with a different architecture. This is also the way I use to work in Windows Docker containers from a MacBook (you need a Windows host for Windows containers).


## Conclusion & Next Steps

Taking two weeks to switch over to it being my primary work laptop is much faster than expected. This happily contrasts with my initial fears that it would take months to make people productive on it and I can sleep better now knowing that if some random colleague buys such a machine, I can get them up to speed in a matter of hours over the phone.

Part of this serenity comes from the fact that Rosetta 2 is doing a great job if you want to run software that hasn't been ported, the other part comes from the good preparation of the communities around the package managers using their available infrastructure like access to the development toolkit (DTK) machines as well as being able to cross-compile on CI systems from Intel macOS for ARM64 macOS.

As the next step, I will report on what it takes/took to get Apache Arrow and its dependencies smoothly running on the M1. This is the part I'm most excited about as here I'm familiar with nearly the whole stack and thus am also quite happy to dive deep into problems.

*Title picture: Photo by [Natalie Grainger](https://unsplash.com/@missnjc?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*


---
layout: post
title: Deploying conda environments in (Docker) containers - how to do it right
feature_image: "/images/jonas-smith-aL6tG-j-E4Y-unsplash.jpg"
---

Deploying conda environments inside a container looks like a straight-forward `conda install`. But with a bit more love for details, you can optimise the process so that the build is faster and the resulting container much smaller.

For optimising conda-environments-in-docker [Jim Crist-Harif's "Smaller Docker images with Conda"](https://jcristharif.com/conda-docker-tips.html) blog post has been the go-to resource for a long time. It is still as valid nowadays as it was back then. Due to changes in the [default content of conda-forge packages](https://github.com/conda-forge/cfep/blob/master/cfep-18.md) and [specific optimisation](https://uwekorn.com/2020/09/08/trimming-down-pyarrow-conda-1-of-x.html) of [some particular heavy packages](https://uwekorn.com/2020/10/28/trimming-down-pyarrow-conda-2-of-x.html), it will though yield fewer savings nowadays.

Over the course of time, we have also developed new techniques that help trim down the sizes of Docker containers even further. In this article, I though want to start at the beginning again and have a look at the most obvious way one would put a conda environment into a Docker container. We will look at all the flaws this approach brings with it and optimise the workflow bit-for-bit until we reach a near-perfect container.

## Starting with just a few lines

There are several simple but not so effective approaches to put conda environments into a container. I have chosen a particularly bad one here to explain some more things. While I have seen this approach in the wild, most of the ones I have seen were already using the basic things I will improve in the first sections.

As an example, I have built a really [simple machine learning model using `scikit-learn` and `LightGBM` that predicts a fare price for a taxi trip in New York City](https://github.com/xhochy/nyc-taxi-fare-prediction-deployment-example). The model is served via FastAPI and is already using the best practices like `uvicorn` there.

In our initial setting, we have checked out the `git` repository and are running all commands from the repository root.

```dockerfile
FROM continuumio/anaconda3:2020.11

COPY . .
RUN conda env create
RUN conda run -n nyc-taxi-fare-prediction-deployment-example \
  python -m pip install --no-deps -e .
CMD [ \
  "conda", "run", "-n", "nyc-taxi-fare-prediction-deployment-example", \
  "gunicorn", "-k", "uvicorn.workers.UvicornWorker", "nyc_taxi_fare.serve:app" \
]
```

Building this image using `make anaconda` in the repository yields a container with the whopping size of 4.48GB.

```
% docker image ls nyc-taxi-anaconda
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
nyc-taxi-anaconda   latest    ebf2b267e4d8   30 seconds ago   4.48GB
```

The first thing we look at is **"Could you make it even worse?"**. A typical issue I see here is that people develop on their local machine without keeping track of their environment specification and simply export their list of installed packages using `conda env export`. I have posted [a gist of my local environment](https://gist.github.com/xhochy/ebe4100d6f30e591c4c0b1a6359e2457) but the important bits why this is problematic can be seen in:

```yaml
name: nyc-taxi-fare-prediction-deployment-example
channels:
  - conda-forge
dependencies:
  …
  - libcurl=7.71.1=h9bf37e3_8
  - libcxx=11.0.1=habf9029_0
  - libedit=3.1.20191231=hed1e85f_2
  …
```

It suffices actually to only look at the `libcxx=11.0.1=habf9029_0` line. This shows two issues that come with a local export on an operating system than your deployment operating system. The file was generated on macOS while the Docker container is a Linux system. The last bit of the version `habf9029_0` is the build number/string/hash. This is different for different builds but the same version of a package. Most importantly it is different between macOS and Linux when you have compiled code inside the package. Thus `conda` won't be able to find a package with this build string for Linux. We could avoid this issue by using `--no-builds` on the export but will directly face the next issue that native dependencies are different between operating systems. In this case, `libcxx` is the C++ library used for builds done with the `clang` compiler. This is only used on macOS in `conda-forge` and thus not necessary for Linux environments. While in this case, it can be installed on Linux but sometimes you also have packages that are only available on one OS and then `conda` wouldn't be able to find an installable version if after you have removed the build string.

To avoid these issues without keeping track of what you have installed, you can ask `conda` with `conda env export --from-history` to export an environment file with only the package specifications you have explicitly requested on the command line. This tiny bit of RTFM is [my most popular tweet to date](https://twitter.com/xhochy/status/1199405251280936960). In this case, we would get the committed `environment.yml`.

# Use a smaller base image

After having built a base image and already outlined another mistake you can run into, let's improve the image a bit. For that we look at the individual sizes of the container layers:

```
% docker history nyc-taxi-anaconda
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
ebf2b267e4d8   18 minutes ago   /bin/sh -c #(nop)  CMD ["conda" "run" "-n" "…   0B
a113c9d31b1a   18 minutes ago   /bin/sh -c conda run -n nyc-taxi-fare-predic…   1.05MB
04c85327c969   18 minutes ago   /bin/sh -c conda env create                     1.5GB
a749aca38f88   22 minutes ago   /bin/sh -c #(nop) COPY dir:461eb7a23d1121123…   274MB
5e5dd010ead8   2 months ago     /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      2 months ago     /bin/sh -c wget --quiet https://repo.anacond…   2.43GB
<missing>      2 months ago     /bin/sh -c apt-get update --fix-missing &&  …   211MB
<missing>      2 months ago     /bin/sh -c #(nop)  ENV PATH=/opt/conda/bin:/…   0B
<missing>      2 months ago     /bin/sh -c #(nop)  ENV LANG=C.UTF-8 LC_ALL=C…   0B
<missing>      2 months ago     /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      2 months ago     /bin/sh -c #(nop) ADD file:d2abb0e4e7ac17737…   69.2MB
```

We see several large layers here. The ones created `2 months ago` are part of the `continuumio/anaconda3` base image. This base image alone is 2.71GB in size. This is due to the fact that it already brings the whole `anaconda` environment. This contains a vast set of scientific Python packages in it.

```
% docker image ls continuumio/anaconda3
REPOSITORY              TAG       IMAGE ID       CREATED        SIZE
continuumio/anaconda3   2020.11   5e5dd010ead8   2 months ago   2.71GB
```

Instead, we only need an image that contains an environment that can execute `conda` and the other basic things we need in a container build. For that you have the following three options:

 * `continuumio/miniconda3`: Debian Buster with a minimal conda installation configured for the Anaconda `defaults` channel (`continuumio/miniconda3:4.9.2` is 437MB in uncompressed size)
 * `conda-forge/mambaforge3`: Ubuntu 20.04 with a minimal conda and mamba installation configured with `conda-forge` as the default package source (`conda-forge/mambaforge:4.9.2-5` is 411MB in uncompressed size)
 * `conda-forge/mambaforge-pypy3`: Ubuntu 20.04 with a minimal conda and mamba installation configured with `conda-forge` as the default package source and using PyPy instead of CPython (`conda-forge/mambaforge-pypy3:4.9.2-5` is 449MB in uncompressed size)

We will continue here using `conda-forge/mambaforge:4.9.2-5` and we will also use `mamba` instead of `conda` to create the environment as this is simply faster. This already brings our final image down to `2.13GB`. Looking at the `docker history` output, we see that the environment creation layer is by far the largest one:

```
% docker history nyc-taxi-mambaforge                                                                                            :(
IMAGE          CREATED         CREATED BY                                      SIZE      COMMENT
e823c8c825c7   2 minutes ago   /bin/sh -c #(nop)  CMD ["conda" "run" "-n" "…   0B
61e5273a176f   2 minutes ago   /bin/sh -c conda run -n nyc-taxi-fare-predic…   1.04MB
5e1acb2d4a11   2 minutes ago   /bin/sh -c mamba env create                     1.45GB
07e8fc9e0576   4 minutes ago   /bin/sh -c #(nop) COPY dir:32d87108966cf6795…   274MB
73fe058d8ee5   2 hours ago     CMD ["/bin/bash"]                               0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ENTRYPOINT ["tini" "--"]                        0B        buildkit.dockerfile.v0
<missing>      2 hours ago     RUN |3 MINIFORGE_NAME=Mambaforge MINIFORGE_V…   338MB     buildkit.dockerfile.v0
<missing>      2 hours ago     ENV PATH=/opt/conda/bin:/usr/local/sbin:/usr…   0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ENV LANG=C.UTF-8 LC_ALL=C.UTF-8                 0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ENV CONDA_DIR=/opt/conda                        0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ARG TINI_VERSION=v0.18.0                        0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ARG MINIFORGE_VERSION=4.9.2-5                   0B        buildkit.dockerfile.v0
<missing>      2 hours ago     ARG MINIFORGE_NAME=Miniforge3                   0B        buildkit.dockerfile.v0
<missing>      3 weeks ago     /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      3 weeks ago     /bin/sh -c mkdir -p /run/systemd && echo 'do…   7B
<missing>      3 weeks ago     /bin/sh -c [ -z "$(apt-get indextargets)" ]     0B
<missing>      3 weeks ago     /bin/sh -c set -xe   && echo '#!/bin/sh' > /…   811B
<missing>      3 weeks ago     /bin/sh -c #(nop) ADD file:2a90223d9f00d31e3…   72.9MB
```

## Removing unnecessary files

In the above layers, we can see that we copy 274MB of local context into the container. Instead, we only want to have the code to run the model inside the container. Thus we should exclude larger files that are not needed by the `.dockerignore`. With the example project, we ignore the `data/` folder that contains 258MB training data and `.mypy_cache` that is also 3MB.

Additionally, we currently download the conda packages during the `env create` step and then extract them into the environment. The package tarballs are though saved in conda's cache and thus also in the image. This is something we don't want and thus we should also remove them using `conda clean -afy`. 

With the above two changes we are now at an overall size of 1.31GB for the image.

## Only install packages used for predict

The `environment.yml` we use to build the Docker container is the specification of our development environment and contains a vast set of packages that we don't need during runtime. It would be nice if we could mark some of those as not-necessary-for-prod but [the functionality is not yet available](https://github.com/conda-incubator/conda-lock/issues/76).

Instead, there two approaches you can take:

1. Build a conda package out of your project. The run(time) dependencies of the conda package is only what you need at prediction time.
2. Manually specify your runtime environment in a separate `predict-environment.yml`.

Personally, I prefer to use option 1 as this enables one to use the package also in various other conda environments. Though I can understand to have a container that directly embeds the code instead of going through the indirection of a package build.

Thus we create a `predict-environment.yml` with just the things we need for serving:

```yaml
name: nyc-taxi-fare-prediction-deployment-example
channels:
  - conda-forge
  - nodefaults
dependencies:
  - pip
  - click
  - fastapi
  - gunicorn
  - lightgbm
  - pandas
  - scikit-learn
  - setuptools_scm
  - uvicorn
```

Instead of using that as an input for the container build, we are using [conda-lock](https://github.com/conda-incubator/conda-lock) to render the requirements into locked pinnings for different architectures. This enables us to have a consistent environment to make developer as well as production environments reproducible. Another benefit of using `conda-lock` is that you can generate the lockfile for any platform from any other platform. This means we can develop on macOS and have production systems on Linux but still generate the lockfiles for all of them on either systems. We can do this using the following commands:

```
conda lock -p osx-64 -p osx-arm64 -p linux-64 -p linux-aarch64 -f environment.yml
conda lock -p osx-64 -p osx-arm64 -p linux-64 -p linux-aarch64 -f predict-environment.yml --filename-template 'predict-{platform}.lock'
```

I'm pinning here for Intel and Arm64 architectures on macOS and Linux at the same time as these are the typical environments I run my code in. If you don't have any macOS or ARM64 systems, feel free to omit those.


In the `Dockerfile`, we now load the lock file first, install the dependencies and add the application code only afterwards. This enables us to cache the environment and not rebuild it on every source code change.

```
COPY predict-linux-64.lock .
RUN mamba create --name nyc-taxi-fare-prediction-deployment-example --file predict-linux-64.lock && \ 
    conda clean -afy
```

With the now reduced set of dependencies, we now get the overall container down to `851MB` in size where the conda environment with `438MB` accounts for roughly half the size of the container. The remaining share of the docker container is the base conda and the base Ubuntu installation.

## Distroless container

As conda environments are not only Python environments but actually include all dependencies of any language, they need neither the Ubuntu distribution nor the conda installation to run the code. The only outside dependency is the `libc`.

Thus we only need to have a docker container that ships with a `libc` (and its dependencies) and nothing else. Such containers are provided by the https://github.com/GoogleContainerTools/distroless project. To build an image using these containers, we are going to use multi-stage builds. First, we build the environment in a `mambaforge` container and then copy it into the `distroless` one:

```dockerfile
# Container for building the environment
FROM condaforge/mambaforge:4.9.2-5 as conda

COPY predict-linux-64.lock .
RUN mamba create --copy -p /env --file predict-linux-64.lock && conda clean -afy
COPY . /pkg
RUN conda run -p /env python -m pip install --no-deps /pkg

# Distroless for execution
FROM gcr.io/distroless/base-debian10

COPY --from=conda /env /env
COPY model.pkl .
CMD [ \
  "/env/bin/gunicorn", "-b", "0.0.0.0:8000", "-k", "uvicorn.workers.UvicornWorker", "nyc_taxi_fare.serve:app" \
]
```

With that, we can bring the total container size to 457 MiB. Of the total size, the conda environment is making up 90%. Thus we finally reach rock bottom in what we can optimise in the size of the container and the size of the environment given the current minimal set of dependencies.

```
% docker image ls nyc-taxi-distroless
REPOSITORY            TAG       IMAGE ID       CREATED        SIZE
nyc-taxi-distroless   latest    a08a43bbb9a7   21 hours ago   457MB
% docker image history nyc-taxi-distroless
IMAGE          CREATED        CREATED BY                                      SIZE      COMMENT
a08a43bbb9a7   21 hours ago   /bin/sh -c #(nop)  CMD ["/env/bin/gunicorn" …   0B
35001d8906ca   21 hours ago   /bin/sh -c #(nop) COPY file:985b6980c4f46538…   326kB
f413381a803c   21 hours ago   /bin/sh -c #(nop) COPY dir:474655ed326942b52…   437MB
af31651e48fe   51 years ago   bazel build ...                                 17.4MB
<missing>      51 years ago   bazel build ...                                 1.82MB
```

## Using `--mount=type=cache` instead of `conda clean` with BuildKit

In the above approaches, we have always explicitly removed the conda cache after installing the environment. We also did download the packages fully on each non-cached build. In newer versions of docker you can use its BuildKit backend though which also supports mounting cache volumes during the build phase. For this, we need to adjust the `mamba create` line to:

```
RUN --mount=type=cache,target=/opt/conda/pkgs mamba create --copy -p /env --file predict-linux-64.lock
```

Additionally, we now call `docker buildx build` instead of the typical `docker build` to force it to use the BuildKit backend. This has the benefit that we don't need to clean the cache explicitly anymore but also that build time drop on my machine from 56s down to 23s if the packages were already downloaded during a previous container build.

## (Expert) Deleting unnecessary files from the environment

While we have now trimmed down our container to the minimal overhead and trimmed the package list of the conda environment down to the minimal set, we still are at 457MiB in size. This is much more than we actually need to run the web app. When you really know what you are doing, you can now look into the internals of the conda environment and delete files that aren't needed for the specific app to trim down the image size.

For doing this, we run the last single-stage image that was mentioned in the article using `docker run -ti nyc-taxi-mambaforge-predict-only-deps /bin/bash` and use `apt update && apt install -y ncdu` to install `ncdu` as a command line UI to inspect the conda environment in detail. Using `du -sbh nyc-taxi-fare-prediction-deployment-example` we a starting size of 422MiB.

Here we can split the list of tasks into two categories: Deleting files that you can get rid of because you know *exactly* that you don't need them in this case and secondly deleting files that shouldn't be part of the conda package at all.

With the knowledge that we are running a LightGBM model inside a FastAPI app inside of `gunicorn`, we can get rid of the following things and saves storage space:

* Delete the conda metadata in `conda-meta`; saves 5MiB.
* Delete C/C++ includes in `include` (only required at compile-time); saves 5MiB.
* Delete `libpython3.9.so.1.0`; this works as `gunicorn` is started using the `python` executable and this is statically lined, i.e. contains the same symbols as `libpython.so`. This wouldn't work if we had an application that would embed the Python interpreter itself. Saves 20.5MiB.
* Delete `__pycache__` with `find -name '__pycache__' -type d -exec rm -rf '{}' '+'`; This saves 54MiB at the cost of a bit of startup time.
* Delete `pip` with `rm -rf /env/lib/python3.9/site-packages/pip` as we don't plan to install any packages; saves 5MiB.
* Delete `lib/python3.9/i{dlelib, ensurepip}` from the standard library, saves 4MiB.
* Delete `lib{a,t,l,u}san.so` as the [various sanitizers](https://clang.llvm.org/docs/index.html) are not used during production; saves 23MiB.
* Delete the binaries `x86_64-conda-linux-gnu-ld, sqlite3, openssl` from `bin`. These are the largest binaries in `bin` and none of them is used during runtime; saves 5MiB.
* Delete `share/terminfo` as we don't expect to run a terminal;  saves 2MiB.

In addition, the following removals also brought us a bit of space. These removals are not specific to our usage of the packages but should in general not be part of the main conda package of these artefacts.

* Delete static libraries with `find -name '*.a' -delete`. These are not needed at runtime, only during compile-time. Bit-by-bit we remove them from the main conda packages as suggested by [CFEP-18](https://github.com/conda-forge/cfep/blob/master/cfep-18.md). This saves 3 MiB.
* Delete `scipy`'s tests, open issue at [scipy-feedstock#160](https://github.com/conda-forge/scipy-feedstock/issues/160); saves 11MB
* Delete `pandas`' tests; saves 10MB
* Delete `numpy`'s tests; saves 4MB
* Delete site-packages/uvloop/loop.c, 7MB


Overall, with all these changes, we can bring the total container size down to 296MiB.

## Conclusion

Starting from a 4.48GiB image with a lot of beginner mistakes, we brought the Docker container for the FastAPI+gunicorn+LightGBM+scikit-learn setup down to 296MiB. It is probably possible to strip this even further down but less than 300MiB sounds like an acceptable size for such a service.

We have settled on the following, multi-stage, BuildKit-caching distroless Dockerfile:


```dockerfile
# Container for building the environment
FROM condaforge/mambaforge:4.9.2-5 as conda

COPY predict-linux-64.lock .
RUN --mount=type=cache,target=/opt/conda/pkgs mamba create --copy -p /env --file predict-linux-64.lock && echo 4
COPY . /pkg
RUN conda run -p /env python -m pip install --no-deps /pkg
# Clean in a separate layer as calling conda still generates some __pycache__ files
RUN find -name '*.a' -delete && \
  rm -rf /env/conda-meta && \
  rm -rf /env/include && \
  rm /env/lib/libpython3.9.so.1.0 && \
  find -name '__pycache__' -type d -exec rm -rf '{}' '+' && \
  rm -rf /env/lib/python3.9/site-packages/pip /env/lib/python3.9/idlelib /env/lib/python3.9/ensurepip \
    /env/lib/libasan.so.5.0.0 \
    /env/lib/libtsan.so.0.0.0 \
    /env/lib/liblsan.so.0.0.0 \
    /env/lib/libubsan.so.1.0.0 \
    /env/bin/x86_64-conda-linux-gnu-ld \
    /env/bin/sqlite3 \
    /env/bin/openssl \
    /env/share/terminfo && \
  find /env/lib/python3.9/site-packages/scipy -name 'tests' -type d -exec rm -rf '{}' '+' && \
  find /env/lib/python3.9/site-packages/numpy -name 'tests' -type d -exec rm -rf '{}' '+' && \
  find /env/lib/python3.9/site-packages/pandas -name 'tests' -type d -exec rm -rf '{}' '+' && \
  find /env/lib/python3.9/site-packages -name '*.pyx' -delete && \
  rm -rf /env/lib/python3.9/site-packages/uvloop/loop.c

# Distroless for execution
FROM gcr.io/distroless/base-debian10

COPY --from=conda /env /env
COPY model.pkl .
CMD [ \
  "/env/bin/gunicorn", "-b", "0.0.0.0:8000", "-k", "uvicorn.workers.UvicornWorker", "nyc_taxi_fare.serve:app" \
]
```

This container can be build and run using the following two commands:

```
docker buildx build -t nyc-taxi-distroless-buildkit-expert --load -f docker/Dockerfile.distroless-buildkit-expert .
docker run -ti -p 8000:8000 nyc-taxi-distroless-buildkit-expert
```

As a follow-up, I'll write a second article that is more in a cheatsheet version that you can refer as a general reminder on how to containerise conda environments.

Another follow-up could be a comparison of the container we have built here with the various versions that are offered by `@tiangolo`'s [uvicorn-gunicorn-fastapi-docker](https://github.com/tiangolo/uvicorn-gunicorn-fastapi-docker). We will definitely see differences in size but it would also be interesting on how they compete in build time and runtime performance.

Given that I look at a lot of problems with Data Science / Machine Learning in mind, we could also have a look on how to reduce the runtime dependencies by using runtimes like [ONNX](https://onnx.ai/) or [`treelite`](https://treelite.readthedocs.io/en/latest/).

## Improving the Stats Quo

It would be easy but also a bit boring to just report on the status quo. Thus during the writing of the blog post, I made the following pull requests to improve the status quo a bit:

 * [Add version tags and make miniforge images multi-arch](https://github.com/conda-forge/miniforge-images/pull/2)
 * [Move miniforge image builds to Azure Pipelines as conda-forge has a bigger allowance there](https://github.com/conda-forge/miniforge-images/pull/3)
 * [Ensure a pinned mamba version in miniforge](https://github.com/conda-forge/miniforge/pull/110)
 * [Request `linux-aarch64` build of `lightgbm` and `pre-commit` packages](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/1236)
 * [Fix `linux-aarch64` builds in the `lightgbm` feedstock](https://github.com/conda-forge/lightgbm-feedstock/pull/32)

*Title picture: Photo by [Jonas Smith](https://unsplash.com/@jonassmith?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*


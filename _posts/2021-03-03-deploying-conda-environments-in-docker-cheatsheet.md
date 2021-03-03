---
layout: post
title: Deploying conda environments in (Docker) containers - The Cheatsheet!
feature_image: "/images/jonas-smith-aL6tG-j-E4Y-unsplash.jpg"
---

Deploying conda environments inside a container looks like a straight-forward `conda install`. But with a bit more love for details, you can optimise the process so that the build is faster and the resulting container much smaller.

This post intends to be an addition to a [Jim Crist-Harif's "Smaller Docker images with Conda"](https://jcristharif.com/conda-docker-tips.html) and a condensed version of the [long-read I've published at the same time as this post](https://uwekorn.com/2021/03/01/deploying-conda-environments-in-docker-how-to-do-it-right.html). If you want to understand the reasoning of the tips here, you should head over to the long-read. Otherwise, I hope that this post serves as a cheatsheet to lookup every time you want to put a conda environment inside a (Docker) container.

# Lock your environment

The dependencies of your to-be-containerised environment should be specified as you typically specify a conda environment using an `environment.yml` file. An example looks like:

```yaml
name: production-service
channels:
  - conda-forge
  - nodefaults
dependencies:
  - click
  - fastapi
  - gunicorn
  - lightgbm
  - pandas
  - scikit-learn
  - setuptools_scm
  - uvicorn
```

While this is a nice human-readable file, it doesn't provide you with reproducibility across runs. Thus we pin the requirements down to the actual build of each package using [`conda-lock`](https://github.com/conda-incubator/conda-lock). This has the advantage that with the given lockfile, you can recreate the exact environment every time. Also, the lockfiles switch `conda` into a special mode where it only installs packages but doesn't do a new solving step. This dramatically improves installation time.

To generate a lock file from an `environment.yml` you can use the following command:

```
conda lock -p linux-64 -f environment.yml
```

And install it using:

```
conda create --name production-service --file conda-linux-64.lock
```

# Use a small base container to build the environment

As the base of your images, you should pick one the following three containers that come with a basic / minimal `conda` installation. In the case of using one of the conda-forge provided images, we always pick the ones that come with `mamba` included as this speeds up the installation dramatically.

* `continuumio/miniconda3`: Debian Buster with a minimal conda installation configured for the Anaconda `defaults` channel (`continuumio/miniconda3:4.9.2` is 437MB in uncompressed size)
 * `conda-forge/mambaforge3`: Ubuntu 20.04 with a minimal conda and mamba installation configured with `conda-forge` as the default package source (`conda-forge/mambaforge:4.9.2-5` is 411MB in uncompressed size)
 * `conda-forge/mambaforge-pypy3`: Ubuntu 20.04 with a minimal conda and mamba installation configured with `conda-forge` as the default package source and using PyPy instead of CPython (`conda-forge/mambaforge-pypy3:4.9.2-5` is 449MB in uncompressed size)

# Deploy using distroless containers

If you do not plan to use your container interactively but only want to package a service inside of it, you should use multi-stage builds. In the first stage, you should take one of the above containers and build the conda environment in it. As the second stage, you should pick the most minimal container that is required to run the conda environment namely `gcr.io/distroless/base-debian10`. This container only contains the bare essential like a `libc` but no shell or package manager at all.

A `Dockerfile` for this multi-stage approach looks like the following:

```dockerfile
# Container for building the environment
FROM condaforge/mambaforge:4.9.2-5 as conda

COPY conda-linux-64.lock .
## Use BuildKit's caching mounts to speed up builds.
##
## If you don't use BuildKit,
## add `&& conda clean -afy` at the end of this line.
RUN --mount=type=cache,target=/opt/conda/pkgs mamba create --copy -p /env --file conda-linux-64.lock
## Now add any local files from your repository.
## As an example, we add a Python package into
## the environment.
COPY . /pkg
RUN conda run -p /env python -m pip install --no-deps /pkg

# Distroless for execution
FROM gcr.io/distroless/base-debian10

## Copy over the conda environment from the previous stage.
## This must be located at the same location.
COPY --from=conda /env /env
## The command needs to be in […] notation as
## the distroless container doesn't contain
## a shell.
CMD [ \
  "/env/bin/gunicorn", "-b", "0.0.0.0:8000", "-k", "uvicorn.workers.UvicornWorker", "nyc_taxi_fare.serve:app" \
]
```

# (experienced) Remove files unnecessary at runtime

This picks up the ideas of [Jim's post](https://jcristharif.com/conda-docker-tips.html). While conda environments come with all batteries included, batteries weigh quite a bit and you don't need all of them at runtime. Thus you should get rid of those that you know that you will not need.

* Delete the conda metadata in `conda-meta`
* Delete C/C++ includes in `include` (only required at compile-time)
* Delete `libpython*.so.*`; this only works in the case where your entrypoint is calling the `python` executable which is statically linked, i.e. contains the same symbols as `libpython.so`. This wouldn't work if we had an application that would embed the Python interpreter itself.
* Delete `__pycache__` with `find -name '__pycache__' -type d -exec rm -rf '{}' '+'`
* Delete `pip` with `rm -rf /env/lib/python3.9/site-packages/pip` as we don't plan to install any packages
* Delete `lib/python3.9/i{dlelib, ensurepip}` from the standard library
* Delete `lib{a,t,l,u}san.so` as the [various sanitizers](https://clang.llvm.org/docs/index.html) are not used during production
* Delete the binaries like `x86_64-conda-linux-gnu-ld, sqlite3, openssl` from `bin`. In most cases, these binaries are not used for running a service
* Delete `share/terminfo` as we don't expect to run a terminal

The above tips boil down to the following statement. Be aware though that not every file listed here can be safely removed in all cases.

```
  find -name '*.a' -delete && \
  find -name '__pycache__' -type d -exec rm -rf '{}' '+' && \
  rm -rf /env/conda-meta \
    /env/include \
    /env/lib/libpython*.so.* \
    /env/lib/python*/site-packages/pip \
    /env/lib/python*/idlelib \
    /env/lib/python*/ensurepip \
    /env/lib/libasan.so.* \
    /env/lib/libtsan.so.* \
    /env/lib/liblsan.so.* \
    /env/lib/libubsan.so.* \
    /env/bin/x86_64-conda-linux-gnu-ld \
    /env/bin/sqlite3 \
    /env/bin/openssl \
    /env/share/terminfo 
```

*Title picture: Photo by [Jonas Smith](https://unsplash.com/@jonassmith?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*


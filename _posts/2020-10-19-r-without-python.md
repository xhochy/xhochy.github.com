---
layout: post
title: "Removing Python as a dependency of R"
feature_image: "/images/boris-misevic-kjH2m_tl7Pg-unsplash.jpg"
---

*Surprisingly Python was a runtime dependency of R on conda-forge. As R doesn't need Python to run, this was a bit weird. We got rid of this by splitting up the GLib package.*

Typically people argue whether they should use R or Python to solve a problem.
In the case of conda-forge's `r-base` package, on Linux, you got both when you only asked for one as Python was listed as a runtime dependency of R.
As they are both independent languages one doesn't need the other to run, this was rather surprising.

Creating a new environment for R 4.0 using `conda create -n r-test -c conda-forge r-base=4.0` lead to the following solution at the beginning of September:

```
The following NEW packages will be INSTALLED:

  _libgcc_mutex      conda-forge/linux-64::_libgcc_mutex-0.1-conda_forge
  _openmp_mutex      conda-forge/linux-64::_openmp_mutex-4.5-1_gnu
  _r-mutex           conda-forge/noarch::_r-mutex-1.0.1-anacondar_1
  binutils_impl_lin~ conda-forge/linux-64::binutils_impl_linux-64-2.35-h18a2f87_9
  binutils_linux-64  conda-forge/linux-64::binutils_linux-64-2.35-heab0d09_28
  bwidget            conda-forge/linux-64::bwidget-1.9.14-0
  bzip2              conda-forge/linux-64::bzip2-1.0.8-h516909a_3
  c-ares             conda-forge/linux-64::c-ares-1.16.1-h516909a_3
  ca-certificates    conda-forge/linux-64::ca-certificates-2020.6.20-hecda079_0
  cairo              conda-forge/linux-64::cairo-1.16.0-h3fc0475_1005
  certifi            conda-forge/linux-64::certifi-2020.6.20-py38h32f6830_0
  curl               conda-forge/linux-64::curl-7.71.1-he644dc0_5
  fontconfig         conda-forge/linux-64::fontconfig-2.13.1-h1056068_1002
  freetype           conda-forge/linux-64::freetype-2.10.2-he06d7ca_0
  fribidi            conda-forge/linux-64::fribidi-1.0.10-h516909a_0
  gcc_impl_linux-64  conda-forge/linux-64::gcc_impl_linux-64-7.5.0-hdb87b24_16
  gcc_linux-64       conda-forge/linux-64::gcc_linux-64-7.5.0-hf34d7eb_28
  gettext            conda-forge/linux-64::gettext-0.19.8.1-hc5be6a0_1002
  gfortran_impl_lin~ conda-forge/linux-64::gfortran_impl_linux-64-7.5.0-h1104b78_16
  gfortran_linux-64  conda-forge/linux-64::gfortran_linux-64-7.5.0-ha781d05_28
  glib               conda-forge/linux-64::glib-2.66.0-h0dae87d_0
  graphite2          conda-forge/linux-64::graphite2-1.3.13-he1b5a44_1001
  gsl                conda-forge/linux-64::gsl-2.6-h294904e_0
  gxx_impl_linux-64  conda-forge/linux-64::gxx_impl_linux-64-7.5.0-h1104b78_16
  gxx_linux-64       conda-forge/linux-64::gxx_linux-64-7.5.0-ha781d05_28
  harfbuzz           conda-forge/linux-64::harfbuzz-2.7.2-hee91db6_0
  icu                conda-forge/linux-64::icu-67.1-he1b5a44_0
  jpeg               conda-forge/linux-64::jpeg-9d-h516909a_0
  kernel-headers_li~ conda-forge/noarch::kernel-headers_linux-64-2.6.32-h77966d4_13
  krb5               conda-forge/linux-64::krb5-1.17.1-hfafb76e_3
  ld_impl_linux-64   conda-forge/linux-64::ld_impl_linux-64-2.35-h769bd43_9
  libblas            conda-forge/linux-64::libblas-3.8.0-17_openblas
  libcblas           conda-forge/linux-64::libcblas-3.8.0-17_openblas
  libcurl            conda-forge/linux-64::libcurl-7.71.1-hcdd3856_5
  libedit            conda-forge/linux-64::libedit-3.1.20191231-he28a2e2_2
  libev              conda-forge/linux-64::libev-4.33-h516909a_1
  libffi             conda-forge/linux-64::libffi-3.2.1-he1b5a44_1007
  libgcc-devel_linu~ conda-forge/linux-64::libgcc-devel_linux-64-7.5.0-h42c25f5_16
  libgcc-ng          conda-forge/linux-64::libgcc-ng-9.3.0-h24d8f2e_16
  libgfortran-ng     conda-forge/linux-64::libgfortran-ng-7.5.0-hdf63c60_16
  libgomp            conda-forge/linux-64::libgomp-9.3.0-h24d8f2e_16
  libiconv           conda-forge/linux-64::libiconv-1.16-h516909a_0
  liblapack          conda-forge/linux-64::liblapack-3.8.0-17_openblas
  libnghttp2         conda-forge/linux-64::libnghttp2-1.41.0-h8cfc5f6_2
  libopenblas        conda-forge/linux-64::libopenblas-0.3.10-pthreads_hb3c22a3_4
  libpng             conda-forge/linux-64::libpng-1.6.37-hed695b0_2
  libssh2            conda-forge/linux-64::libssh2-1.9.0-hab1572f_5
  libstdcxx-devel_l~ conda-forge/linux-64::libstdcxx-devel_linux-64-7.5.0-h4084dd6_16
  libstdcxx-ng       conda-forge/linux-64::libstdcxx-ng-9.3.0-hdf63c60_16
  libtiff            conda-forge/linux-64::libtiff-4.1.0-hc7e4089_6
  libuuid            conda-forge/linux-64::libuuid-2.32.1-h14c3975_1000
  libwebp-base       conda-forge/linux-64::libwebp-base-1.1.0-h516909a_3
  libxcb             conda-forge/linux-64::libxcb-1.13-h14c3975_1002
  libxml2            conda-forge/linux-64::libxml2-2.9.10-h68273f3_2
  lz4-c              conda-forge/linux-64::lz4-c-1.9.2-he1b5a44_3
  make               conda-forge/linux-64::make-4.3-h516909a_0
  ncurses            conda-forge/linux-64::ncurses-6.2-he1b5a44_1
  openssl            conda-forge/linux-64::openssl-1.1.1g-h516909a_1
  pango              conda-forge/linux-64::pango-1.42.4-h7062337_4
  pcre               conda-forge/linux-64::pcre-8.44-he1b5a44_0
  pcre2              conda-forge/linux-64::pcre2-10.35-h2f06484_0
  pip                conda-forge/noarch::pip-20.2.3-py_0
  pixman             conda-forge/linux-64::pixman-0.38.0-h516909a_1003
  pthread-stubs      conda-forge/linux-64::pthread-stubs-0.4-h14c3975_1001
  python             conda-forge/linux-64::python-3.8.5-h1103e12_7_cpython
  python_abi         conda-forge/linux-64::python_abi-3.8-1_cp38
  r-base             conda-forge/linux-64::r-base-4.0.2-he766273_1
  readline           conda-forge/linux-64::readline-8.0-he28a2e2_2
  sed                conda-forge/linux-64::sed-4.8-hbfbb72e_0
  setuptools         conda-forge/linux-64::setuptools-49.6.0-py38h32f6830_0
  sqlite             conda-forge/linux-64::sqlite-3.33.0-h4cf870e_0
  sysroot_linux-64   conda-forge/noarch::sysroot_linux-64-2.12-h77966d4_13
  tk                 conda-forge/linux-64::tk-8.6.10-hed695b0_0
  tktable            conda-forge/linux-64::tktable-2.10-h555a92e_3
  wheel              conda-forge/noarch::wheel-0.35.1-pyh9f0ad1d_0
  xorg-kbproto       conda-forge/linux-64::xorg-kbproto-1.0.7-h14c3975_1002
  xorg-libice        conda-forge/linux-64::xorg-libice-1.0.10-h516909a_0
  xorg-libsm         conda-forge/linux-64::xorg-libsm-1.2.3-h84519dc_1000
  xorg-libx11        conda-forge/linux-64::xorg-libx11-1.6.12-h516909a_0
  xorg-libxau        conda-forge/linux-64::xorg-libxau-1.0.9-h14c3975_0
  xorg-libxdmcp      conda-forge/linux-64::xorg-libxdmcp-1.1.3-h516909a_0
  xorg-libxext       conda-forge/linux-64::xorg-libxext-1.3.4-h516909a_0
  xorg-libxrender    conda-forge/linux-64::xorg-libxrender-0.9.10-h516909a_1002
  xorg-renderproto   conda-forge/linux-64::xorg-renderproto-0.11.1-h14c3975_1002
  xorg-xextproto     conda-forge/linux-64::xorg-xextproto-7.3.0-h14c3975_1002
  xorg-xproto        conda-forge/linux-64::xorg-xproto-7.0.31-h14c3975_1007
  xz                 conda-forge/linux-64::xz-5.2.5-h516909a_1
  zlib               conda-forge/linux-64::zlib-1.2.11-h516909a_1009
  zstd               conda-forge/linux-64::zstd-1.4.5-h6597ccf_2
```


As one can spot, among several other dependencies, `python` and `pip` were listed. To check why `python` was installed, we use [conda-tree](https://github.com/rvalieris/conda-tree) to find the reverse dependencies:

```
# conda-tree whoneeds python
['certifi', 'python_abi', 'setuptools', 'pip', 'wheel', 'glib'
```

Except `glib`, all of these packages are actually cyclic dependencies of `python`:

```
# conda-tree cycles
python_abi -> python -> pip -> setuptools -> python_abi
python_abi -> python -> pip -> setuptools -> certifi -> python_abi
certifi -> python -> pip -> setuptools -> certifi
wheel -> python -> pip -> wheel
setuptools -> python -> pip -> setuptools
python -> pip -> python
```

Thus the issue boils down to certain parts of `glib` needing `python` [as declared in the meta.yaml](https://github.com/conda-forge/glib-feedstock/blob/30e093ce18cdcea181c4f62d4c566d2025233059/recipe/meta.yaml#L39-L40).

```yaml
# Note that Python is in fact a runtime dependency because this package
# provides a couple of developer support tools that are Python scripts.

requirements:
  build:
    - …
    - python >=2.7
  host:
    - …
  run:
    - python >=2.7
    - …
```

`python` is a runtime dependency of `glib` as some of its building tooling is based on Python scripts.
We don't need that build tooling when we only want to run code that depends on `glib`, thus we create a new package called `libglib` which only contains the runtime dependencies and components.
The main package that is listed in the `host` requirements is still `glib` as we need these Python-depending build tools in most cases.
The main trick is that `glib` has a {% raw %}`run_exports: - {{ pin_subpackage("libglib") }}`{% endraw %}, meaning that packages built with `glib` in `host` will automatically depend on `libglib` at runtime.
To make this work, we have [rebuilt the latest version, 2.66.1](https://github.com/conda-forge/glib-feedstock/pull/57) and [2.64.6](https://github.com/conda-forge/glib-feedstock/pull/68), the last stable version before 2.66 and also the one that still works with 10.9 macOS SDK.

Another important aspect of `2.64.6` is that we haven't built this patch version before.
This means that we use safely use it in [a migration](https://github.com/conda-forge/conda-forge-pinning-feedstock/pull/873) to rebuild all packages depending on `glib` so that the `libglib` `run_exports` is applied correctly.
If we were using a version of `glib` where we already had built a version without the package split, we wouldn't be 100% sure in all cases that the variant with the split packages would be selected.

While the new recipes and the new pinning should ensure that the correct `run_exports` is set, we sadly still have `glib` mentioned explicitly in many of the recipes.
To ensure that on an `r-base` installation only `libglib` and not `glib` and `python` are installed, we need to remove `glib` from the `run` sections of the packages that `r-base` pulls in.
This is in addition to `r-base` also [cairo](https://github.com/conda-forge/cairo-feedstock/pull/55), [pango](https://github.com/conda-forge/pango-feedstock/pull/34), [harfbuzz](https://github.com/conda-forge/harfbuzz-feedstock/pull/56).

Finally, we also run the `glib` migration for r-base: https://github.com/conda-forge/r-base-feedstock/pull/140.
Here we can already see as part of the CI builds that `conda-build` doesn't install `python` or `glib` in the test environment.

Creating a new environment for R 4.0 using `conda create -n r-test -c conda-forge r-base=4.0` leads to the following solution as of 18th October:

```
  _libgcc_mutex      conda-forge/linux-64::_libgcc_mutex-0.1-conda_forge
  _openmp_mutex      conda-forge/linux-64::_openmp_mutex-4.5-1_gnu
  _r-mutex           conda-forge/noarch::_r-mutex-1.0.1-anacondar_1
  binutils_impl_lin~ conda-forge/linux-64::binutils_impl_linux-64-2.35-h18a2f87_9
  binutils_linux-64  conda-forge/linux-64::binutils_linux-64-2.35-hc3fd857_29
  bwidget            conda-forge/linux-64::bwidget-1.9.14-0
  bzip2              conda-forge/linux-64::bzip2-1.0.8-h516909a_3
  c-ares             conda-forge/linux-64::c-ares-1.16.1-h516909a_3
  ca-certificates    conda-forge/linux-64::ca-certificates-2020.6.20-hecda079_0
  cairo              conda-forge/linux-64::cairo-1.16.0-h488836b_1006
  curl               conda-forge/linux-64::curl-7.71.1-he644dc0_8
  fontconfig         conda-forge/linux-64::fontconfig-2.13.1-h1056068_1002
  freetype           conda-forge/linux-64::freetype-2.10.3-he06d7ca_0
  fribidi            conda-forge/linux-64::fribidi-1.0.10-h516909a_0
  gcc_impl_linux-64  conda-forge/linux-64::gcc_impl_linux-64-7.5.0-hda68d29_13
  gcc_linux-64       conda-forge/linux-64::gcc_linux-64-7.5.0-he2a3fca_29
  gettext            conda-forge/linux-64::gettext-0.19.8.1-hf34092f_1003
  gfortran_impl_lin~ conda-forge/linux-64::gfortran_impl_linux-64-7.5.0-hfca37b7_17
  gfortran_linux-64  conda-forge/linux-64::gfortran_linux-64-7.5.0-ha081f1e_29
  graphite2          conda-forge/linux-64::graphite2-1.3.13-he1b5a44_1001
  gsl                conda-forge/linux-64::gsl-2.6-h294904e_0
  gxx_impl_linux-64  conda-forge/linux-64::gxx_impl_linux-64-7.5.0-h64c220c_13
  gxx_linux-64       conda-forge/linux-64::gxx_linux-64-7.5.0-h547f3ba_29
  harfbuzz           conda-forge/linux-64::harfbuzz-2.7.2-hb1ce69c_1
  icu                conda-forge/linux-64::icu-67.1-he1b5a44_0
  jpeg               conda-forge/linux-64::jpeg-9d-h516909a_0
  kernel-headers_li~ conda-forge/noarch::kernel-headers_linux-64-2.6.32-h77966d4_13
  krb5               conda-forge/linux-64::krb5-1.17.1-hfafb76e_3
  ld_impl_linux-64   conda-forge/linux-64::ld_impl_linux-64-2.35-h769bd43_9
  libblas            conda-forge/linux-64::libblas-3.8.0-17_openblas
  libcblas           conda-forge/linux-64::libcblas-3.8.0-17_openblas
  libcurl            conda-forge/linux-64::libcurl-7.71.1-hcdd3856_8
  libedit            conda-forge/linux-64::libedit-3.1.20191231-he28a2e2_2
  libev              conda-forge/linux-64::libev-4.33-h516909a_1
  libffi             conda-forge/linux-64::libffi-3.2.1-he1b5a44_1007
  libgcc-ng          conda-forge/linux-64::libgcc-ng-9.3.0-h5dbcf3e_17
  libgfortran-ng     conda-forge/linux-64::libgfortran-ng-7.5.0-hae1eefd_17
  libgfortran4       conda-forge/linux-64::libgfortran4-7.5.0-hae1eefd_17
  libglib            conda-forge/linux-64::libglib-2.66.1-h0dae87d_1
  libgomp            conda-forge/linux-64::libgomp-9.3.0-h5dbcf3e_17
  libiconv           conda-forge/linux-64::libiconv-1.16-h516909a_0
  liblapack          conda-forge/linux-64::liblapack-3.8.0-17_openblas
  libnghttp2         conda-forge/linux-64::libnghttp2-1.41.0-h8cfc5f6_2
  libopenblas        pkgs/main/linux-64::libopenblas-0.3.10-h5a2b251_0
  libpng             conda-forge/linux-64::libpng-1.6.37-hed695b0_2
  libssh2            conda-forge/linux-64::libssh2-1.9.0-hab1572f_5
  libstdcxx-ng       conda-forge/linux-64::libstdcxx-ng-9.3.0-h2ae2ef3_17
  libtiff            conda-forge/linux-64::libtiff-4.1.0-hc7e4089_6
  libuuid            conda-forge/linux-64::libuuid-2.32.1-h14c3975_1000
  libwebp-base       conda-forge/linux-64::libwebp-base-1.1.0-h516909a_3
  libxcb             conda-forge/linux-64::libxcb-1.13-h14c3975_1002
  libxml2            conda-forge/linux-64::libxml2-2.9.10-h68273f3_2
  lz4-c              conda-forge/linux-64::lz4-c-1.9.2-he1b5a44_3
  make               conda-forge/linux-64::make-4.3-hd18ef5c_1
  ncurses            conda-forge/linux-64::ncurses-6.2-he1b5a44_2
  openssl            conda-forge/linux-64::openssl-1.1.1h-h516909a_0
  pango              conda-forge/linux-64::pango-1.42.4-h80147aa_5
  pcre               conda-forge/linux-64::pcre-8.44-he1b5a44_0
  pcre2              conda-forge/linux-64::pcre2-10.35-h2f06484_0
  pixman             conda-forge/linux-64::pixman-0.38.0-h516909a_1003
  pthread-stubs      conda-forge/linux-64::pthread-stubs-0.4-h14c3975_1001
  r-base             conda-forge/linux-64::r-base-4.0.3-hd272fe0_2
  readline           conda-forge/linux-64::readline-8.0-he28a2e2_2
  sed                conda-forge/linux-64::sed-4.8-hbfbb72e_0
  sysroot_linux-64   conda-forge/noarch::sysroot_linux-64-2.12-h77966d4_13
  tk                 conda-forge/linux-64::tk-8.6.10-hed695b0_1
  tktable            conda-forge/linux-64::tktable-2.10-h555a92e_3
  xorg-kbproto       conda-forge/linux-64::xorg-kbproto-1.0.7-h14c3975_1002
  xorg-libice        conda-forge/linux-64::xorg-libice-1.0.10-h516909a_0
  xorg-libsm         conda-forge/linux-64::xorg-libsm-1.2.3-h84519dc_1000
  xorg-libx11        conda-forge/linux-64::xorg-libx11-1.6.12-h516909a_0
  xorg-libxau        conda-forge/linux-64::xorg-libxau-1.0.9-h14c3975_0
  xorg-libxdmcp      conda-forge/linux-64::xorg-libxdmcp-1.1.3-h516909a_0
  xorg-libxext       conda-forge/linux-64::xorg-libxext-1.3.4-h516909a_0
  xorg-libxrender    conda-forge/linux-64::xorg-libxrender-0.9.10-h516909a_1002
  xorg-renderproto   conda-forge/linux-64::xorg-renderproto-0.11.1-h14c3975_1002
  xorg-xextproto     conda-forge/linux-64::xorg-xextproto-7.3.0-h14c3975_1002
  xorg-xproto        conda-forge/linux-64::xorg-xproto-7.0.31-h14c3975_1007
  xz                 conda-forge/linux-64::xz-5.2.5-h516909a_1
  zlib               conda-forge/linux-64::zlib-1.2.11-h516909a_1010
  zstd               conda-forge/linux-64::zstd-1.4.5-h6597ccf_2
```

Besides still being quite some dependencies, this no longer includes `glib` and `python`, thus we have achieved the goal of this task.

*Title picture: Photo by [boris misevic](https://unsplash.com/@borisview?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*

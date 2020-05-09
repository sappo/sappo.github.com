---

title: Cross compiling autotools projects for the Raspberry Pi
date: 2016-10-13 20:09:05
tags: pi

---

There is a lot of information on the internet that explains how to cross compile
software for the Raspberry Pi. But this information is scattered into tiny
pieces and a lot of advice is just wrong or cumbersome.

In this blog post I will explain how to cross compile most autotools projects.

## Toolchain

At first we need to get a cross compile toolchain and a sysroot environment for
the Raspberry Pi. Luckily the Raspberry Pi Foundation does provide both on their
github profile. So we'll can just clone them to our computer.

```bash
$ git clone --depth 1 https://github.com/raspberrypi/tools rpi-tools
```

Note that I used the attribute `--depth 1` which tells git only to fetch data
from the latest commit. As the tools repository is rather large this minimizes
the amount of data we have to download.

We will use the `arm-linux-gnueabihf` cross compiler which is at the time of
this the latest and seems to support all Raspberry Pi devices. The next step is
to get a hold of tools the toolchain provides.

```bash
TOOLCHAIN_HOST="arm-linux-gnueabihf"
TOOLCHAIN_PATH="$PWD/rpi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/bin"
CPP="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-cpp"
CC="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-gcc"
CXX="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-g++"
LD="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-ld"
AS="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-as"
AR="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-ar"
RANLIB="${TOOLCHAIN_PATH}/${TOOLCHAIN_HOST}-ranlib"
```

There are more but for most projects those are sufficient. Further we need the
sysroot environment for this toolchain and add it to our compiler flags.

```bash
SYSROOT=$PWD/rpi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/arm-linux-gnueabihf/sysroot
CFLAGS+="--sysroot=${SYSROOT}"
CPPFLAGS+="--sysroot=${SYSROOT}"
CXXFLAGS+="--sysroot=${SYSROOT}"
```

## Cross Compiling

Any autotools project is configured with a configure script. Thus we need to
provide the configure script with the tools and flags we prepared.

```bash
CONFIG_OPTS=()
CONFIG_OPTS+=("CFLAGS=${CFLAGS}")
CONFIG_OPTS+=("CPPFLAGS=${CPPFLAGS}") CONFIG_OPTS+=("CXXFLAGS=${CXXFLAGS}")
CONFIG_OPTS+=("LDFLAGS=${LDFLAGS}") CONFIG_OPTS+=("PKG_CONFIG_DIR=")
CONFIG_OPTS+=("--host=${TOOLCHAIN_HOST}")
CONFIG_OPTS+=("CPP=${CPP}")
CONFIG_OPTS+=("CC=${CC}")
CONFIG_OPTS+=("CXX=${CXX}")
CONFIG_OPTS+=("LD=${LD}")
CONFIG_OPTS+=("AS=${AS}")
CONFIG_OPTS+=("AR=${AR}")
CONFIG_OPTS+=("RANLIB=${RANLIB}")
```

To get the cross compiled resources we need to install them to a location of our
choosing.

```bash
BUILD_PREFIX=$PWD/tmp
CONFIG_OPTS+=("--prefix=${BUILD_PREFIX}")
```

A lot of project use pkgconf to find dependencies. Therefore we tell autotools
where to find Raspberry Pi specific pkgconf configuartions.

```bash
CONFIG_OPTS+=("PKG_CONFIG_DIR=")
CONFIG_OPTS+=("PKG_CONFIG_LIBDIR=${SYSROOT}/usr/lib/arm-linux-gnueabihf/pkgconfig:${SYSROOT}/usr/share/pkgconfig")
CONFIG_OPTS+=("PKG_CONFIG_SYSROOT=${SYSROOT}")
```

We might have to cross compile dependencies ourselves. That's why we let pkgconf
look into our install directory for dependencies as well.

```bash
CONFIG_OPTS+=("PKG_CONFIG_PATH=${BUILD_PREFIX}/lib/pkgconfig")
```

## Building

Finally we can start building our project. In this example I will compile
ZeroMQ's `libzmq` and higher level binding `czmq` to demonstrate that self
compiled dependencies are detected as well. If you've used autotools before you
should be familar with the build steps. Only difference are the `CONFIG_OPTS` we
supply.

```bash
git clone --depth 1 https://github.com/zeromq/libzmq.git
pushd libzmq
(
    ./autogen.sh &&
    ./configure "${CONFIG_OPTS[@]}" &&
    make -j4 &&
    make install
) || exit 1
popd

git clone --depth 1 https://github.com/zeromq/czmq.git
pushd czmq
(
    ./autogen.sh &&
    ./configure "${CONFIG_OPTS[@]}" &&
    make -j4
    make install
) || exit 1
popd
```

## Update

The czmq library contains a cross-compile build script for the RPI that I
created after writing this blog post:

https://github.com/zeromq/czmq/tree/master/builds/rpi

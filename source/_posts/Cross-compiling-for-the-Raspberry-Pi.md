---
title: Cross compiling autotools projects for the Raspberry Pi
date: 2016-08-12 09:27:02
tags: pi, autotools
---

There is are a lot of information on the internet that explains how to cross
compile software for the Raspberry Pi. But this information is scattered into
tiny pieces and a lot of advice is just wrong or cumbersome.

In this blog I'll exlain to you how to cross compile most autotools projects.

## Toolchain

At first we need to get a cross compiler toolchain and a sysroot environment.
Luckily the raspberry pi foundation does provide both on their github profile.
So we'll can just clone them onto our computer.

```bash
$ git clone --depth 1 https://github.com/raspberrypi/tools rpi-tools
```

Note that I used the attribute `--depth 1` which tells git only to fetch data
from the latest commit. As the tools repository is rather large this minimizes
the amount of data we have to download.

We will use the `arm-linux-gnueabihf` cross compiler which is at the
time of this writing the latest and seems to support all raspberry device
versions. The next step to get a hold of tools the toolchain provides.

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

There are more but for most projects those are suffcient. Further we need the
sysroot environment for this toolchain and add it to our compiler flags.

```bash
SYSROOT=$PWD/rpi-tools/arm-bcm2708/arm-rpi-4.9.3-linux-gnueabihf/arm-linux-gnueabihf/sysroot
CFLAGS+="--sysroot=${SYSROOT}"
CPPFLAGS+="--sysroot=${SYSROOT}"
CXXFLAGS+="--sysroot=${SYSROOT}"
```

## Cross Compiling

Any autotools project is configured with the configure script. Thus we need to
provide the configure script with the tools and flags we already prepared.

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

A lot of project use pkg-conf to find dependencies therefore we tell autotools
where to find raspberry pi specific pkg-conf configuartions.

```bash
CONFIG_OPTS+=("PKG_CONFIG_DIR=")
CONFIG_OPTS+=("PKG_CONFIG_LIBDIR=${SYSROOT}/usr/lib/arm-linux-gnueabihf/pkgconfig:${SYSROOT}/usr/share/pkgconfig")
CONFIG_OPTS+=("PKG_CONFIG_SYSROOT=${SYSROOT}")
```

We might have to cross compile dependencies ourself. That's why we let pkg-conf
look into our install directory for dependencies as well.

```bash
CONFIG_OPTS+=("PKG_CONFIG_PATH=${BUILD_PREFIX}/lib/pkgconfig")
```

```bash
if [ ! -e libzmq ]; then
    git clone --depth 1 https://github.com/zeromq/libzmq.git
fi
pushd libzmq
(
    ./autogen.sh &&
    ./configure "${CONFIG_OPTS[@]}" &&
    make -j4 &&
    make install
) || exit 1
popd

if [ ! -e czmq ]; then
    git clone --depth 1 https://github.com/zeromq/czmq.git
fi
pushd czmq
(
    ./autogen.sh &&
    ./configure "${CONFIG_OPTS[@]}" &&
    make -j4
    make install
) || exit 1
popd
```

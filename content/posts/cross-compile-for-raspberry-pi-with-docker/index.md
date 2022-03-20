---
title: "Cross compile for Raspberry Pi with Docker"
description: "Compile C/C++ code for ARM using  Docker and buildx."
date: 2020-10-02T19:51:00+08:00
lastmod: 2022-03-20T11:42:34+08:00
draft: no
authors: ["rolandjitsu"]

tags: ["cpp", "cross-compile", "raspberry-pi", "docker"]
categories: ["DevOps"]

toc:
  enable: yes
---

The [Raspberry Pi](https://www.raspberrypi.org/) is a pretty useful tool for quickly prototyping some IoT product, simulating an embedded environment, running a small [Kubernetes cluster](https://www.jeffgeerling.com/tags/turing-pi), etc; there are probably lots of reasons why you'd want to use one (or maybe not?).

As for developing software that can run on it, if you're writing it in C or C++,  there's many ways to go about it:
1. Sync the code changes you make on the Pi and build it natively on it (via rsync, scp, [SSHFS](https://www.raspberrypi.org/documentation/remote-access/ssh/sshfs.md), git, etc)
2. Use [crosstool-ng](https://crosstool-ng.github.io/) to setup a toolchain and use it to cross-compile your software; then copy the resulting binaries onto the Pi
3. Use a [precompiled toolchain](https://github.com/abhiTronix/raspberry-pi-cross-compilers) to cross-compile the software; then copy the binaries onto the Pi
4. Develop directly on the Pi through an SSH session
5. Use a containerized OS such as [Balena](https://www.balena.io/)

And you can probably find many other creative ways developers came up with.

This mean that choosing which way to do it can be difficult, unless you have some clear project requirements.

For most of the projects that I've been working on, the requirements have been:
1. No source code on the Pi (and consequently no build tools installed on it)
2. Cannot run code inside containers on the Pi (the latency cost for communicating with peripherals is too high - especially when accessing the camera)
3. Anyone (on any platform) can clone the source code and build the binaries on their machine
4. A CI pipeline that builds everything in the cloud can be integrated later on
5. Builds don't take more than a few minutes (ideally less than a minute)

And after trying most of the options I could find out there, I've found that using [Docker](https://www.docker.com/) and cross-compilation works best. And with the release of [buildx](https://github.com/docker/buildx), it's even easier to build and dump the binaries on the host system.

## Setup your machine
You need to install:
1. [Docker](https://docs.docker.com/engine) >= `19.03.13`
2. [buildx(https://github.com/docker/buildx#installing)] >= `v0.4.1 `

### Setup the builder
You will need to setup a Docker builder instance (see [working with builder instances](https://github.com/docker/buildx#working-with-builder-instances)) that uses the `docker` driver.

Check current builder instances:
```bash
docker buildx ls
```

If you see an instance that uses the `docker` driver, switch to it (it's usually the `default` instance):
```bash
docker buildx use <instance name>
```

Otherwise, create a builder:
```bash
docker buildx create --name my-builder --driver docker --use
```

{{< admonition type=note >}}
You cannot create more than one instance using the `docker` driver.
{{< /admonition >}}


Then inspect and bootstrap it:
```bash
docker buildx inspect --bootstrap
```

## Setup the base images
Create a simple base image with the necessary tools to cross-compile:
```dockerfile
# Dockerfile.cross
FROM debian:stretch

ENV GNU_HOST=arm-linux-gnueabihf
ENV C_COMPILER_ARM_LINUX=$GNU_HOST-gcc
ENV CXX_COMPILER_ARM_LINUX=$GNU_HOST-g++

ENV CROSS_INSTALL_PREFIX=/usr/$GNU_HOST
ENV CROSS_TOOLCHAIN=/arm.toolchain.cmake

# https://cmake.org/cmake/help/v3.16/manual/cmake-toolchains.7.html#cross-compiling-for-linux
# https://cmake.org/cmake/help/v2.8.11/cmake.html#variable%3aCMAKE_PREFIX_PATH
RUN echo "set(CMAKE_SYSTEM_NAME Linux)" >> $CROSS_TOOLCHAIN && \
  echo "set(CMAKE_SYSTEM_PROCESSOR arm)" >> $CROSS_TOOLCHAIN && \
  echo "set(CMAKE_C_COMPILER /usr/bin/$C_COMPILER_ARM_LINUX)" >> $CROSS_TOOLCHAIN && \
  echo "set(CMAKE_CXX_COMPILER /usr/bin/$CXX_COMPILER_ARM_LINUX)" >> $CROSS_TOOLCHAIN && \
  echo "set(CMAKE_PREFIX_PATH $CROSS_INSTALL_PREFIX)" >> $CROSS_TOOLCHAIN

RUN apt-get update && \
  apt-get --no-install-recommends install -y autoconf \
    automake \
    build-essential \
    ca-certificates \
    curl \
    # C/C++ cross compilers
    gcc-$GNU_HOST \
    g++-$GNU_HOST \
    git \
    gnupg \
    libssl-dev \
    openssh-client \
    pkg-config \
    software-properties-common \
    wget && \
  rm -rf /var/lib/apt/lists/*

ENV CMAKE_VERSION 3.16.4

RUN export CMAKE_DIR=cmake-$CMAKE_VERSION && \
  export CMAKE_ARCH=$CMAKE_DIR.tar.gz && \
  wget --progress=bar:force:noscroll https://github.com/Kitware/CMake/releases/download/v$CMAKE_VERSION/$CMAKE_ARCH && \
  tar -xzf $CMAKE_ARCH && \
  cd $CMAKE_DIR && \
  ./bootstrap --parallel=`nproc` && \
  make -j `nproc` && \
  make install && \
  rm -rf ../$CMAKE_ARCH \
    ../$CMAKE_DIR
```

And build it:
```bash
docker buildx build -f Dockerfile.cross --tag cross-stretch .
```

{{< admonition type=note >}}
By default, the image is going to be available to use on the host as `cross-stretch`. If `docker images` doesn't show it, add the `--load` flag when building.
{{< /admonition >}}

{{< admonition type=tip >}}
To bust the cache, use `--no-cache`.
{{< /admonition >}}

Now create a base image w/ some common libs usually available on the Pi:
```dockerfile
# Dockerfile.cross-pi
FROM cross-stretch

# NOTE: These versions must be the same on the target host
ENV RPI_SHA=f73fca0
ENV WIRINGPI_SHA=6d9ce35

# MMAL/VCOS (https://github.com/raspberrypi/userland)
RUN RPI_DIR=/raspberrypi && \
  RPI_BUILD_DIR=$RPI_DIR/build/arm-linux/release && \
  git clone --single-branch --branch master https://github.com/raspberrypi/userland.git $RPI_DIR && \
  cd $RPI_DIR && \
  git checkout $RPI_SHA && \
  mkdir -p $RPI_BUILD_DIR && \
  cd $RPI_BUILD_DIR && \
  cmake -DCMAKE_TOOLCHAIN_FILE=../../../makefiles/cmake/toolchains/arm-linux-gnueabihf.cmake \
    -DCMAKE_BUILD_TYPE=Release \
    ../../.. && \
  make -j `nproc` && \
  make install DESTDIR=$CROSS_INSTALL_PREFIX/ && \
  cd / && \
  rm -rf $RPI_DIR && \
  # WiringPi
  # https://github.com/WiringPi/WiringPi
  WPI_DIR=/wpi && \
  WPI_WIRING_PI_DIR=$WPI_DIR/wiringPi && \
  git clone --single-branch --branch master https://github.com/WiringPi/WiringPi.git $WPI_DIR && \
  cd $WPI_DIR && \
  git checkout $WIRINGPI_SHA && \
  cd $WPI_WIRING_PI_DIR && \
  CC=$C_COMPILER_ARM_LINUX make -j `nproc` && \
  make install DESTDIR=$CROSS_INSTALL_PREFIX PREFIX="" && \
  cd / && \
  rm -rf $WPI_DIR
```

And build it:
```bash
docker buildx build -f Dockerfile.cross-pi --tag cross-pi .
```

## Compile some binaries
Now that we have the base images ready, we can go ahead and create a simple programs to illustrate how to cross-compile.

So, let's create a program to print the current Raspberry Pi model and revision:
```cpp
// hello.cpp
#include <iostream>
#include <fstream>

int main()
{
  std::string m;
  std::ifstream stream("/sys/firmware/devicetree/base/model");
  std::getline(stream, m);

  std::cout << m << std::endl;

  return 0;
}
```

And the dockerfile for building it:
```dockerfile
# Dockerfile.hello
FROM cross-stretch AS builder

COPY ./hello.cpp /code/

ENV BIN_DIR /tmp/bin

RUN mkdir -p $BIN_DIR && \
  $CXX_COMPILER_ARM_LINUX /code/hello.cpp -Ofast -Wall -o $BIN_DIR/hello

FROM scratch
COPY --from=builder /tmp/bin /
```

Now compile the binary:
```bash
docker buildx build -f Dockerfile.hello -o type=local,dest=./bin .
```

If you copy the binary over to the Pi and run it, you should be seeing your Pi's model and revision, e.g:
![hello.cpp output](./hello.png "hello-output.png")

While the above example might be good enough to illustrate how we can use Docker to cross-compile, it's probably incomplete without illustrating how to use some of the libs that are usually available on the Pi.

Therefore, let's create another program that makes use of the popular [wiringPi](https://github.com/WiringPi/WiringPi) lib and the [libs that interface with the GPU](https://github.com/raspberrypi/userland) (mind the copyright):
```c
// hello-pi.c
/*
Copyright (c) 2012, Broadcom Europe Ltd
All rights reserved.
Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:
    * Redistributions of source code must retain the above copyright
      notice, this list of conditions and the following disclaimer.
    * Redistributions in binary form must reproduce the above copyright
      notice, this list of conditions and the following disclaimer in the
      documentation and/or other materials provided with the distribution.
    * Neither the name of the copyright holder nor the
      names of its contributors may be used to endorse or promote products
      derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY
DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "interface/vmcs_host/vc_vchi_gencmd.h"
#include "interface/vmcs_host/vc_gencmd_defs.h"
#include <wiringPi.h>

#define TEMP_KEY_SIZE 5

void chop(char *str, size_t n);

int main()
{
  // Get board model
  int m, rev, mem, maker, ov;
  piBoardId(&m, &rev, &mem, &maker, &ov);

  printf("Raspberry Pi: %s\n", piModelNames[m]);
  printf("Revision: %s\n", piRevisionNames[rev]);

  // Get SoC temperature
  // https://github.com/raspberrypi/userland/blob/master/host_applications/linux/apps/gencmd/gencmd.c
  // vcgencmd measure_temp
  VCHI_INSTANCE_T vchi_instance;
  VCHI_CONNECTION_T *vchi_connection = NULL;

  vcos_init();

  if (vchi_initialise(&vchi_instance) != 0)
  {
    fprintf(stderr, "VCHI init failed\n");
    return 1;
  }

  // Create a vchi connection
  if (vchi_connect(NULL, 0, vchi_instance) != 0)
  {
    fprintf(stderr, "VCHI connect failed\n");
    return 1;
  }

  vc_vchi_gencmd_init(vchi_instance, &vchi_connection, 1);

  char buffer[GENCMDSERVICE_MSGFIFO_SIZE];
  size_t buffer_offset = 0;
  int ret;

  // Reset the buffer
  buffer[0] = '\0';

  // Set the cmd
  buffer_offset = vcos_safe_strcpy(buffer, "measure_temp", sizeof(buffer), buffer_offset);

  // Send the gencmd for the argument
  if ((ret = vc_gencmd_send("%s", buffer)) != 0)
  {
    printf("vc_gencmd_send returned non-zero code: %d\n", ret);
    return 1;
  }

  // Get + print out the response
  if ((ret = vc_gencmd_read_response(buffer, sizeof(buffer))) != 0)
  {
    printf("vc_gencmd_read_response returned a non-zero code: %d\n", ret);
    return 1;
  }
  buffer[sizeof(buffer) - 1] = 0;

  if (buffer[0] != '\0')
  {
    if (strncmp(buffer, "temp=", TEMP_KEY_SIZE) == 0)
    {
      chop(buffer, TEMP_KEY_SIZE);
      printf("Temperature: %s\n", buffer);
    }
    else
      puts(buffer);
  }

  vc_gencmd_stop();

  // Close the vchi connection
  if (vchi_disconnect(vchi_instance) != 0)
  {
    fprintf(stderr, "VCHI disconnect failed\n");
    return 1;
  }

  return 0;
}

void chop(char *str, size_t n)
{
  size_t len = strlen(str);
  if (n > len)
    return;
  memmove(str, str + n, len - n + 1);
}
```

The dockerfile for building it:
```dockerfile
# Dockerfile.hello-pi
FROM cross-pi AS builder

COPY ./hello-pi.c /code/
COPY ./CMakeLists.txt /code/

ENV BIN_DIR /tmp/bin
ENV BUILD_DIR /code/build

RUN mkdir -p $BIN_DIR && \
  mkdir -p $BUILD_DIR && \
  cd $BUILD_DIR && \
  cmake -DCMAKE_TOOLCHAIN_FILE=$CROSS_TOOLCHAIN \
    -DCMAKE_INSTALL_PREFIX=$CROSS_INSTALL_PREFIX \
    ..  && \
  make -j `nproc` && \
  cp ./hello-pi $BIN_DIR/

FROM scratch
COPY --from=builder /tmp/bin /
```

And the cmake config to compile it:
```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.16)

project(hello-pi)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_INSTALL_PREFIX}/lib/cmake/ /usr/lib/cmake)
set(VC_INST_PATH ${CMAKE_PREFIX_PATH}/opt/vc)
set(VC_LIBS ${VC_INST_PATH}/lib)
set(VC_INCLUDES ${VC_INST_PATH}/include)
set(VCOS_PLATFORM pthreads)
add_definitions(-Wall -Werror)

find_library(mmalcore_LIBS NAMES mmal_core PATHS "${VC_LIBS}")
find_library(mmalutil_LIBS NAMES mmal_util PATHS "${VC_LIBS}")
find_library(mmal_LIBS NAMES mmal PATHS "${VC_LIBS}")
if((NOT mmal_LIBS) OR (NOT mmalutil_LIBS) OR (NOT mmalcore_LIBS))
  message(FATAL_ERROR "Could not find mmal libs")
endif()

find_path(vmcs_host_INC NAMES vmcs_host PATHS "${VC_INCLUDES}/interface")
if(NOT vmcs_host_INC)
  message(FATAL_ERROR "Could not find vmcs_host interfaces")
endif()

find_library(vchostif_LIBS NAMES vchostif PATHS "${VC_LIBS}")
find_library(vchiq_arm_LIBS NAMES vchiq_arm PATHS "${VC_LIBS}")
find_library(vcos_LIBS NAMES vcos PATHS "${VC_LIBS}")
if((NOT vcos_LIBS) OR (NOT vchiq_arm_LIBS) OR (NOT vchostif_LIBS))
  message(FATAL_ERROR "Could not find vcos/vchiq_arm/vchostif libs")
endif()

find_library(wiringPi_LIB NAMES wiringPi)
if(NOT wiringPi_LIB)
  message(FATAL_ERROR "Could not find wiringPi")
endif()

include_directories(${VC_INCLUDES}
  ${VC_INCLUDES}/interface/vcos
  ${VC_INCLUDES}/interface/vcos/${VCOS_PLATFORM})

add_executable(hello-pi hello-pi.c)
target_link_libraries(hello-pi
  optimized
  pthread
  ${vcos_LIBS}
  ${vchiq_arm_LIBS}
  ${vchostif_LIBS}
  wiringPi)
```

Now compile the binary:
```bash
docker buildx build -f Dockerfile.hello-pi -o type=local,dest=./bin .
```

And if you copy the binary over to the Pi and run it, you should be seeing your Pi's model and revision and the SoC temperature, e.g:
![hello-pi.c output](./hello-pi.png "hello-pi-output.png")

And there you go. I hope that this is enough to get you started.

You can find the source and further examples at [rolandjitsu/raspi-cross](https://github.com/rolandjitsu/raspi-cross).

# Mellanox SDK build infrastructure in sonic-buildimage design

## Table of content
- [Mellanox SDK build infrastructure in sonic-buildimage design](#mellanox-sdk-build-infrastructure-in-sonic-buildimage-design)
  - [Table of content](#table-of-content)
  - [Overview and motivation](#overview-and-motivation)
  - [Build Mellanox SDK from sources support](#build-mellanox-sdk-from-sources-support)
    - [1. Expose new build variable for Mellanox](#1-expose-new-build-variable-for-mellanox)
    - [2. Merge sdk-src.mk and sdk.mk](#2-merge-sdk-srcmk-and-sdkmk)
    - [3. Define *phony* target to build **all** SDK packages in one make command](#3-define-phony-target-to-build-all-sdk-packages-in-one-make-command)
  - [Build SONiC image with any FW, SDK, SAI, hw-mgmt? (for integration purpose)](#build-sonic-image-with-any-fw-sdk-sai-hw-mgmt-for-integration-purpose)
  - [sonic_build.sh](#sonicbuildsh)
- [Flow](#flow)
  - [SDK flow](#sdk-flow)
  - [FW flow](#fw-flow)
  - [SAI flow](#sai-flow)
  - [hw-mgmt flow](#hw-mgmt-flow)

## Overview and motivation
Currently we have local **develop** branch based on **master** branch of Azure *sonic-buildimage* repositry which includes additional makefiles to build Mellanox SDK binaries from sources using local infrastructure;
<p>
Issues we encounter with this approach:

+ **develop** branch diverges from **master** and makes more merge conflicts happen
+ Seperate makefiles with target for building from source and downloading pre-built binaries
+ Untill *sx-kernel* is not open-sourced, MSFT cannot rebuild *sx-kerenel* on their own when linux kernel version changes
+ No way to build all SDK packages in one command
+ Hard to automate SDK build procedure

So, our goal is to:

+ Push SDK makefiles to upstream
+ Provide user an option to *build* or *download* SDK
  + Only users with granted access to SDK sources (Mellanox, MSFT) can use **build**
+ Provide ability to build image with any FW, SDK, SAI version
  + later will be used in Jenkins job


This design document tries to provide a solution for above issues and consists of two parts:
- [Build Mellanox SDK from sources support](#build-mellanox-sdk-from-sources-support)
- [Build SONiC image with any FW, SDK, SAI (for integration purpose)](#build-sonic-image-with-any-fw-sdk-sai-for-integration-purpose)


## Build Mellanox SDK from sources support

### 1. Expose new build variable for Mellanox

```MLNX_SDK_SOURCES_BASE_URL```

**NOTE**:
<br>
Assumption: ```$(MLNX_SDK_SOURCES_BASE_URL)/$(PACKAGE_NAME)-$(MLNX_SDK_VERSION)-$(MLNX_SDK_ISSU_VERSION).tar.gz```

**NOTE**:<br>
New variable **MLNX_SDK_ISSU_VERSION** is added; developer who integrates new SDK must pay attantion and update this variable in sdk.mk 

### 2. Merge sdk-src.mk and sdk.mk

*platform/mellanox* tree structure:
```
platform/
...
├── mellanox/
        ├── sdk.mk
        ├── sdk-src/
        │   ├── applibs/
        │   │   └── Makefile
        │   ├── iproute2/
        │   │   └── Makefile
        │   ├── python-sdk-api/
        │   │   └── Makefile
        │   ├── sx-complib/
        │   │   └── Makefile
        │   ├── sxd-libs/
        │   │   └── Makefile
        │   ├── sx-examples/
        │   │   ├── Makefile
        │   ├── sx-gen-utils/
        │   │   └── Makefile
        │   ├── sx-kernel/
        │   │   ├── Makefile
        │   │   └── sx_kernel_makefile_sonic_build.patch
        │   └── sx-scew/
        │       └── Makefile
        ...
...
```

An example how it is done for some SDK *$PACKAGE*:<br>
*platform/mellanox/sdk.mk*:
```
...
$(PACKAGE)_SRC = $(PLATFORM_PATH)/sdk-src/package/ # for build
$(PACKAGE)_URL = $(MLNX_SDK_BASE_URL)/$(PACKAGE) # for download
...

ifeq ($(MLNX_SDK_PROCURE_METHOD),build)
SONIC_MAKE_DEBS += $(PACKAGE)
else
SONIC_ONLINE_DEBS += $(PACKAGE)
endif
```

### 3. Define *phony* target to build **all** SDK packages in one make command

To be able to build only SDK and its dependencies

*platform/mellanox/sdk.mk*:
```
mlnx-sdk-pacakges : $(addprefix $(DEBS_PATH),$(MLNX_SDK_ALL))

SONIC_PHONY_TARGETS += mlnx-sdk-packages
```

Usage:
```
$ make -f Makefile.work BLDENV=stretch mlnx-sdk-packages
```

## Build SONiC image with any FW, SDK, SAI, hw-mgmt? (for integration purpose)

## sonic_build.sh

```
Usage: ./sonic_build.sh [OPTIONS]

OPTIONS:
  -r, --repo-url         SONiC git repository url
  -b, --branch-name      SONiC git branch name
  -d, --build-dir        Build directory path
  -c, --continue-build   Continue build in --build-dir directory
  -p, --patch-file       SONiC patch file path
  -s, --series           Series file with patches
  -j, --jobs             SONiC build jobs number
  -q, --quiet            Print errors only

      --rpc              Enable SONiC syncd rpc
      --debugging        Enable SONiC debugging and install debug tools (SONIC_DEBUGGING_ON=y INSTALL_DEBUG_TOOLS=y)
      --profiling        Enable SONiC profiling and install debug tools (SONIC_PROFILING_ON=y INSTALL_DEBUG_TOOLS=y)
                         NOTE: this option will override --debugging option

ASIC components:

 SDK options:
      --sdk-version      SDK version string, e.g "4.3.0134"
      --sdk-issu-version SDK issu version string, e.g “100”
      --sdk-sources-url  SDK sources URL
                         If not specified and --sdk-version is defined
                         will use "http://arc-build-server/sdk/sx_sdk_eth-<sdk-version>/SOURCES/"
 FW options:
      --fw-spc-version   FW SPC1 version string
      --fw-spc2-version  FW SPC2 version string
      --fw-binaries-url  FW mfa binaries URL
                         If not specified and --fw-spc-version or --fw-spc2-version is defined
                         will use "http://arc-build-server/fw/"
 SAI options:
      --sai-repo-url     SAI implementation repo
      --sai-commit       SAI commit (branch, tag)
                         Will automatically change debian version string

 hw-mgmt options
      --hw-mgmt-commit   hw-mgmt commit (branch, tag)
                         Will automatically change debian version string

  -h, --help         Print help
```

# Flow

## SDK flow

1. If specified "sdk-version" override MLNX_SDK_VERSION in platform/mellanox/sdk.mk
2. If specified "sdk-issu-version" override MLNX_SDK_ISSU_VERSION in platform/mellanox/sdk.mk
3. If specified "sdk-version" sdk-sources-url or explicitely specified "sdk-sources-url"
   override MLNX_SDK_SOURCES_BASE_URL in platform/mellanox/sdk.mk

## FW flow
1. If specified "fw-spc-version" override MLNX_FW_SPC_VERSION in platform/mellanox/fw.mk
2. If specified "fw-spc2-version" override MLNX_FW_SPC2_VERSION in platform/mellanox/fw.mk
3. If any "fw-spc-version" or "fw-spc2-version" specified or explicitely specified "fw-binaries-url"
   override MLNX_FW_BASE_URL in platform/mellanox/fw.mk

## SAI flow
1. If specified "sai-repo-url" and "sai-commit"
   1. Override submodule url in .gitmodules
   2. git submodule sync && git submodule update $path
2. If specified "sai-commit"
   1. pushd $path && git fetch origin && git checkout $sai_commit && popd
   2. parse debian version string mlnx_sai/debian/changelog
   3. override MLNX_SAI_VERSION

## hw-mgmt flow
1. If specified "hw-mgmt-commit"
2. pushd $path && git fetch origin && git checkout $hw-mgmt-commit && popd
4. parse debian version string mlnx_sai/debian/changelog
5. override MLNX_HW_MANAGEMENT_VERSION


Can we use SONIC_DPKG_DEBS target for SAI and hw-mgmt

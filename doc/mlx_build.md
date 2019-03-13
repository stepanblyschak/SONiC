# Mellanox SDK build infrastructure in sonic-buildimage design

## Table of content
- [Mellanox SDK build infrastructure in sonic-buildimage design](#mellanox-sdk-build-infrastructure-in-sonic-buildimage-design)
  - [Table of content](#table-of-content)
  - [Overview and motivation](#overview-and-motivation)
  - [Build Mellanox SDK from sources support](#build-mellanox-sdk-from-sources-support)
    - [1. Expose new build variable for Mellanox](#1-expose-new-build-variable-for-mellanox)
    - [2. Merge sdk-src.mk and sdk.mk](#2-merge-sdk-srcmk-and-sdkmk)
    - [3. Define *phony* target to build **all** SDK packages in one make command](#3-define-phony-target-to-build-all-sdk-packages-in-one-make-command)
  - [Build SONiC image with any FW, SDK, SAI (for integration purpose)](#build-sonic-image-with-any-fw-sdk-sai-for-integration-purpose)
    - [1. Expose FW, SDK, SAI version variables during build](#1-expose-fw-sdk-sai-version-variables-during-build)
    - [2. Override versions in relevant makefiles](#2-override-versions-in-relevant-makefiles)

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
If configured platform is *"mellanox"* two additional variables become available during build:

| Variable                  | Possible values                | Comment                                                                                                                                                                                   |
| ------------------------- | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MLNX_SDK_PROCURE_METHOD   | "build"<br>"download"(default) | Build SDK from sources; in this case **MLNX_SDK_SOURCES_BASE_URL** must be specified, otherwise buildsystem raises an error; <br> Download SDK binaries from **MLNX_SDK_BASE_URL** defined in sdk.mk |
| MLNX_SDK_SOURCES_BASE_URL |                                | A base URL to SDK sources tarball;<br> Has no affect when **MLNX_SDK_PROCURE_METHOD** is not "build"                                                                                          |

**NOTE**:
<br>
Assumption: ```$(MLNX_SDK_SOURCES_BASE_URL)/$(PACKAGE_NAME)-$(MLNX_SDK_VERSION)-$(MLNX_SDK_ISSU_VERSION).tar.gz```

**NOTE**:<br>
New variable **MLNX_SDK_ISSU_VERSION** is added; developer who integrates new SDK must pay attantion and update this variable in sdk.mk 

Usage:
```
make MLNX_SDK_PROCURE_METHOD=build MLNX_SDK_SOURCES_BASE_URL=http://arc-build-server/sx_sdk_eth/sx_sdk_eth-4.3.0134/
SONiC Build System

Build Configuration
...
"MLNX_SDK_PROCURE_METHOD"         : "build"
"MLNX_SDK_SOURCES_BASE_URL"       : "http://arc-build-server/sx_sdk_eth/sx_sdk_eth-4.3.0134/"
```

**NOTE**:
<br>
Symbolic link in /var/www/html/sx_sdk_eth/ to sx_sdk_eth/ should be created on arc-build-server;
<br>
No need to mount specific directories in sonic-slave docker

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

## Build SONiC image with any FW, SDK, SAI (for integration purpose)

### 1. Expose FW, SDK, SAI version variables during build

e.g:
```
$ make MLNX_SDK_VERSION=4.3.0136 MLNX_SDK_PROCURE_METHOD=build MLNX_SDK_SOURCES_BASE_URL=http://arc-build-server/sx_sdk_eth/sx_sdk_eth-4.3.0136/
```

Pros:
+ Simple for automation

Cons:
+ Exposes a lot of additional vendor specific variables during build
+ Since no changes are made to source tree, images are built with same name and build commit, it is inconvinient for two reasons:
  + These images will contain different FW, SDK, SAI versions
  + sonic_installer will refuse to install image which have the same name

### 2. Override versions in relevant makefiles

Pros:
+ No changes are requiered
+ No issue with image name - need to either commit change or dirty image with timestamp in the name will be built 

Cons:
+ Less simple for automation


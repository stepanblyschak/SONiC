# Mellanox Fast-Fast boot in SONiC
# High Level Design Document
### Rev 0.2
## Table of Contents
## List of Tables
###### Revision
| Rev |     Date    |       Author          | Change Description                |
|:---:|:-----------:|:---------------------:|-----------------------------------|
| 0.2 |             | Stepan Blyshchak      | Initial version                   |
## About this Manual
This document provides general information about the Mellanox fastfast boot feature implementation in SONiC.
## Scope
This document describes the high level design and flows of the Mellanox fastfast boot
## Definitions/Abbreviation
###### Table 2: Abbreviations
| Definitions/Abbreviation | Description                                |
|--------------------------|--------------------------------------------|
| WB                       | Warm boot                                  |
| FB                       | Fast boot                                  |
| FFB                      | Fast-Fast boot                             |
| ISSU                     | In Service Software Upgrade                |
| SDK                      | Software Development Kit                   |
| API                      | Application Programmable Interface         |
| SAI                      | Switch Abstraction Interface               |
| CLI                      | Command Line Interface                     |

# 1 Overview
Mellanox FFB is an implementation of system level warm reboot in SONiC for Mellanox platforms that has a minimum data plane down time (less than 1 sec) for MSFT T0 configuration.

<b>NOTE</b>: This flow is not relevant for the 0 sec down time.

## Requirements
* The FFB flow is issued by ```sudo warm-reboot``` CLI command on Mellanox platforms
  * On "warm boot enabled" SKUs - proceed with FFB flow
  * On "warm boot disabled" SKUs - return with appropriate error message
* Assumed only system level warm reboot and docker level warm restart (except syncd) via FFB is supported
* DP downtime should be less than 1 sec
* CP downtime should be the same as for FB flow - less than 90 sec

## Limitations
* Port mapping configuration in SAI XML profile should match port_config.ini. Otherwise correct SDK/HW behaviour is NOT guaranteed
* In "warm boot enabled" mode half of resources (RIFs, Routes, ACLs, etc.) are available
* The solution is tailored for MSFT AZURE T0 configuration


# 2 Components changes

### 2.2 SDK
Mellanox does not plan to break SDK upgrade in fastfast reboot frequently. But in rare cases in may happen. To be sure "warm-reboot" to new SDK is possible we need to expose SDK version string in SONiC image FS.
SDK version will be exposed in ```/etc/mlnx/sdk_version```. This file will be generated at image build time.


E.g
sonic-buildimage/platform/mellanox/sdk_version.j2:
```jinja2
{{ SDK_VERSION }}
```

/etc/mlnx/sdk_version:
```
4.2.9108
```

### 2.1 SAI
SAI XML profile exposes a new node that indicates wheter SDK starts in "warm boot enabled" mode.
E.g:
```xml
<issu-enabled>1</issu-enabled>
```

SAI should handle ```SAI_BOOT_TYPE=1``` to handle warm start and start SDK in "fastfast" mode

SAI should handle ```SAI_KEY_WARM_BOOT_WRITE_FILE``` value to tell where SDK can put persistent files for fastfast boot.

SAI should hanlde next attributes:

| SAI Attributes | Description                                |
|--------------------------|--------------------------------------------|
| SAI_SWITCH_ATTR_PRE_SHUTDOWN | Call SDK issu start API |
| SAI_SWITCH_ATTR_RESTART_WARM | Ignore |
| SAI_SWITCH_ATTR_FAST_API_ENABLE| Call SDK issu end API |


## 2.2 warm boot status CLI command
A new platform specific show command in ```show``` commands group will be added in order to get whether SDK was started in "warm boot enabled" or normal mode

This command will be used during FFB to check whether we can perform FFB.

E.g:

```bash
admin@sonic:~ show platform mellanox issu status
ENABLED
```

The script will parse SAI XML profile to get the ISSU status node which is described in 2.1.

The same approach uses Mellanox SDK Sniffer configuration.

## 2.2 New kernel boot parameter
In order to have a way to handle Mellanox warm reboot specifically a new kernel SONiC boot mode parameter will be introduced:
```
SONIC_BOOT_TYPE=fastfast
```

## 2.2 syncd changes

* The syncd daemon should handle new startup command line flag ```-t fastfast```.
* When "fastfast" boot mode is specified syncd will ignore warm boot flag indication in redis DB and proceed as usual
* syncd should pass to SAI ```SAI_KEY_BOOT_TYPE = 1```, so SAI will initialize SDK in FFB way.
* On received APPLY_VIEW notification in "fastfast" mode syncd sets ```SAI_SWITCH_ATTR_FAST_API_ENABLE``` to ```false```
This will notify Mellanox SDK that restoration phase is complete and it's time to switch to a "new" bank


## 2.3 syncd_init_common.sh changes

The syncd_init_common.sh should configure syncd to start in "fastfast" mode only if two below conditions are met:

* Kernel boot parameter is SONIC_BOOT_TYPE=fastfast
* /host/warmboot/warm-starting exists

NOTE: Additional second condition is requiered because user can do "config reload" while kernel boot arguments remain the smae

## 2.4 mlnx-ffb.sh

The new script will live in /usr/bin/ and will contain a check function to be called in "warm-reboot" script

## 2.5 warm-reboot

warm-reboot script need handle two special flows for fastfast reboot:

* warm-reboot script will be responsible to check that FFB is possible by using mlnx-ffb.sh helper script.
Two checks should be performed:
    * ISSU is enabled for this HWSKU
    * SDK upgrade is supported
* ASIC_DB, COUNTERS DB have to be fushed right after pre shutdown request is executed successfully.
  
### 3.1 Mellanox fast fast reboot flow 
![](https://raw.githubusercontent.com/stepanblyschak/SONiC/fast-fast/doc/warm-reboot/img/mlnx_fastfast_boot_hld/fastfast-shutdown.svg)


# Open issues

## 

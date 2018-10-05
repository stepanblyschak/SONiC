# Mellanox Fast-Fast boot in SONiC
# High Level Design Document
### Rev 0.1
## Table of Contents
## List of Tables
###### Revision
| Rev |     Date    |       Author          | Change Description                |
|:---:|:-----------:|:---------------------:|-----------------------------------|
| 0.1 |             | Stepan Blyshchak      | Initial version                   |
## About this Manual
This document provides general information about the MLNX FFB feature implementation in SONiC.
## Scope
This document describes the high level design and flows of the MLNX FFB feature
## Definitions/Abbreviation
###### Table 2: Abbreviations
| Definitions/Abbreviation | Description                                |
|--------------------------|--------------------------------------------|
| FB                       | Fast boot                                  |
| FFB                      | Fast-Fast boot                             |
| ISSU                     | In Service Software Upgrade                |
| SDK                      | Software Development Kit                   |
| API                      | Application Programmable Interface         |
| SAI                      | Switch Abstraction Interface               |
| CLI                      | Command Line Interface                     |

So, MLNX Warm boot == MLNX FFB
# 1 Overview
MLNX FFB is an implementation of system level Warm boot in SONiC for Mellanox platforms that has a minimum data plane down time (less than 1 sec) for MSFT T0 configuration.

<b>NOTE</b>: This flow is not relevant for the 0 sec down time.

## Requirements
* The FFB flow is issued by Warm reboot CLI command on Mellanox platforms
  * On "warm boot enabled" devices - proceed with FFB flow
  * On "warm boot disabled" devices - return with appropriate error message
* Assumed only system level warm reboot via FFB is supported
* DP downtime should be less than 1 sec
* CP downtime should be the same as for FB flow - less than 90 sec

## Limitations
* Port mapping configuration in SAI XML should match port_config.ini. Otherwise correct SDK/HW behaviour is NOT guaranteed
* In "warm boot enabled" mode half of resources (RIFs, Routes, ACLs, etc.) are available
* The solution is tailored for MSFT AZURE T0 configuration


# 2 Components changes

### 2.1 SAI
SAI XML profile exposes a new node that indicates wheter SDK starts in "warm boot enabled" mode.

### 2.2 SDK
SDK version will be exposed in ```/etc/mlnx/sdk_version```. This file will be generated at image build time.

## 2.2 warm boot status CLI command
A new platform show command in ```show``` commands group will be added in order to get whether SDK was started in "warm boot enabled" or normal mode

This command will be used during FFB to check whether we can FFB.

E.g:

```
admin@sonic:~ show platform mellanox warm_boot status
ENABLED
```

The script will parse SAI XML profile to get the ISSU status node.

The same approach uses MLNX SDK Sniffer configuration.

## 2.2 syncd changes

* The syncd daemon should handle new startup command line flag ```-t fast-fast```.
* Syncd should pass to SAI ```SAI_KEY_BOOT_TYPE = 1```, so SAI will initialize SDK in FFB way.
* Also, unlike WB, syncd should NOT perform WARM start comparison logic.


## 2.3 syncd_init_common.sh changes

The syncd_init_common.sh should get that reboot cause was 'fast-fast-boot' from ```/proc/cmdline``` and pass option ```-t fast-fast``` to syncd start options.

## 2.4 mlnx-ffb.sh

The new script will live in /usr/bin/ and will contain 2 check functions to check whether ISSU is enabled on a device and SDK supports ISSU upgrade.

These functions will be used by warm-reboot script

## 2.4 Shutdown flow

'System-wide Warmboot' design document is describing the flow in general right now.

Specificaly, it is not clear who will do change config in ```WARM_RESTART_TABLE```. 

- User issues Warm reboot CLI:
  - If Mellanox platform then perform FFB
  - Else - do regular SONiC warm reboot flow for the rest of platforms
  
### 3.1 Mellanox fast fast reboot flow 
#### ```warm-reboot```
  - MLNX specific flow:
  
    - source ```/usr/bin/mlnx-ffb.sh``` script
  
    - Check if FFB is supported by HWSKU, if not - reject the command
      <br>
      Example error message:
      <br>
        ```Warm reboot is not supported by this HWSKU```
     
    - Check if FFB is supported by SDK:
      <br>
      Call the script ```issu.py --check``` inside ```syncd``` and pass current SDK version and next SDK version as parameters
      <br>
      Example error message:
      <br>
        ```Warm reboot is not supported to new SDK version $VERSION_NEW```
      <br>
      SDK version we can get from ```/etc/mlnx/sdk_version```
      <br>
      
      <b>NOTE:</b> On rare cases upgrade will be not supported

    - Unset ```WARM_RESTART:system```
      
    - Burn new FW if new FW is available in next boot SONiC image: 
      <br>
      ```mlnx-fw-upgrade.sh --upgrade```
 
    - Execute ISSU start script ```issu.py --start``` inside ```syncd``` container
    
  - Dump ARP/FDB entries from APP DB - existing step in FB
      
  - bgp, teamd dockers config restart in regular WARM SONiC way via config DB key
    <br>
    ```WARM_RESTART_TABLE|bgp```
    <br>
    ```WARM_RESTART_TABLE|teamd```
    <br>
    stop bgp and teamd services
    <br>
    
    According to system level WB design doc bgp will enable GR, teamd - just kill
  
  - Dump WARM_RESTART table in config DB and state DB
    
  - execute ```docker kill``` on every other container (swss, syncd, pmon, snmp, lldp)
  
  - Mark reboot cause file - existing step in FB
    - Similar to FB:
      ```User issued 'fast-fast-reboot' command [User: ${REBOOT_USER}, Time: ${REBOOT_TIME}]```
  
  - BOOT_OPTIONS += 'fast-fast-reboot' (instead of 'fast-reboot' in FB case)
  
  - kexec $BOOT_OPTIONS

## 3.2 Startup flow
Similar to fast-reboot:
  - database
    - Restore WARM_RESTART table in config DB and state DB
  - Start services
    - swss started normaly in non warm way
    
    - syncd:
      - ```if 'fast-fast-reboot' in $(cat /proc/cmdline); then```
        - started with with ```-t fast-fast```
        
    - bgp, teamd starts in warm way
    
  - Orchagent starts configuring HW
  
  - When configuration is done:
    - set ```SAI_SWITCH_ATTR_FAST_API_ENABLE``` to ```false```, so SAI will call ISSU end API
    
  - END

# Open issues

## <b> When to execute ISSU end? </b>

## <b> Who should set ```WARM_RESTART_TABLE```? CLI? User? </b>

## <b> Can we addapt SONiC warm reboot flow by using INIT/APPLY view approach inside syncd? </b>

Assume swss will restart in Warm way.

The APP DB is restored and orchagent will process the APP DB and pushes configuration to syncd.

Syncd is in INIT_VIEW mode and will construct a temp view (no configuration applied on HW).

Orchagent notifies syncd to do APPLY_VIEW. syncd will do comparison logic between current (the view discovered from current ASIC state)  and temporary view and push the delta config to the HW.

If we can reuse this approach in different way - apply all configuration that is in temp view instead of computing delta, so we can restore the pre-reboot state

It could solve 1st issue, beacause there is exact moment of time when exact pre-shutdown configuration is applied.

All other dockers should restart in warm way. Only syncd handles it specificaly.

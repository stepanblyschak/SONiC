# What Just Happened in SONiC HLD

# High Level Design Document

#### Rev 0.3

# Table of Contents
- [What Just Happened in SONiC HLD](#what-just-happened-in-sonic-hld)
- [High Level Design Document](#high-level-design-document)
      - [Rev 0.3](#rev-03)
- [Table of Contents](#table-of-contents)
- [List of Tables](#list-of-tables)
- [List of Figures](#list-of-figures)
- [Revision](#revision)
- [About this Manual](#about-this-manual)
- [Scope](#scope)
- [Definitions/Abbreviation](#definitionsabbreviation)
- [1. Overview](#1-overview)
- [2 Requirements Overview](#2-requirements-overview)
  - [2.1 Functional requirements](#21-functional-requirements)
- [3 Modules design](#3-modules-design)
  - [3.1 WJH build and runtime dependencies](#31-wjh-build-and-runtime-dependencies)
  - [3.2 WJH docker container in SONiC](#32-wjh-docker-container-in-sonic)
  - [3.3 WJH service in SONiC](#33-wjh-service-in-sonic)
  - [3.4 WJH feature table](#34-wjh-feature-table)
  - [3.5 WJH reasons and severity configuration file](#35-wjh-reasons-and-severity-configuration-file)
  - [3.6 WJH and debug counters](#36-wjh-and-debug-counters)
  - [3.7 Config DB](#37-config-db)
    - [3.7.1 WJH table schema](#371-wjh-table-schema)
    - [3.7.2 WJH table defaults](#372-wjh-table-defaults)
  - [3.8 WJH default configuration for debug mode](#38-wjh-default-configuration-for-debug-mode)
  - [3.9 WJH provided data](#39-wjh-provided-data)
    - [3.9.1 Raw Channel](#391-raw-channel)
    - [3.9.2 Aggregated channel](#392-aggregated-channel)
  - [3.10 Mapping SDK IDs to SONiC IDs/object names](#310-mapping-sdk-ids-to-sonic-idsobject-names)
    - [3.10.1 SDK logical port ID/netdev IF_INDEX to SONiC port name mapping](#3101-sdk-logical-port-idnetdev-if_index-to-sonic-port-name-mapping)
    - [3.10.2 SDK logical LAG ID to SONiC LAG name mapping](#3102-sdk-logical-lag-id-to-sonic-lag-name-mapping)
    - [3.10.3 SDK ACL rule ID to SONiC ACL rule name mapping](#3103-sdk-acl-rule-id-to-sonic-acl-rule-name-mapping)
  - [3.11 CLI](#311-cli)
    - [3.11.1 Enabled/disable WJH feature](#3111-enableddisable-wjh-feature)
    - [3.11.2 Config WJH global parameters](#3112-config-wjh-global-parameters)
    - [3.11.3 Raw](#3113-raw)
    - [3.11.4 Aggregated](#3114-aggregated)
  - [3.12 WJH daemon](#312-wjh-daemon)
    - [3.12.1 Push vs Pull](#3121-push-vs-pull)
    - [3.12.2 WJH daemon packet parsing library](#3122-wjh-daemon-packet-parsing-library)
    - [3.12.3 WJH communication with CLI](#3123-wjh-communication-with-cli)
    - [3.13 WJH debug dump](#313-wjh-debug-dump)
- [4 Flows](#4-flows)
  - [4.1 wjhd init flow](#41-wjhd-init-flow)
  - [4.2 wjhd channel create and set flow](#42-wjhd-channel-create-and-set-flow)
  - [4.3 wjhd channel remove flow](#43-wjhd-channel-remove-flow)
  - [4.4 wjhd deinit flow](#44-wjhd-deinit-flow)
- [5 Warm Boot Support](#5-warm-boot-support)
  - [5.1 System level](#51-system-level)
  - [5.2 Service level](#52-service-level)
- [6 Fast Boot Support](#6-fast-boot-support)
- [7 Unit testing](#7-unit-testing)
- [8 Open Questions](#8-open-questions)

# List of Tables
* [Table 1: Abbreviations](#definitionsabbreviation)
* [Table 2: Raw Channel Data](#raw-channel-1)
* [Table 3: Aggregated Channel Data](#aggregated-channel)

# List of Figures
* [Init flow](#41-wjhd-init-flow)
* [Channel create and set flow](#42-wjhd-channel-create-and-set-flow)
* [Channel remove flow](#43-wjhd-channel-remove-flow)
* [Deinit flow](#44-wjhd-deinit-flow)

# Revision
| Rev | Date     | Author          | Change Description                 |
|:---:|:--------:|:---------------:|------------------------------------|
| 0.1 | 01/20    | Stepan Blyschak | Initial version                    |
| 0.2 | 01/28    | Stepan Blyschak | Debug mode approaches              |
| 0.2 | 01/30    | Stepan Blyschak | Addressed design review comments   |

# About this Manual
This document provides an overview of the implementation and integration of What Just Happened feature in SONiC.

# Scope
This document describes the high level design of the What Just Happened feature in SONiC.

# Definitions/Abbreviation
| Abbreviation  | Description                               |
|---------------|-------------------------------------------|
| SONiC         | Software for open networking in the cloud |
| API           | Application programming interface         |
| WJH           | What Just Happened                        |
| SAI           | Switch Abstraction Interface              |
| SDK           | Software Developement Kit                 |


# 1. Overview

What Just Happened feature provides the user a way to debug a network problem by providing the as much information information about dropped packets due to L1, L2, L3, ACL or buffer related drop reasons. 

# 2 Requirements Overview

## 2.1 Functional requirements

What Just Happened feature in SONiC should meet the following high-level functional requirements:

- WJH feature provides functionality as a seperate docker container built in Mellanox SONiC image
- WJH feature is optional in SONiC, thus can be enabled or disabled per user request at runtime
- Dropped packets information is provided either as raw packet or aggregate counter which is based on a key, for example 5-tuple for IP packets, each with a specific drop reason message
- The following list of drop reason groups are be supported: L1, Buffer, L2, L3, Tunnel, ACL
- WJH can work in one of two modes:
  - User can retrive the data through CLI with an option to write dropped packets into a PCAP file, this mode is called debug mode. Debug mode should not consume Redis resources therefore a different IPC should be used
  - Data is sent to remote collector either by software or through a direct channel without control plane overhead
- Provide CLI for configuring WJH channels and collector

Phase 1
- WJH debug mode
- Configuring WJH channels is not supported, configuration is pre-defined
- Raw and aggregated buffer drops are not supported on SPC 1

# 3 Modules design

## 3.1 WJH build and runtime dependencies

To enable aggregated mode which is built on eBPF feature in kernel, the following requirements have to be met

- Linux Kernel
  - CONFIG_BPF=y
  - CONFIG_BPF_SYSCALL=y
  - CONFIG_DEBUG_FS=y
  - CONFIG_PERF_EVENTS=y
  - CONFIG_EVENT_TRACING=y
- LLVM
  - llvm >= 3.7.1 (llc with bpf target support)
  - clang >= 3.4.0
- libelf1 [runtime dependency]
- debugfs mounted in container

*NOTE:* Make sure WJH library compiled with *-DWJH_EBPF_PRESENT*

## 3.2 WJH docker container in SONiC

A new docker image will be built for mellanox target called "docker-wjh" and included in Mellanox SONiC build by default.<p>
Build rules for WJH docker will reside under *platform/mellanox/docker-wjh.mk* while actual Dockerfile.j2 and other sources under *platform/mellanox/docker-wjh/* directory.

```
admin@sonic:~$ docker ps
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS              PORTS               NAMES
...
2b199b2309f9        docker-wjh:latest                    "/usr/bin/supervisord"   17 hours ago        Up 11 hours                             what-just-happened
```

* SDK Unix socket needs to be mapped to container at runtime
* SDK /dev/shm needs to be mapped to container at runtime
* *debugfs* mounted inside container (requires RW access to */sys/kernel/debug/*)
* */var/log/mellanox/* mounted inside container (used for pcap files)
* */var/run/wjh/* mounted inside container (used for Unix domain socket)

*NOTE*: In order to improve build speed and reduce SONiC image size an intermediate SDK docker image can be created and *syncd*, *pmon* and *wjh* docker may built on top of SDK intermediate docker image.

## 3.3 WJH service in SONiC

A corresponding service file has to be created to manage the lifetime and dependencies of WJH docker container.

WjH service, as an ASIC-dependent service, should have a hard requirement for syncd service, so it will be started, restarted, stopped when syncd service is.

The following systemd unit configurations and dependencies will be set:

```
Requiers=database.service syncd.service
After=database.service syncd.service
Before=ntpconfig.service
StartLimitIntervalSec=1200
StartLimitBurst=3
```

The following service settings will be set:

```
Restart=always
RestartSec=30
```

Thus, the service is configured to be restarted in case of a failure. If the service was autorestarted 3 times within 20 minutes the service will fail.

## 3.4 WJH feature table

Community [introduced](https://github.com/Azure/SONiC/blob/master/doc/Optional-Feature-Control.md) a way to enable/disable optional features at runtime and provided a seperate **FEATURE** table in CONFIG DB.

```
admin@sonic:~$ show features 
Feature               Status
-------------------   --------
telemetry             enabled
what-just-happened    enabled
```

```
admin@sonic:~$ sudo config feature what-just-happened [enabled|disabled]
```

The above config command translates into:

enabled command:
```
sudo systemctl enable what-just-happened
sudo systemctl start what-just-happened
```

disabled command:
```
sudo systemctl stop what-just-happened
sudo systemctl disable what-just-happened
```

By default WJH feature will be disabled; this configuration will be put in */etc/sonic/init_cfg.json* at build time

## 3.5 WJH reasons and severity configuration file

WJH requires a XML formatted file on initialization. The file columns are:
```
ID              : drop reason ID;
REASON          : human readable message describing the reason;
SEVERITY        : severity of the drop;
PROPOSED ACTION : proposed action message (if there is);
```

SONiC has to provide a default XML. Default XML will be built into image and put under */etc/sonic/wjh/wjh.xml*.
The path /etc/sonic is mapped to every container in RO mode, so WJH daemon will be able read it.

By default, in case this file does not exists, WJH library will be initialized to use default XML provided by WJH library.

## 3.6 WJH and debug counters

Debug counters in SAI and WJH are mutually exclusive. WJH works directly through SDK, so SAI cannot know when it’s enabled/disabled.

There are going to be two checks:

1. Upon WJH container start in the start script, check for debug counter entries in DB **DEBUG_COUNTER** table.<br>
   If found - log the error, abort the operation. Service will not be attempted to be restarted by systemd.
2. To support WJH enable/disable at runtime, debug counters configuration producers have to be notified that debug counters became unavailable.<br>
   The following option to implement this may be considered: once started, WJH has to reset capability table, speicifically
   it has to cleanup **DEBUG_COUNTER_CAPABILITIES** table populated by orchagent, in order for **DEBUG_COUNTER** table producers to know that debug counters became unavailable.
   On teardown, WJH has to restore **DEBUG_COUNTER_CAPABILITIES** table to the original state.
   Prefered to not put this logic in daemon part but in the service start script by saving table into json file under either */tmp/* or */var/run/wjh/*. ```ExecStopPost=``` action will ensure the content of the table will be restored even on unexpected failures.

## 3.7 Config DB

### 3.7.1 WJH table schema

```
; Describes global configuration for WJH
key          = WJH|global
nice_level   = integer ; wjh daemon process "nice" value
pci_bandwith = percentage ; percent of PCI bandwidth used by WJH, range [0-100]
mode         = debug ; only debug mode will be availabe right now
```

### 3.7.2 WJH table defaults

```json
{
  "WJH": {
    "global": {
      "nice_level": "1",
      "pci_bandwidth": "50",
      "mode": "debug"
    }
  }
}
```

Default configuration is stored in */etc/sonic/init_cfg.json*. It's generation can be extended to generate WJH table for Mellanox build.
Porcesses inside WJH will have their own defaults, which will be the same.

## 3.8 WJH default configuration for debug mode

At phase 1 the following default channel configuration will be set:

* Forwarding group
  * Drop reasons - L2, L3, Tunnel
  * WJH_USER_CHANNEL_CYCLIC_AND_AGGREGATE
* ACL group
  * Drop reasons - ACL
  * WJH_USER_CHANNEL_CYCLIC_AND_AGGREGATE
* Buffer [SPC2]
  * Drop reasons - buffer
  * WJH_USER_CHANNEL_CYCLIC_AND_AGGREGATE
* L1
  * Drop reasons - L1
  * WJH_USER_CHANNEL_CYCLIC_AND_AGGREGATE

The above configuration is default for debug mode. For now, no option to change it.

To support both raw data and aggregate data in CLI WJH channels are created in WJH_USER_CHANNEL_CYCLIC_AND_AGGREGATE mode.

We'll use WJH default ring buffer and aggregate buffer size which is 1024.

Buffer group is only supported on SPC2 for now. Support for SPC1 will require recirculation port.

## 3.9 WJH provided data

### 3.9.1 Raw Channel

| Data               | Buffer   | L1       | L2       | Router   | Tunnel   | ACL      |
|:------------------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| packet             |    x     |          |    x     |    x     |    x     |    x     |
| packet size        |    x     |          |    x     |    x     |    x     |    x     |
| ingress port       |    x     |    x     |    x     |    x     |    x     |    x     |
| egress port        |    x*    |          |          |          |          |          |
| TC                 |    x*    |          |          |          |          |          |
| drop reason        |    x     |          |    x     |    x     |    x     |    x     |
| timestamp          |    x     |    x     |    x     |    x     |    x     |    x     |
| ingress LAG        |          |          |    x     |    x     |    x     |    x     |
| original occupancy |    x*    |          |          |          |          |          |
| original latency   |    x*    |          |          |          |          |          |
| bind point         |          |          |          |          |          |    x     |
| rule ID            |          |          |          |          |          |    x     |
| acl name           |          |          |          |          |          |    x     |
| rule name          |          |          |          |          |          |    x     |

- \* SPC2 only

### 3.9.2 Aggregated channel

| Data               | Buffer   | L1       | L2       | Router   | Tunnel   | ACL      |
|:------------------:|:--------:|:--------:|:--------:|:--------:|:--------:|:--------:|
| ingress port       |          |    x     |          |          |          |          |
| 5 tuple            |    x     |          |    x     |    x     |    x     |    x     |
| drop reason        |    x     |          |    x     |    x     |    x     |          |
| aggregate time     |    x     |    x     |    x     |    x     |    x     |    x     |
| flows num          |          |          |          |          |          |    x*    |
| rule ID            |          |          |          |          |          |    x     |
| acl name           |          |          |          |          |          |    x     |
| rule name          |          |          |          |          |          |    x     |
| smac               |          |          |    x     |          |    x     |          |
| dmac               |          |          |    x     |          |    x     |          |
| is port up         |          |    x     |          |          |          |          |
| port down reason   |          |    x     |          |          |          |          |
| description        |          |    x     |          |          |          |          |
| state change count |          |    x     |          |          |          |          |
| symbol error count |          |    x     |          |          |          |          |
| crc error count    |          |    x     |          |          |          |          |

- \* SPC2 only

## 3.10 Mapping SDK IDs to SONiC IDs/object names

### 3.10.1 SDK logical port ID/netdev IF_INDEX to SONiC port name mapping

For ports WJH library has to options:
- WJH_INGRESS_INFO_TYPE_LOGPORT
- WJH_INGRESS_INFO_TYPE_IF_INDEX

First one returns an SDK logical port ID, while second one returns the kernel interface index if such exits.


Using first one, requires WJH daemon to generate SAI OID based on SDK logical port ID and map it to SAI redis OID using *RIDTOVID* map in ASIC DB and then map it to SONiC port name using *COUNTERS_PORT_NAME_MAP*.


For SONiC use case, since SONiC creates host interface for every physical port, it is preferable to use WJH_INGRESS_INFO_TYPE_IF_INDEX since it is easier to map it to SONiC port name which is the same as host interface net device name.
 
### 3.10.2 SDK logical LAG ID to SONiC LAG name mapping

There is no table in DB that will map SONiC LAG name to SAI redis virtual OID. Without SONiC+ infrastructure it may be tricky to map SDK LAG ID to SONiC LAG name, while it is still possible. Since WJH library provides ingress logical port ID and ingress LAG ID in case ingress port is a LAG member it is possible to map it to the LAG uniquely by LAG member port. However, in order to simplify developement the LAG ID data provided by WJH library will be ignored. Besides, it is not worth the required effort, since user can map ingress port to LAG on his own.

### 3.10.3 SDK ACL rule ID to SONiC ACL rule name mapping

Complex to do such mapping without SONiC+, therefore this data will be ignored for now.

ACL name and Rule name can be used as description for drop;

e.g.:

```
admin@sonic:~$ show acl rule
Table    Rule            Priority  Action    Match
-------  ------------  ----------  --------  ------------------
DATAACL  RULE_1              9999  DROP      SRC_IP: 1.1.1.1/32
```

Generated ACL rule name by WJH library is following:
```
Priority[10001];KEY[SIP: 1.1.1.1/255.255.255.255];ACTION[COUNTER: COUNTER_ID = 9972];ACTION[FORWARD: FORWARD_ACTION = DISCARD]
```

While such description is not aligned with SONiC as it comes from SDK level it may be still usefull for user.
In the future, we can use ACL rule ID and map it to SONiC rule name, which is "RULE_1" from table "DATAACL" in this case. However, such mapping requires SONiC+ infrastructure.

## 3.11 CLI

Since user channel polling is read and clear, having two WJH clients, e.g. user that debugs drops and streaming collector, results in a problem when one client can clear the raw infromation or aggregated counters for another one. Thus, first approach that is considered is to disable streaming and debug mode via "mode" configuration knob.

Following mode design stands on assumptions:
  - no periodic polling/no streaming
  - one debug user

User will have to switch to debug mode if not already in debug mode:

```
admin@sonic:~$ sudo config what-just-happened mode debug
```

### 3.11.1 Enabled/disable WJH feature

Already implemented as part of "optional features" feature:

Config CLI:
```
admin@sonic:~$ config feature what-just-happened [enabled|disabled]
```
Show CLI:
```
admin@sonic:~$ show features 
Feature               Status
-------------------   --------
telemetry             enabled
what-just-happened    enabled
```

### 3.11.2 Config WJH global parameters

Global parameters will be create only.

### 3.11.3 Raw

- Whenever user requests raw packets via CLI, WJH daemon will pull packets from WJH channel. No periodic polling is done, since streaming is disabled
- User can optionally pass one or multiple drop reason groups as argument to CLI to filter out drops user is not interested in
- User can pass *--pcap* option to write the drop data to pcap file

```
admin@sonic:~$ show what-just-happened [l1|acl|buffer|forwarding]                  # to show raw events
admin@sonic:~$ show what-just-happened acl --pcap # to show all ACL related raw events and create a pcap file
```

The printed table will include a timestamp, source port, destination port, VLAN ID, source MAC, destination MAC, ethernet type, source IP and port, destination IP and port, IP protocol, drop group, severity and drop reason with recommented action if there is such.
If any of the field is not available it will be shown as "N/A". While most of the data consumes constant amount of characters to be printed, drop reason message will be printed on next line in same column.

Expected CLI example output:
```
admin@sonic:~$ show what-just-happened forwarding --pcap

Pcap file generated at /var/log/mellanox/wjh/forwarding_2020_01_31_01_40_02.pcap

#      Timestamp                sPort       dPort    VLAN    sMAC               dMAC               EthType    Src IP:Port    Dst IP:Port    IP Proto    Drop Group    Severity    Drop reason - recomended action
----   -----------------------  ----------  -------  ------  -----------------  -----------------  ---------  -------------  -------------  ----------  ------------  ----------  ---------------------------------
1      2019/07/18 11:06:31.277  Ethernet24  N/A      N/A     7C:FE:90:6F:39:BB  00:00:00:00:00:02  IPv4       1.1.1.1:171    127.0.0.1:172  TCP         L3            Critical    Destination IP is loopback
```

### 3.11.4 Aggregated

- Whenever user requests aggregated counters via CLI, WJH daemon will pull counters from WJH library
- User can optionally pass one or multiple drop reason groups as argument to CLI to filter out drops user is not interested in

```
admin@sonic:~$ show what-just-happened buffer aggregated # show aggregated counters
admin@sonic:~$ show what-just-happened L1 aggregated # show all aggregated counters for L1 group
```

Expected CLI output:
```
admin@sonic:~$ show what-just-happened buffer aggregated
Sample window: 2019/07/18 11:06:31 - 2019/07/18 11:06:36
#      sMac          dMac           Src IP:Port    Dst IP:Port    IP Proto    Drop Group    Count    Severity    Drop reason - recomended action
----   ------------- -------------  -------------  -------------  ----------  ------------  ------   ----------  ---------------------------------
1      N/A           N/A            1.1.1.1:171    127.0.0.1:172  TCP         L3            532      Critical    WRED
2      N/A           N/A            N/A            N/A            N/A         Buffer        1542     Critical    WRED
```
*NOTE*: A sample window is between two CLI invocations.

Same for L1 counters, however L1 counter table header is different:

```
admin@sonic:~$ show wjh L1 aggregated
Sample Window : 2019/07/18 11:06:31 - 2019/07/18 11:06:36 
#   Port         State      Down Reason                        State Change  Symbol Error     FCS Error  Transceiver Overheat
--- -----------  --------   -------------------------------    ------------  --------------   ---------- --------------------
1   Ethernet4    Down       Logical mismatch with peer link    1             4                8          2
2   Ethernet20   Down       Link training failure              1             0                0          0
3   Ethernet0    Down       Port admin down                    2             2                45         1 

```

## 3.12 WJH daemon

### 3.12.1 Push vs Pull

In push mode, the WJH library periodically queries the dropped packets or statistics and deliver them via user callbacks.
In pull mode, the WJH library stops the periodical query. The dropped packets or statistics can be delivered via user callbacks
which are explicitly triggered by API *wjh_user_channel_pull*. Please note that the user callbacks will be executed in the same
context/thread of the caller of *wjh_user_channel_pull*.

Pull mode is prefered here because no syncronization with WJH library thread will be required.

### 3.12.2 WJH daemon packet parsing library

CLI requires source, destination MAC, Ethernet type, source, destination IP:port and IP protocol on raw channel. A packet parsing library can be used:

* https://github.com/mfontanini/libtins

### 3.12.3 WJH communication with CLI

In order to not produce additional load in Redis DB or bringing another Redis instance specifically for WJH another IPC mechanism will be used.
A suggested alternative is a Unix domain socket. It may be placed under */var/run/wjh/wjh.sock* wich will be mapped to WJH container.

On CLI request WJH daemon will pull the related WJH user channel and write back a complete table output.

CLI/WJH daemon communication protocol will be text based in the following format:

```
type    = string ; either raw or aggregated
group   = string ; [l1|buffer|acl|forwarding]
pcap    = string ; string representing boolean whether daemon needs to write a pcap file
```

For example, raw L2 and L3 packets

```
request=pull type=raw group=forwarding pcap=True
```

While the daemon reply is as simple as text output of the table.

Since the design is focused on one CLI client, only one connection will be handled at the time.

A considerable timeout has to be set on socket so that send/recv will not block CLI or daemon if one side unexpectedly terminates.

### 3.13 WJH debug dump

Techsupport dump should include the WJH debug dump data.

As WJH will be a SONiC out of tree addon in the future, the mechanism to generate WJH dump has to be generalized, however, this is out of scope for this document, so *generate_dump.sh* script has to be updated with a condition - if it is a mellanox platform and wjh feature is enabled then request the WJH daemon to generate WJH dump and put the file into dump archive.

A small utility program inside a docker image will do the request - *wjh_generate_dump* by using the same mechanism as used to request daemon to pull WJH channel.

```
request=dump
```

A reply from daemon will be a text of debug dump from WJH. The text is the output of *wjh_dbg_generate_dump* API.

# 4 Flows

## 4.1 wjhd init flow

![wjhd init](/doc/wjh/wjhd_init.svg)

## 4.2 wjhd channel create and set flow

![wjhd create_set](/doc/wjh/wjhd_channel_create_set.svg)

*NOTE*: channel create/set/remove actions are not planned for phase 1, however the flow provides the WJH API usage flow. 

## 4.3 wjhd channel remove flow

![wjhd removal](/doc/wjh/wjhd_channel_remove.svg)

*NOTE*: channel create/set/remove actions are not planned for phase 1, however the flow provides the WJH API usage flow.

## 4.4 wjhd deinit flow

![wjhd deinit](/doc/wjh/wjhd_deinit.svg)

*NOTE*: It is good to implement *deactive_cb*, so that if other docker takes control of WJH library (like neo, or customer specific container), the wjh will shutdown and print a log message that another application took control.

# 5 Warm Boot Support

## 5.1 System level

WJH service will be shutdown prior to syncd as because of systemd dependencies, however ISSU start isn't called in syncd stop action, so in the *warm-reboot* script it needs to be stopped explicitely before ISSU start which is called on syncd's request for warm pre shutdown action. So before system goes kexec WJH will be able to shutdown properly and clean all resources. 

## 5.2 Service level

WJH service level warm restart has no inpact on traffic.

Other services warm restart (like swss, teamd) won't trigger WJH restart. Only syncd service restart triggers WJH service restart which makes sense because WJH depends on sxkernel service, however syncd service warm restart is not supported in SONiC in any case.

It is required to test the new service start impact on system performance during system *warm* startup to ensure no additional delay is added in control plane restoration time.

# 6 Fast Boot Support

No special support is required. However, it is required to test the new service start impact on system performance during system *fast* startup.

# 7 Unit testing

# 8 Open Questions

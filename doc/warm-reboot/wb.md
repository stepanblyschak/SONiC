<!-- omit in toc -->
# Warm Reboot

<!-- omit in toc -->
## Table of Content

- [LibSAI](#libsai)
- [LibSAI/Application flow](#libsaiapplication-flow)
- [Persistent Location](#persistent-location)
- [Database](#database)
- [Linux Restart](#linux-restart)
- [Application warm restart configuration and state](#application-warm-restart-configuration-and-state)
- [SYNCD](#syncd)
  - [SYNCD in Warm Boot Flow](#syncd-in-warm-boot-flow)
  - [SYNCD in Fast Fast Boot](#syncd-in-fast-fast-boot)
- [SWSS common APIs](#swss-common-apis)
- [Orchagent](#orchagent)
- [Neighbors](#neighbors)
- [Teamd](#teamd)
- [BGP](#bgp)
- [Management plane](#management-plane)
- [Control plane assistant (CPA)](#control-plane-assistant-cpa)
- [Warm Boot finalizer](#warm-boot-finalizer)
- [System flow](#system-flow)

<!-- omit in toc -->
### Revision

|  Rev  |  Date   |      Author      | Change Description                         |
| :---: | :-----: | :--------------: | ------------------------------------------ |
|  0.0  |  2018   |       MSFT       | Initial                                    |
|  0.1  | 09/2020 | Stepan Blyshchak | Update and align to current implementation |

### Scope  

This document describes warm-reboot functionality in SONiC.

### Definitions/Abbreviations 

| **Abbreviation** | **Definition**                        |
| ---------------- | ------------------------------------- |
| SONiC            | Software for Open Networking in Cloud |
| DB               | Database                              |
| API              | Application Programming Interface     |
| SAI              | Switch Abstraction Interface          |
| CPA              | Control plane assistant               |

### Overview

The goal of SONiC warm reboot is to be able to restart and upgrade SONiC software without impacting the data plane.
Warm restart of each individual *feature* is also part of the goal.

#### Use cases

#### In-Service restart

The mechanism of restarting a component without impact to the service. This assumes that the software version of the component has not changed after the restart.
There could be data changes during restart window:
- new/stale route
- port state change
- fdb change 

Component here could be the whole SONiC system or just one or multiple of the *feature*s running in SONiC.

#### In-Service upgrade

The mechanism of upgrading to a newer version of a component without impacting the service.

Component here could be the whole SONiC system or just one or multiple of the dockers running in SONiC.

#### Un-Planned restart

It is desired for all network applications and orchagent to be able to handle unplanned restart, and restore gracefully.
It is not a requirement on syncd and ASIC/LibSAI due to dependency on ASIC processing.

#### Per-component examples

#### BGP restart

After BGP restart, new routes may be learned from BGP peers and some routes which had been pushed down to APP DB and ASIC may be gone.
The system should be able to clear the stale route from APP DB down to ASIC and program the new route.

#### swss restart

After swss restart, all the port/LAG, vlan, interface, ARP and route data should be restored from CFG DB, APP DB, Linux Kernel and other reliable sources.
There could be port state, ARP, FDB changes during the restart window, proper sync processing should be performed.

#### syncd restart

The restart of syncd should leave data plane intact. After restart, syncd resumes control of ASIC/LibSAI and communication with swss.
All other functions which run in syncd should be restored too.

#### teamd restart

The restart of teamd should not cause link flapping or any traffic loss. All lags at data plane should remain the same.

### Requirements

- Keep data plane running while software restarts, in a *happy* path (no network state changes occurred during the restart window) none of the data packets are dropped nor black holed.
- In case the network state changes have occurred during the restart window, the software and hardware states have to become in sync with the network state eventually.
- The control plane downtime is not exceeding the 90 seconds downtime limit. 90 seconds is a strict requirement for teamd restoration to keep LAGs running LACP active on the peers.
- The management plane downtime has relaxed requirements but usually management services (telemetry, SNMP, etc.) restart in less than 5 minutes.

### High-Level Design

#### ASIC

SONiC supports two kinds of hardware warm restart designs.

First is more traditional in which the software restarts without resetting the ASIC and restores its state
from a dump made before restart so that in the end the software and the ASIC are in sync.

Another flavor of hardware implementation of warm restart includes logical division of the ASIC TCAM into two banks, one of which is
the currently active one while another is not in use and reserved to be used in the *new-life*. After software restarts it is
required to reprogram the second bank while the first bank is still active and once the configuration is finished atomically switch
to the second bank wiis dInthout dropping a single packet. This, however, means that the hardware can use only half of TCAM resources when
such capability is enabled. Also, some counters do not persist across warm boot in such case due to recreation of those counters.

The two banks approach is known as **fast-fast** boot approach in SONiC.

Whatever first or second approach is chosen by the ASIC vendor it is just an implementation detail of how warm reboot is implemented for a
particular hardware hidden in the lower layers of SONiC (syncd and below). From the user's point of view it is the same "warm-reboot".

Nvidia platform is currently the only one implementing **fast-fast** boot approach.

## LibSAI

SAI API has support for planned warm restart. The API provides a set of attributes and init profile values that are used to perform warm restart.

| SAI Attribute                   | SAI Data Type | Description                                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| SAI_SWITCH_ATTR_RESTART_WARM    | boolean       | Indicates controlled warm restart to the LibSAI implementation. <br/> This hint is set as part of the shutdown sequence, before boot.                                                                                                                                                                                                                        |
| SAI_SWITCH_ATTR_PRE_SHUTDOWN    | boolean       | Indicates controlled switch pre-shutdown as first step of warm shutdown. <br/> The scope of pre-shutdown is to backup SAI/SDK data, but leave CPU port active for some final control plane traffic to go out. <br/> This attribute has special meaning for fast-fast boot and indicates to LibSAI implementation to initialize the fast-fast boot procedure. |
| SAI_SWITCH_ATTR_FAST_API_ENABLE | boolean       | Indicates the end of the fast-fast boot operation when set to False. <br/> The LibSAI will switch the banks in a non-disruptive manner. <br/> Only required to support fast-fast boot implementation.                                                                                                                                                        |

<!-- omit in toc -->
###### Table 1. SAI Attributes 

| SAI Profile Value            | SAI Data Type | Description                                                                                                    |
| ---------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------- |
| SAI_KEY_WARM_BOOT_WRITE_FILE | string        | The filepath that SAI will use when generating the dump to restore from later on in the *new-life*.            |
| SAI_KEY_WARM_BOOT_READ_FILE  | boolean       | The filepath where generated SAI dump is located that SAI will use when restoring its state in the *new-life*. |
| SAI_BOOT_TYPE                | integer       | Indicates to SAI which boot type it is. 0 - cold boot, 1 - fast boot, 2 - warm boot.                           |

<!-- omit in toc -->
###### Table 2. SAI Profile keys 

## LibSAI/Application flow

This sections describes the general SAI client application flows to perform warm/fast-fast restarts.

The below shows the shutdown process starting from a regular cold booted SAI and is applicable to both warm and fast-fast restarts:

```mermaid
sequenceDiagram
    App->>LibSAI: sai_api_initialize() with <br/>SAI_KEY_WARM_BOOT_WRITE_FILE=<dump-filepath> <br/> SAI_BOOT_TYPE=0 (cold boot)
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App->>LibSAI: sai_switch_api->create_switch()
    activate App
    activate LibSAI
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    Note right of App: Regular ASIC configuration through SAI API
    App-->LibSAI: 
     Note right of App: App decides to perform planned warm restart
    App->>LibSAI: sai_switch_api->set_switch_attribute() <br/> SAI_SWITCH_ATTR_PRE_SHUTDOWN=true
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App->>LibSAI: sai_switch_api->set_switch_attribute() <br/> SAI_SWITCH_ATTR_RESTART_WARM=true
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App->>LibSAI: sai_switch_api->remove_switch()
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS 
    deactivate LibSAI
    deactivate App
```
<!-- omit in toc -->
###### Figure 1. SAI warm shutdown procedure

Then, after restart the initialization part requires to pass SAI_BOOT_TYPE=2 into the SAI profile map to let SAI know to restore from a dump.
After switch is created all ASIC related configurations (with exception of host interfaces, etc.) are available as they were prior to boot:

```mermaid
sequenceDiagram
    App->>LibSAI: sai_api_initialize() with <br/>SAI_KEY_WARM_BOOT_READ_FILE=<dump-filepath> <br/> SAI_BOOT_TYPE=2 (warm boot)
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App->>LibSAI: sai_switch_api->create_switch()
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
     Note left of LibSAI: LibSAI state is restored and in sync with ASIC
     Note right of App: App continues to operate in a regular mode. <br/> App is now required to propagate the <br/> differences occurred in the network state during <br/> restart window down to the ASIC. <br/> E.g. Delete stale routes, program newly advertised routes, etc.
```

<!-- omit in toc -->
###### Figure 2. SAI warm startup procedure

In the fast-fast boot SAI is also initialized with SAI_BOOT_TYPE=2, but requires to re-play the configuration into the new bank.
One configuration is done, application should set switch attribute SAI_SWITCH_ATTR_FAST_API_ENABLE=false to signal LibSAI to switch to the new bank:

```mermaid
sequenceDiagram
    App->>LibSAI: sai_api_initialize() with <br/>SAI_KEY_WARM_BOOT_READ_FILE=<dump-filepath> <br/> SAI_BOOT_TYPE=2 (warm boot)
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App->>LibSAI: sai_switch_api->create_switch()
    activate LibSAI
    activate App
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
     Note left of LibSAI: LibSAI requires to replay the configuration on platform which implement fast-fast boot
     Note right of App: Regular ASIC configuration through SAI API
    App->>LibSAI: sai_switch_api->set_switch_attribute() <br/> SAI_SWITCH_ATTR_FAST_API_ENABLE=false
    activate LibSAI
    activate App
     Note right of App: App tells LibSAI that new configuration is pushed to the hardware and LibSAI can switch to the new bank.
    LibSAI-)App: SAI_STATUS_SUCCESS
    deactivate LibSAI
    deactivate App
    App-->LibSAI: 
     Note right of App: App continues to operate in a regular mode. Warm restart has finished
```

<!-- omit in toc -->
###### Figure 3. SAI fast-fast startup procedure

## Persistent Location

A lot of components will require to save a dump of the state into a persistent location. In particular LibSAI/SDK dumps and SONiC DB dump.
In SONiC this location is */host/warmboot/* directory.
This location does not only persist across SONiC restarts but also persists across SONiC upgrades allowing the new SONiC image version to warm boot
from a dump saved by the old SONiC image version.


## Database

Database is a crucial component of the SONiC switch software. It does not support warm restarting individually, only together with all other SONiC services.
Database is the first service that starts in SONiC boot process. It is required to distinguish between different boot types and act respectively.

In shutdown phase the dump of the database is performed and the dump is saved into a persistent location - */host/warmboot/dump.rdb*.

In **fast-fast** boot scenario the ASIC DB is flushed since we are going to reprogram the a non active bank in the ASIC as well as some of the counters are not persistent across reboots
and will be recreated, like flow counters, ACL counters, etc., while other counters will be just repopulated after system starts, like port counters.

For both types of hardware implementations of warm reboot, most of the content of STATE DB is flushed except of few tables which are used to correctly restore the state after reboot.
What kind of tables are dumped from the STATE DB depends on the application and its restoration logic. For example:
- orchagent requires to know the pre-boot mirror session state (whether active or not) to understand whether to program the mirror session on the hardware or not
- WARM_RESTART_TABLE has to be persistent by its definition.

At boot, the *database.sh* start script determines wether the boot type is either *warm* or *fast-fast* and if so and the dump exists, loads the data from */host/warmboot/dump.rdb*, otherwise falls back to cold boot. *db-migrator.py* is executed to make sure the data in DB is aligned to the latest DB schema.

## Linux Restart

Linux already supports warm reboot by *kexec* kernel feature.

*kexec*, abbreviated from kernel execute and analogous to the Unix/Linux kernel call exec, is a mechanism of the Linux kernel that allows booting of a new kernel from the currently running one.
Essentially, kexec skips the bootloader stage and hardware initialization phase performed by the system firmware (BIOS or UEFI), and directly loads the new kernel into main memory and starts
executing it immediately. This avoids the long times associated with a full reboot, and can help systems to meet high-availability requirements by minimizing downtime.

SONiC warm reboot leverages *kexec* Linux kernel functionality to perform warm reboot.

SONiC appends an additional kernel parameter *SONIC_BOOT_TYPE* and sets its value to either "warm" or "fastfast" depending on the kind of HW implementation the platform supports. Using that parameter in the *new-life* the software knows that the warm reboot was done by looking at */proc/cmdline* and act respectively, though this kernel parameter should be used only once by *database.sh* to known whether to load the dump of the DB. Later, applications can read ```WARM_RESTART_TABLE``` to figure out the boot type.

The relevant source code - [warm-reboot](https://github.com/sonic-net/sonic-utilities/blob/master/scripts/fast-reboot).


## Application warm restart configuration and state

At start of the warm boot procedure most of the daemons set their state to *initialized*.

Once the application has read its configuration and pre-boot state previously uploaded to the database its state transitions into *restored*.

Once the *new-life* state is compared with the *old-life* state and the differences are reprogrammed on the lower layers its state transitions into *reconciled*
and the warm-restart job of this application is formally done, the application continues to operate in a normal way.

Once, the warm restart is executed again, the state transition *reconciled* -> *initialized* happens and the flow repeats.

Some application skip *restored* state and directly transitions into *reconciled* state from *initialized*.

The state machine is described as follows:

```mermaid
stateDiagram
    [*] --> initialized

    initialized --> restored
    restored --> reconciled
    reconciled --> initialized
    initialized --> reconciled
```
<!-- omit in toc -->
###### Figure 4. Application warm restart states

<!-- omit in toc -->
### WARM_RESTART_ENABLE_TABLE

In order for all the other services to understand that the system is in transient state performing warm restart a special configuration table exists in CFG DB
named *WARM_RESTART_ENABLE_TABLE*:

```abnf
;Stores system warm start and feature warm start enable/disable configuration
key                 = WARM_RESTART_ENABLE_TABLE:name ; name is the name of SONiC feature or "system" for global configuration.

enable              = "true" / "false"  ; Default value as false.
                                        ; If "system" warm start knob is true, docker level knob will be ignored.
                                        ; If "system" warm start knob is false, docker level knob takes effect.
```

The warm boot aware daemons are reading the CFG DB table and understand wether they are in a warm start process by looking at either the name of the feature they are part of or "system" key in *WARM_RESTART_ENABLE_TABLE*.

<!-- omit in toc -->
### WARM_RESTART_TABLE

They are recording the state of the warm start procedure in the STATE DB table called *WARM_RESTART_TABLE*:

```abnf
;Stores application warm start status, persistent across warm reboots.
key             = WARM_RESTART_TABLE|process_name         ; process_name is a unique process identifier.
                                                          ; with exception of 'warm-shutdown' operation.
                                                          ; 'warm-shutdown' operation key is used to
                                                          ; track warm shutdown stages and results.
                                                          ; Added to this table to leverage the existing
                                                          ; "show warm-restart state" command.

restore_count   = 1*10DIGIT                               ; a value between 0 and 2147483647 to keep track
                                                          ; of the number of times that an application has
                                                          ; 'restored' its state from its associated redis
                                                          ; data-store; which is equivalent to the number
                                                          ; of times an application has iterated through
                                                          ; a warm-restart cycle.

state           = "initialized" / "restored" / "reconciled"  ; initialized: initial FSM state for processes
                                                             ; with warm-restart capabilities turned on.
                                                             ;
                                                             ; restored: process restored the state previously
                                                             ; uploaded to redis data-stores.
                                                             ;
                                                             ; reconciled: process reconciled 'old' and 'new'
                                                             ; state collected in 'restored' phase. Examples:
                                                             ; dynamic data like port state, neighbor, routes
                                                             ; and so on.
```

## SYNCD

Syncd start script inside syncd container *syncd_start.sh* determines the reboot cause and start syncd with either ```-t warm``` or ```-t fast-fast``` depending on *SONIC_BOOT_TYPE* kernel boot parameter.

### SYNCD in Warm Boot Flow

Essentially there will be two views created for warm restart. The current view represents the ASIC state before shutdown, temp view represents the new intended ASIC state after restart.
Based on the SAI object data model, each view is a directed acyclic graph, all objects are linked together.

<!-- omit in toc -->
#### View comparison logic

Utilizing the meta data of object, with those invariants as anchor points, for each object in temp view, it starts as root of a tree and go down to all layer of children node until leaf to find best match.
If no match is found, the object in temp view should be created, it is object CREATE operation.
If best match is found, but there is attributes different between the object in temp view and current view, SET operation should be performed.
Exact match yields Temp VID to Current VID translation, which also paves the way for upper layer comparison.
All objects in current VIEW which have reference count 0 at the end should be deleted, REMOVE operation.

Temporary view is recorded in ASIC DB under table *TEMP_ASIC_STATE* and cleared after view transition

Orchagent sends INIT_VIEW to syncd at start, syncd creates a temporary view and records new state, once orchagent sends APPLY_VIEW the difference between the current and the temporary view is pushed to the ASIC.

### SYNCD in Fast Fast Boot

In fast-fast boot mode, syncd does not create a temporary view, instead all SAI calls from orchagent are executed immediately.
At the end of configuration restoration, orchagent sends APPLY_VIEW. Syncd uses APPLY_VIEW event as an indication of configuration end and sets switch attribute SAI_SWITCH_ATTR_FAST_API_ENABLE=false and LibSAI switches the ASIC banks.

## SWSS common APIs

1. [WarmStart](https://github.com/sonic-net/sonic-swss-common/blob/master/common/warm_restart.h) 
provides methods to get/set application warm state, timers, check if the application currently starts in warm-restart or system warm-reboot mode, etc. Available for Python via swig bindings.
2. [AppRestartAssist](https://github.com/sonic-net/sonic-swss/blob/master/warmrestart/warmRestartAssist.h) wraps WarmStart class and provides higher level APIs for getting/setting application warm start state as well as provides an interface to help perform reconciliation.


## Orchagent

Orchagent translates the configuration in APP DB and partially CFG DB into SAIRedis API calls which are then sent to syncd daemon and translated to actual SAI calls.

Orchagent shutdown flow disables FDB aging and learning to not let HW change FDB table while software is down as that could lead to inconsistency between FDBs in redis vs. ASIC after reboot and freezes:

```mermaid
sequenceDiagram
    participant warm-reboot
    loop Retry RESTARTCHECK N[5] times until it succeeds
        warm-reboot -->> Redis DB: RESTARTCHECK request
        activate Redis DB
        Redis DB -->> orchagent: RESTARTCHECK request
        activate orchagent
        Redis DB -->> warm-reboot: 
        deactivate Redis DB
        alt Are pending tasks in m_toSync present?
            orchagent -->> Redis DB: RESTARTCHECK failure
            activate Redis DB
            Redis DB -->> warm-reboot: RESTARTCHECK failure
            deactivate Redis DB
            activate warm-reboot
            opt Number or retries reached N times?
                warm-reboot -->> User: Fail the operation
            end
            deactivate warm-reboot
        else
            orchagent -->> syncd: Disable FDB aging
            activate syncd
            syncd -->> orchagent: 
            deactivate syncd
            loop For each port
                orchagent -->> syncd: Disable bridge port FDB learning
                activate syncd
                syncd -->> orchagent: 
                deactivate syncd
            end
            orchagent -->> Redis DB: RESTARTCHECK success
            activate Redis DB
            Redis DB -->> warm-reboot: RESTARTCHECK success
            deactivate Redis DB
            activate warm-reboot
            warm-reboot -->> User: Operation is successful,<br/> orchagent is ready for warm restart
            deactivate warm-reboot
            orchagent -->> orchagent: Freeze, no more tasks are <br/> executed from APP DB, CFG DB
            deactivate orchagent
        end
    end

```
<!-- omit in toc -->
###### Figure 5. Orchagent shutdown 

The main operation of orchagent warm-boot startup consists of:
1. Call Orch::bake() on all Orch to refill internal task queue - *m_toSync* with data from APPL_DB/CFG_DB
2. Call Orch::doTask() on all Orch to execute tasks from *m_toSync*

The tasks form a tree due to dependencies (e.g. LAG member creation only after the LAG is created itself)
and orchagent executes Orch::doTask() 3 times to ensure there are no pending tasks that were postponed due to missing dependency.

```mermaid
sequenceDiagram
    participant orchagent
    participant STATE_DB
    participant Orch
    participant APPL_DB
    participant syncd

    orchagent -->> STATE_DB: WarmStart::setWarmStartState("orchagent", WarmStart::INITIALIZED);
    activate STATE_DB
    activate orchagent
    STATE_DB-->>orchagent: 
    deactivate STATE_DB
    deactivate orchagent
    orchagent -->> syncd: SAI_REDIS_SWITCH_ATTR_NOTIFY_SYNCD=SAI_REDIS_NOTIFY_SYNCD_INIT_VIEW
    activate orchagent
    activate syncd
    syncd -->> orchagent: SAI_STATUS_SUCCESS
    deactivate orchagent
    deactivate syncd
    orchagent -->> APPL_DB: 

    activate orchagent

    loop For each Orch
        orchagent -->> Orch: Orch::bake()
        activate Orch
        Orch -->> APPL_DB: 
        activate APPL_DB
        APPL_DB -->> Orch: Read configuration and save into m_toSync
        deactivate APPL_DB
        Orch -->> orchagent: 
        deactivate Orch
    end 

    loop For each Orch
        orchagent -->> Orch: Orch::doTask()
        activate Orch
        Orch -->> syncd: 
        activate syncd
        syncd -->> Orch: Execute SAI APIs
        deactivate syncd
        Orch -->> orchagent: 
        deactivate Orch
    end 

    deactivate orchagent

    opt Are more pending tasks in m_toSync?
        activate orchagent
        orchagent->>orchagent: Warm restart failure
        deactivate orchagent
        note right of orchagent: If any of the tasks wasn't processed,<br> it means there's inconsistent state between <br>orchagent and SAI and we fail immediately.
    end

    activate orchagent
    orchagent -->> STATE_DB: WarmStart::setWarmStartState("orchagent", WarmStart::RESTORED);
    activate STATE_DB
    STATE_DB-->>orchagent: 
    deactivate STATE_DB
    deactivate orchagent
    orchagent -->> syncd: SAI_REDIS_SWITCH_ATTR_NOTIFY_SYNCD=SAI_REDIS_NOTIFY_SYNCD_APPLY_VIEW
    activate orchagent
    activate syncd
    syncd -->> orchagent: SAI_STATUS_SUCCESS
    deactivate orchagent
    deactivate syncd
    orchagent -->> STATE_DB: WarmStart::setWarmStartState("orchagent", WarmStart::RECONCILED);
    activate STATE_DB
    activate orchagent
    STATE_DB-->>orchagent: 
    deactivate STATE_DB
    deactivate orchagent
    orchagent -->> syncd: sync ports operational status from HW
    activate orchagent
    activate syncd
    syncd -->> orchagent: 
    deactivate orchagent
    deactivate syncd
     Note right of orchagent: Warm start flow is finished, continue to operate in normal mode
```
<!-- omit in toc -->
###### Figure 6. Orchagent startup 

## Neighbors

Neighbors configuration is a crucial part of the L3 switch software. It is required that the neighbor configuration on the hardware is in sync with the actual switch neighbors on the network.
It can't be assumed that neighbors won't change during warm restart window, while the software is restarting, the SONiC switch software has to be ready for scenarios in which during the restart window:
- Existing neighbors went down, e.g: VMs crashed on the server connected to ToR switch which undergoes warm-reboot.
- New neighbors appeared on the network, e.g: VMs created on the server connected to ToR switch which undergoes warm-reboot.
- MAC changes, e.g: VMs re-created or re-configured on the server connected to ToR switch which undergoes warm-reboot.

This is handled by the neighbors reconciliation flow. Three applications are participating in this flow:
- *orchagent*
- *neighsyncd*
- *restore_neighbors.py*

During the restoration process, all known pre-boot neighbors are programmed to the Linux Kernel as STALE neighbors and an ARP or NDP packet is sent depending on whether the neighbors is IPv4 or IPv6 neighbor.
This is done by the process restore_neighbors.py which is part of swss.
This process is repeated from few times until the neighsyncd_timer expires, which by default is set to 110 sec. It is required to continuously repeat the process since some *host* interfaces might not be created
yet or not yet operationally up. In case the neighbor is alive it will reply with a valid ARP/NDP with up-to-date MAC address and the Linux Kernel itself will mark the neighbor as REACHABLE.
Once timer expires, restore_neighbors.py set a flag to STATE DB table called NEIGH_RESTORE_TABLE field "restored" with value "true".

In the meantime, the neighsyncd process is waiting for the "restored" flag to be set to "true" by the restore_neighbors.py script and starts the reconciliation process, which calculates the difference between
Linux Kernel neighbors (*new-life* state) and APP DB neighbors (*old-life* state) and pushes the difference to APP DB:
- New neighbors are set to APP DB
- Existing neighbors whose MAC has changed are set to APP DB with the new MAC
- Stale entries are removed from the APP DB

Once that is done, orchagent will apply the changes down to SAIRedis, syncd and SAI.

```mermaid
sequenceDiagram
    neighsyncd -->> neighsyncd: Wait till NeighSync::isNeighRestoreDone() is true

    activate restore_neighbors
    restore_neighbors -->> STATE_DB: WarmStart.initialize("neighsyncd", "swss")
    activate STATE_DB
    STATE_DB -->> restore_neighbors: 
    deactivate STATE_DB

    opt WarmStart.isSystemWarmRebootEnabled()
        Note right of restore_neighbors: Neighbor restoration is not required in case<br/>single service restart, <br/> since kernel already has neighbors
        restore_neighbors -->> restore_neighbors: Start neighsyncd_timer
        deactivate restore_neighbors

        loop neighsyncd_timer not expired
            loop For each neighbor in APPL_DB
                activate restore_neighbors
                restore_neighbors -->> Kernel: Netlink STALE neighbor add IP/MAC
                
                activate Kernel
                Kernel -->> restore_neighbors: 
                deactivate Kernel
                
                
                restore_neighbors -->> Kernel: Send ARP/NDP to resolve neighbor
                activate Kernel
                Kernel -->> restore_neighbors: 
                deactivate Kernel
                deactivate restore_neighbors

                Note right of restore_neighbors: When neighbor replies with ARP/NDP<br/> the kernel state for the neighbor<br/> becomes REACHABLE
            end
        end
    end

    activate restore_neighbors
    restore_neighbors -->> STATE_DB: db.set(db.STATE_DB, 'NEIGH_RESTORE_TABLE|Flags', 'restored', 'true')
    activate STATE_DB
    STATE_DB -->> restore_neighbors: 
    STATE_DB -->> neighsyncd: 
    deactivate STATE_DB
    deactivate restore_neighbors

    
    neighsyncd -->> Kernel: Dump all neighbors   
    activate Kernel 
    
    Kernel -->> neighsyncd: NeighSync::onMsg()
    deactivate Kernel

    activate neighsyncd
    neighsyncd -->> neighsyncd: Save entries into <br/>internal cache map
    neighsyncd -->> APPL_DB: Push the difference into database<br/> set new neighbors, update existing, remove stale.

    deactivate neighsyncd
```
<!-- omit in toc -->
###### Figure 7. Neighbors reconciliation

## Teamd

Teamd user space process implements the LACP protocol. LACP is fully supported by SONiC including the warm-reboot functionality.
The upstream teamd implementation, unfortunately, does not implement warm restart capabilities, so SONiC has [patched](https://github.com/sonic-net/sonic-buildimage/blob/master/src/libteam/patch/0008-libteam-Add-warm_reboot-mode.patch) teamd which implement
the required teamd changes to support SONiC warm-reboot.

Teamd warm-reboot idea is to dump current LACP port channel and port channel member state as well as LACP PDUs last received from the peer.

Teamd has been extended to support starting in warm mode with the addition of '-w' and '-L' command line options:

| Short option | Long option      | Description                          |
| ------------ | ---------------- | ------------------------------------ |
| -w           | --warm-start     | Start in warm mode                   |
| -L           | --lacp-directory | Directory for saved LACP PDU packets |

To implement the shutdown process teamd handles SIGUSR1 signal as the indication of the start of warm shutdown.

Teamd warm restart only works in case LACP slow mode is configured. In slow LACP mode, we have 90 sec (3 * 30s LACP slow period)
to start again otherwise the peer will reset the LAG causing disruption of traffic. So, *fast* LACP mode is not supported for warm restart.

<!-- omit in toc -->
#### Teamd warm restart flow

```mermaid
sequenceDiagram
    participant warm-reboot
    warm-reboot -->> teamd: SIGUSR1
    activate teamd

    teamd -->> teamd: teamd_refresh_ports()
    Note right of teamd: Send last LACP PDU<br/>From this point we have 90 sec to restart before peer will reset the LAG state.

    teamd -->> teamd: teamd_ports_flush_data()
    loop for each teamd member port
        teamd -->> File System: /host/warmboot/teamd/EthernetYYY
        Note right of teamd: Save lacp_port->last_pdu
    end

    teamd -->> teamd: lacp_state_save()
    teamd -->> File System: /host/warmboot/teamd/PortChannelXXX
    Note right of teamd: Save lacp->carrier_up, lacp->wr.nr_of_tdports and<br/> lacp->wr.state[i].name, lacp->wr.state[i].enabled<br> for each teamd member port

    deactivate teamd

```
<!-- omit in toc -->
###### Figure 7. Teamd shutdown 

```mermaid
sequenceDiagram
    participant teammgrd
    teammgrd -->> teamd: Started in warm mode
    activate teamd

    File System -->> teamd: /host/warmboot/PortChannelXXX
    teamd -->> teamd: lacp_state_load()

    loop wait for member host interfaces EthernetYYY to become up
        opt Warm started?
            File System -->> teamd: /host/warmboot/EthernetYYY
            teamd -->> teamd: lacpdu_read() <br>reads LACP packet from the file
            teamd -->> teamd: lacpdu_process() <br>injects the packet into teamd to<br> restore members state
        end
    end

    teamd -->> teamd: stop_wr_mode()
    Note right of teamd: Warm restart fort teamd finished<br/> Regular teamd operational mode
    
    
    deactivate teamd
```
<!-- omit in toc -->
###### Figure 8. Teamd startup 


<!-- omit in toc -->
#### Teamd reconciliation flow

During the warm restart window the peer of the LAG can go down or one of peers LAG members may go down. The network state must be reconciled with the state of the ASIC.
In this process the *teamsyncd* and *teammgrd* processes are involved is involved.

```mermaid
sequenceDiagram
    CFG_DB -->> teammgrd: 
    activate teammgrd
    
    teamsyncd -->> STATE_DB: WarmStart::setWarmStartState(TEAMSYNCD_APP_NAME, WarmStart::INITIALIZED);
    activate teamsyncd
    activate STATE_DB
    STATE_DB -->> teamsyncd: 
    deactivate STATE_DB
    teamsyncd -->> teamsyncd: Start teamsyncd_timer (default 70 sec)
    deactivate teamsyncd

    teammgrd -->> Kernel: Start teamd in warm mode
    teammgrd -->> Kernel: teammctl add ports to teamd
    deactivate teammgrd
    Kernel -->> teamsyncd: TeamSync::onMsg()
    Note left of Kernel: Kernel sends update<br/> of LAG & members state
    activate teamsyncd
    teamsyncd -->> teamsyncd: Save to internal cache

    opt teamsyncd_timer expired
        Note right of teamsyncd: Start reconciliation
        activate APPL_DB
        teamsyncd -->> APPL_DB: Push the difference to APP DB
        APPL_DB -->> teamsyncd: 
        deactivate APPL_DB
    end

    teamsyncd -->> STATE_DB: WarmStart::setWarmStartState(TEAMSYNCD_APP_NAME, WarmStart::RECONCILED);

    deactivate teamsyncd
```
<!-- omit in toc -->
###### Figure 9. Teamsyncd reconciliation 

## BGP
is dIn
FRR suite supports Graceful Restart and EOR capabilities:
- https://datatracker.ietf.org/doc/html/rfc4724

When BGP on a router restarts, all the BGP peers detect that
the session went down and then came up.  This "down/up" transition
results in a "routing flap" and causes BGP route re-computation,
generation of BGP routing updates, and unnecessary churn to the
forwarding tables.

BGP capability, termed "Graceful Restart Capability", is defined that would allow a BGP speaker to express its
ability to preserve forwarding state during BGP restart.

An UPDATE message with no reachable Network Layer Reachability
Information (NLRI) and empty withdrawn NLRI is specified as the End-
of-RIB marker that can be used by a BGP speaker to indicate to its
peer the completion of the initial routing update after the session
is established. 

The following FRR configuration is required to support warm-reboot:

```
 bgp graceful-restart
 bgp graceful-restart restart-time 240
 bgp graceful-restart select-defer-time 45
 bgp graceful-restart preserve-fw-state
```

These settings are only configured in FRR when DEVICE_METADATA field "type" is set to "ToRRouter".

<!-- omit in toc -->
##### FPM sync daemon reconciliation flow

```mermaid
sequenceDiagram
    participant bgp_eoiu_marker.py
    participant STATE_DB
    participant bgpd
    participant zebra 
    participant fpmsyncd
    participant APPL_DB
    
    activate bgp_eoiu_marker.py
    bgp_eoiu_marker.py -->> STATE_DB: warmstart.initialize("bgp", "bgp")
    loop 120s timer

          
        bgp_eoiu_marker.py -->> bgpd: vtysh -c 'show bgp summary json'
        activate bgpd  
        deactivate bgpd
        bgpd -->> bgp_eoiu_marker.py: 
        opt EOIU marker set for IPv4 neighbors?
            bgp_eoiu_marker.py -->> STATE_DB: "set BGP_STATE_TABLE|IPv4|eoiu state true"
        end
        opt EOIU marker set for IPv6 neighbors?
            bgp_eoiu_marker.py -->> STATE_DB: "set BGP_STATE_TABLE|IPv6|eoiu state true"
        end
    end

    activate fpmsyncd
    fpmsyncd -->> fpmsyncd: Start bgp_timer (default 120s)
    fpmsyncd -->> STATE_DB: WarmStartHelper::setState(WarmStart::RESTORED);
    activate STATE_DB
    STATE_DB -->> fpmsyncd: 
    deactivate STATE_DB

    bgpd -->> zebra: RTM_NEWROUTE/<br>RTM_DELROUTE
    zebra -->> fpmsyncd: RTM_NEWROUTE/<br>RTM_DELROUTE 
    fpmsyncd -->> fpmsyncd: WarmStartHelper::insertRefreshMap
    note left of fpmsyncd:  Updates from zebra are cached for later reconciliation
    
    deactivate bgp_eoiu_marker.py

    fpmsyncd -->> fpmsyncd: wait until IPv4|eoiu, IPv6|eoiu<br/> or timer expires
    deactivate fpmsyncd

    activate fpmsyncd
    fpmsyncd -->> APPL_DB: Run reconciliation
    activate APPL_DB
    APPL_DB -->> fpmsyncd: 
    deactivate APPL_DB
    Note right of fpmsyncd: Differences are pushed to APP DB

    deactivate fpmsyncd

    fpmsyncd -->> STATE_DB: WarmStartHelper::setState(WarmStart::RECONCILED);
    activate STATE_DB
    STATE_DB -->> fpmsyncd: 
    deactivate STATE_DB
```
<!-- omit in toc -->
###### Figure 10. Fpmsyncd reconciliation 

## Management plane

Management plane services, including:
- telemetry
- snmp
- management framework

And more, are delayed by the systemd timer by 3m and 30 seconds since Linux kernel boot. This time is enough for critical services to start first.
The main idea behind delaying these services is mostly to free CPU time for more important services at boot.

## Control plane assistant (CPA)

This is an additional configuration done before warm reboot User optionally passes an IP of the control plane assistant to warm-reboot command
It consists of mirroring configuration for ARP/NDP packets and VXLAN configuration for ARP/NDP replies this configuration is removed after warm reboot finishes by the warm-reboot finalizer.

![Control Plane Assistant](img/control-plane-assistant.png)
<!-- omit in toc -->
###### Figure 11. Control Plane Assistant 

Example configuration that is applied by neighbor_advertiser script:

```json
    "VXLAN_TUNNEL": {
        "neigh_adv": {
            "dst_ip": "192.168.8.1",
            "src_ip": "10.1.0.32"
        }
    },
    "VXLAN_TUNNEL_MAP": {
        "neigh_adv|map_1": {
            "vlan": "Vlan1000",
            "vni": "1000"
        }
    },
    "ACL_RULE": {
        "EVERFLOWV6|rule_nd": {
            "ICMPV6_TYPE": "135",
            "PRIORITY": "8887",
            "mirror_action": "neighbor_advertiser"
        },
        "EVERFLOW|rule_arp": {
            "PRIORITY": "8888",
            "ether_type": "2054",
            "mirror_action": "neighbor_advertiser"
        }
    },
```


## Warm Boot finalizer

Every warm boot aware application is writing it’s state to STATE DB WARM_RESTART_TABLE *warmboot-finalizer.service* is a systemd service that starts *finalize-warmboot.sh* script on boot and waits until all services reach “reconciled” state. Once they do, this service performs a cleanup: disables warm-restart in CONFIG_DB, removes control plane configuration if it is present and saves configuration to */etc/sonic/config_db.json*.

Once this service finished, the warm-boot process is completed. Many applications, automation/test scripts use *systemctl is-active warmboot-finalizer.service* to determine whether warm-reboot is still in progress.

```mermaid
sequenceDiagram
    participant fw as finalize-warmboot.sh
    participant STATE_DB
    participant CFG_DB
    participant s as /etc/sonic/config_db.json

    activate fw

    loop for each component
        fw -->> STATE_DB: read component state
        activate STATE_DB
        STATE_DB -->> fw: 

        opt All components reached reconciled state?
            fw -->> fw: end loop
        end

        deactivate STATE_DB
    end

    fw -->> CFG_DB: Remove CPA configuration
    activate CFG_DB
    CFG_DB -->> fw: 
    deactivate CFG_DB

    deactivate fw

    fw -->> CFG_DB: Save configuration
    activate CFG_DB
    CFG_DB ->> s: 
    CFG_DB -->> fw: 
    deactivate CFG_DB
```
<!-- omit in toc -->
###### Figure 12. Warmboot finalizer 

## System flow

TODO:

<!-- omit in toc -->
###### going down path

- stop bgp docker 
  - enable bgp graceful restart
  - same as fast-reboot
- stop teamd docker
  - stop teamd gracefully to allow teamd to send last valid update to be sure we'll have 90 seconds reboot time available.
- stop swss docker
  - disable mac learning and aging
  - freeze orchagent
  - set redis flag WARM_RESTART_TABLE:system
  - kill swss dockers
- save the APPL_DB and asic db into the files.
  - save the whole Redis database into ```/host/warmboot/dump.rdb```
- stop syncd docker
  - warm shutdown
  - save the SAI states in ```/host/warmboot/sai-warmboot.bin```
  - kill syncd docker
- stop database
- use kexec to reboot, plus one extra kernel argument

<!-- omit in toc -->
# going up path

- Use kernel argument ```SONIC_BOOT_TYPE=warm``` to determine in warm starting mode
- start database
  - recover redis from ```/host/warmboot/*.json```
  - implemented in database system service
- start syncd docker
  - implemented inside syncd docker
  - recover SAI state from ```/host/warmboot/sai-warmboot.bin``` 
  - the host interface will be also recovered.
- start swss docker
  - orchagent will wait till syncd has been started to do init view.
  - will read from APP DB and do comparison logic.
- start teamd docker
  - at the same time as swss docker. swss will not read teamd app db until it finishes the comparison logic.
- start bgp docker
  - at the same time as swss docker. swss will not read bgp route table until it finishes the comparison logic.

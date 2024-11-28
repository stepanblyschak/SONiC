# Buffers Configuration using Bulk API #

## Table of Content

### Revision

### Scope

The scope of the document is utilizing SAI bulk API to speed up buffers configuration in fast and warm reboot.

### Definitions/Abbreviations

| Definitions/Abbreviation | Description                             |
| ------------------------ | --------------------------------------- |
| ASIC                     | Application specific integrated circuit |
| HW                       | Hardware                                |
| SW                       | Software                                |
| SWSS                     | Switch state service                    |
| SYNCD                    | ASIC synchronization service            |
| SAI                      | Switch Abstraction Interface            |
| UT                       | Unit test                               |

### Overview

SONiC starts supporting single ASIC switches with more and more data plane ports available. More HWSKUs with more ports become available, e.g. 256 ports.
This raises a problem of SONiC scaling with the number of ports. Each port has some number of PGs and some number of queues per port depending on ASIC that needs to be configure.
E.g. for 256 port system there could be 2048 PGs and 4096 queues to be configured. Given that there is a strict time requirements for fast-reboot and warm-reboot it is neccesary to
configure all ports and all related port objects in time and keep that part optimized.

Batching is a commonly used approach to speed up processing. It allows to minimize an overhead incured by ASIC Redis channel and allows for more efficient configuration processing at SAI level.
SAI already supports it using bulk APIs. This design adds support for buffers configuration using bulk API as an optimization for fast and warm-reboot.

### Requirements

- Utilize bulk API for buffers configuration to improve configuration performance in fast and warm-reboot
- Support both static and dynamic buffer model
- Fallback to legacy per object API if SAI vendor does not implement or support bulk API

### Restrictions/Limitations

Buffer configuration in bulk is going to be restricted to fast and warm-boot only. In cold boot in dynamic mode the buffer configuration is generated during boot time and the APPL_DB
is filled gradually. In this case, there's no practical way to bulk all configuration together, therefore in cold boot orchagent behaves exactly as today.

In fast-reboot and warm-reboot, APPL_DB is already filled with buffer configuration saved from previous boot. This configuration is read all at once in ```bake()``` giving us the oportunity to batch configuration togather to syncd.

### SAI API

APIs that are available as of 1.15:

| API | Attribute(s) |
|-----|--------------|
| set_ports_attribute | SAI_PORT_ATTR_QOS_INGRESS_BUFFER_PROFILE_LIST, SAI_PORT_ATTR_QOS_EGRESS_BUFFER_PROFILE_LIST |
| set_ingress_priority_groups_attribute | SAI_INGRESS_PRIORITY_GROUP_ATTR_BUFFER_PROFILE |
| set_queues_attribute | SAI_QUEUE_ATTR_BUFFER_PROFILE_ID |

### Architecture Design

N/A

### High-Level Design

#### Syncd

Syncd's ```Syncd::processBulkOidSet``` needs to be extended to support ```SAI_COMMON_API_BULK_SET``` in the same way create is implemented with a fallback to non-bulk API:

##### Figure 3. Syncd Bulk Set Flow

```mermaid
%%{
  init: {
    "theme": "forest",
    "sequence": {
      "rightAngles": true,
      "showSequenceNumbers": true
    }
  }
}%%
sequenceDiagram
    participant ASIC_DB
    participant Syncd
    participant SAI

    ASIC_DB-->>Syncd: set_<object-type>s_attribute()

    activate Syncd

    Syncd-->>SAI: set_<object-type>s_attribute()
    activate SAI
    SAI-->>Syncd: return status
    deactivate SAI

    alt status is NOT_IMPLEMENTED or NOT_SUPPORTED
        loop oid
            Syncd-->>SAI: set_<object-type>s_attribute()
            activate SAI
            SAI-->>Syncd: return status
            deactivate SAI
        end
    else else
        Syncd-->>ASIC_DB: return status
    end

    deactivate Syncd
```

#### BufferOrch


```mermaid
%%{
  init: {
    "theme": "forest",
    "sequence": {
      "rightAngles": true,
      "showSequenceNumbers": true
    }
  }
}%%
sequenceDiagram
    participant APPL_DB
    participant OrchDaemon
    participant BufferOrch
    participant Syncd

    activate OrchDaemon

    rect rgba(0, 0, 0, 0.0)

        OrchDaemon-->>BufferOrch: bake()

        activate BufferOrch

        BufferOrch-->>APPL_DB: read BUFFER_PG, BUFFER_QUEUE, PORT
        activate APPL_DB


        APPL_DB-->>BufferOrch: return

        BufferOrch-->>OrchDaemon: return

        deactivate APPL_DB
        deactivate BufferOrch
    end


    rect rgba(0, 0, 0, 0.0)
        OrchDaemon-->>BufferOrch: doTask()

        activate BufferOrch
        BufferOrch-->>BufferOrch: processPriorityGroup()

        BufferOrch-->>BufferOrch: processQueue()
        BufferOrch-->>BufferOrch: processIngressBufferProfileList()
        BufferOrch-->>BufferOrch: processEgressBufferProfileList()

        BufferOrch-->>BufferOrch: flush()


        BufferOrch-->>Syncd: set_ingress_priority_groups_attribute()
        activate Syncd

        Syncd->>BufferOrch: return
        deactivate Syncd

        BufferOrch->>Syncd: set_queues_attribute()
        activate Syncd
        Syncd->>BufferOrch: return
        deactivate Syncd

        BufferOrch->>Syncd: set_ports_attribute()
        activate Syncd
        Syncd->>BufferOrch: return

        deactivate Syncd

        BufferOrch-->>OrchDaemon: return

        deactivate BufferOrch
    end

    deactivate OrchDaemon
```

### Configuration and management
N/A

#### Manifest (if the feature is an Application Extension)
N/A

#### CLI/YANG model Enhancements
N/A

#### Config DB Enhancements
N/A

### Warmboot and Fastboot Design Impact

This design optimizes buffers configuration during warm-boot and fast-boot.

TODO: give time improvement data on VS.

### Memory Consumption
Batching implies increased memory usage in orchagent. The increased memory usage is relevant only for warm and fast boot flows where bulk API is utilized. Batch buffer for PG and QUEUE is using at most ```N * (number_of_pgs_per_port + number_of_queues_per_port) * sizeof(sai_attribute_t)``` additional bytes. (TODO: for port buffer lists)

### Testing Requirements/Design

- Regression testing
- Warmboot/fastboot downtime testing

#### Unit Test cases

- Existing buffer configuration UT
- UT

#### System Test cases

- warm-reboot
- fast-reboot

### Open/Action items - if any

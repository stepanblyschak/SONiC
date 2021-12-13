# Feature State Recording #

## Table of Content

### Revision

### Scope

This documents covers an improvement of *feature* control (*hostcfgd*) in SONiC.

### Definitions/Abbreviations

### Overview

SONiC allows enabling and disabling features through CONFIG DB. This is implemented as a configuration table in CONFIG DB the hostcfgd daemon is subscribed to
and handles service state configuration. However, there is no way to block until the service is started, which is useful in some cases, like upgrade via
sonic-package-manager. Today, sonic-package-manager duplicates the code from hostcfgd related to start, stop of the service and directly executes systemctl
commands. The goal of this document is to delegate all the feature handling job to hostcfgd.

### Requirements

A mechanism to block till the feature service is started or stopped is required.

### Architecture Design

No change in SONiC architecture.

### High-Level Design

*hostcfgd* will write the state into STATE DB:

```
127.0.0.1:6379[6]> hgetall FEATURE|lldp
1) "state"
2) "enabled"
```

There will be a new field "state" in STATE DB under FEATURE table for every feature.
The value of the "state" field can be one of "enabled"/"disabled"/"failed" states.

Once *hostcfgd* has finished executing start or stop operation a coresponding *state* is recorded into STATE DB - "enabled" or "disabled".
In case *hostcfgd* has failed to execute an operation a generic "failed" state is set. The "failed" state can be actually extended with more
information in a field "error_message" which will contain the output of the failed command ("systemctl start/stop") for better debugging, but
this is out of scope for this document.

This new field is not a replacement of those fields in STATE DB introduced by Kubernetes support - https://github.com/Azure/SONiC/pull/680.
The "system_state" and "remote_state" are related to the container state, not the service state.

The *sonic-package-manager* utility when performing an upgrade of the package needs to perform (in high level):
- install new docker image
- if the service was enabled in CONFIG DB:
  - // stop the service (**must** block until it is finished):
  - Set state "disabled" in CONFIG DB
  - Wait for "state" == "disabled" in STATE DB or timeout
- uninstall old docker image
- if the service was enabled in CONFIG DB:
  - // start the service (blocks until it is finished)
  - Set state "enabled" in CONFIG DB
  - Wait for "state" == "enabled" in STATE DB or timeout

### SAI API

N/A

### Configuration and management

N/A

#### CLI/YANG model Enhancements

N/A

#### Config DB Enhancements

N/A

### Warmboot and Fastboot Design Impact

N/A

### Restrictions/Limitations

### Testing Requirements/Design

Covered by sonic-package-manager upgrade and uninstall tests.

#### Unit Test cases

UT added for hostcfgd that the correct state is reflected in STATE DB.

#### System Test cases

N/A

### Open/Action items - if any

N/A

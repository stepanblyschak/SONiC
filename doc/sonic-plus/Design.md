# SONiC extentions HLD

## High Level Design Document

# Table of Contents
- [SONiC extentions HLD](#sonic-extentions-hld)
  - [High Level Design Document](#high-level-design-document)
- [Table of Contents](#table-of-contents)
- [List of Tables](#list-of-tables)
- [List of Figures](#list-of-figures)
- [About this Manual](#about-this-manual)
- [Scope](#scope)
- [Motivation](#motivation)
- [Definitions/Abbreviation](#definitionsabbreviation)
- [Overview](#overview)
- [Requirements](#requirements)
- [Design](#design)
  - [Assumptions](#assumptions)
- [SONiC extention terminology and user flows](#sonic-extention-terminology-and-user-flows)
- [SONiC extention command line interface](#sonic-extention-command-line-interface)
- [...](#)
- [...](#-1)
- [Building packages here, like feature_1.0.0_amd64.deb](#building-packages-here-like-feature100amd64deb)
- [Copy artifacts from build environment](#copy-artifacts-from-build-environment)
- [Installing packages](#installing-packages)
- [Open questions](#open-questions)

# List of Tables

# List of Figures

# About this Manual

This document provides an overview of the design of implementation of SONiC extentions.

# Scope

This document describes the high level design of SONiC extentions infrastructure.

# Motivation



# Definitions/Abbreviation

# Overview

SONiC extentions - a SONiC infrastructure for building and integrating third-party out-of-tree services/features that provide their functionality as a docker container managed as a systemd service.

# Requirements

- Extention provides its functionality as another docker container running alongside SONiC core services
- Extension is distributed as a docker image available though Docker Hub or private registry
- SONiC extention services is a subset to existing *feature* conecpt already available in SONiC, that means, a SONiC+ service is a subject to the same handlers as for *feature*s
- SONiC extention infrastructure provides a CLI interface to install, remove, list available SONiC extensions
- Installed SONiC extenstions migrate during SONiC-2-SONiC upgrades
- An extention is backward and forward compatible between SONiC releases

# Design

## Assumptions
 - For simplicity, it is assumed that a SONiC extention name is the same as the feature name, the same as systemd service name and the same as docker container name.
 - SWSS API and rest of SONiC common API does not introduce breaking changes: e.g. old libswsscommon should work with newer SONiC, otherwise feature compatibility is broken. As there is no way to know SONiC common API version, feature will fail at runtime after start.

# SONiC extention terminology and user flows

There will be two notions a repository and a feature.
A repository is basically a Docker Hub or registry repository with all the images for a feature.
User flow is defined like this:
1. Add a new repository
2. Install a feature from repository

Once a repository is added, feature installation will pull docker image from repository using a SONiC release number as a tag by default (see Versioning section) and not the latest as docker does by default; in case if release number tag was not found will be pulled latest by default.
This will guaranty that the version user pulled from the repository is compatible with the SONiC release user is using.
Once installed, a feature can be managed using the *feature* commands in SONiC.

User flow to uninstall, remove repository:
1. Uninstall the feature
2. Remove the repository from database

# SONiC extention command line interface

To manage SONiC extentions there will be a new CLI utility called sonic-extention. Here is the CLI interface of sonic-extention utility:

```
admin@sonic:~$ sonic-extention
Usage: sonic-extention [OPTIONS] COMMAND [ARGS]...

  CLI to manage SONiC extentions

Options:
  --help  Show this message and exit

Commands:
  repository  List, add or remove SONiC extentions repository list
  install     Install SONiC extention from repository
  uninstall   Uninstall SONiC extention
```

sonic-extention maintains its repository database in */var/lib/sonic-extention/repositories.json*.
Schema definition for this file is following:


Path | Type | Description
--- | --- | ---
/name | string | Name of the application 
/name/repository | string | Repository in docker registry 
/name/description | string | Description of the application 
/name/status | string | One of "Installed"/"Not installed"/"Installation error". Represents the application instalation status

"Installation error" status is used when trying to installed incompatible container (not in SONiC extention format), general error during installation or failure during SONiC-2-SONiC migration procedure that will be described soon.

An example of a content in JSON format:

```json
{
  "featureA": {
    "repository": "user/featureA",
    "description": "feature A description",
    "status": "Installed"
  },
  "featureB": {
    "repository": "user/featureB",
    "description": "feature B description",
    "status": "Not installed"
  },
}
```

Given the above file as a database of features the list command of sonic-extention tool will produce the following output:

```
admin@sonic:~$ sonic-extention repository list
Name       Repository      Description             Status
---------  --------------  ----------------------  --------------
featureA   user/featureA   feature A description   Installed
featureB   user/featureB   feature B description   Not installed
```

Add or remove repository commands:

```
admin@sonic:~$ sudo sonic-extention repository add [NAME] [REPOSITORY] --description=[STRING]
admin@sonic:~$ sudo sonic-extention repository remove [NAME]
```

sonic-extention CLI should prevent user from deleting a repository when extention is installed.

Installation commands example:
```
admin@sonic:~$ # install guarantied compatible featureA
admin@sonic:~$ sudo sonic-extention install featureA
admin@sonic:~$ # install latest version of featureA
admin@sonic:~$ sudo sonic-extention install featureA --tag=latest
```

An option "--tag" in CLI will allow user to use latest feature version.

It will be possible to install the extention from a docker image file:

```
admin@sonic:~$ ls featureA.gz
featureA.gz
admin@sonic:~$ sudo sonic-extention install featureA.gz
```

This option should mainly be used for debugging, developing purpose, while the prefered way will be to pull the image from repository.
Repository list is not updated in this case, because it was a local installation.

Later, SONiC can come with a default list of trusted repositories.

After installation, user can enable the feature:

```
admin@sonic:~$ sudo config feature featureA enabled 
```

```show version``` command can be used to display feature docker image version.

# SONiC application extention interfaces

Why to autogenerate CLI/REST

## SONiC command line interface extention

SONiC command line utilities, like "show", "config", "sonic-clear" have to be modified to support plugins.

Following common python practice to develop plugins there will be an empty package ```show.plugins``` and ```config.plugins```.
Generated commands will be put in a module as part of plugins package. show and config commands should be updated to autodiscover those CLI plugins.

An autogenerated module should provide a module function ```register_cli``` to be invoked with CLI object as parameter.

*show/main.py* (Example is following: [python packaging guidelines](https://packaging.python.org/guides/creating-and-discovering-plugins/#using-namespace-packages)):

```python
# ...
# ...

import show.plugins

def iter_plugins_namespace(ns_pkg):
    return pkgutil.iter_modules(ns_pkg.__path__, ns_pkg.__name__ + ".")

discovered_plugins = {
    name: importlib.import_module(name)
    for finder, name, ispkg
    in iter_namespace(myapp.plugins)
}

for plugin in discovered_plugins.values():
    plugin.register_cli(cli)
```


# SONiC extention format

A SONiC extention should have files neccecary for generation of all needed files that should be installed in host OS.
Specifically, there are the following files required:
- Feature systemd service file
- Feature docker container control script
- Initial configuration for the feature (similar to whole system initial configuration */etc/sonic/init_cfg.json*)
- CLI sub-commands for *show*, *config*

It is possible to autogenerate the above required files based on a service definition and database schema.
It is proposed to have the following files required for SONiC extention docker image and put them under */usr/share/sonic/sonic-extention/* directory inside docker image:
1. manifest.json
2. schema.json
3. additional_commands.json

## Manifest schema definition

The following table describes schema for version 1.0:


Path | Type | Description
--- | --- | ---
/version        | string          | Version of manifest file definition
/name           | string          | Application name
/requires       | list of strings | List of SONiC core services the application requires
/after          | list of strings | List of SONiC core services the application is set to start after
/docker_options | list of strings | List of options, appened to docker run when starting the container


A required "version" field can be used in case the format of manifest.json is changed in the future. In this case a migration script can be applied to convert format to the recent version. This is like SONiC has version inside CONFIG DB.
"requires" and "after" is related to service management, it maps to systemd service attributes and can be extented in the future if needed. "dockerOptions" are used to autogenerate service start/stop script from *docker_image_ctl.j2* file. It is required to have this file as part of SONiC image and put it in */usr/share/sonic/templates/*.

Example of manifest.json for featureA:

```json
{
  "version": "1.0",
  "name": "featureA",
  "requires": [
    "swss",
    "syncd",
    "database"
  ],
  "after": [
    "swss",
    "syncd",
    "database"
  ],
  "docker_options": [
    "-v /dev/shm:/dev/shm:rw",
  ]
}
```

## Database schema definition

schema.json is used to define the database tables that are relevant to the feature. The following table describes schema for version 1.0:

Path | Type | Description
--- | --- | ---
/version | string | Version of schema definition
/dbs | array | Array of databases
/dbs/{{index}}/name | string | Database name
/dbs/{{index}}/tables | array | Array of tables defined in the database
/dbs/{{index}}/tables/{{index}}/name | string | Database table name
/dbs/{{index}}/tables/{{index}}/keys | array | Array of keys definitions in the table
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/type | string | A choice of "predefined"/"object" depending on nature of keyss that reside in this table
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/name | string | A name of the key, for "predefined" also a defined key name. It will be used for CLI sub-command name or REST API path
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/operations | array | List of database operations available: "get", "set", "mod", "del". It will map to CLI "show" or "config" or to REST API - GET, POST, PUT/PATCH or DELETE
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/description | string | Description of the keys
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields | array | Array of field definitions
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/name | string | Database name of the field
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/description | string | Description for the field
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/pretty_name | string | Pretty name of the field (e.g. used as a column name in CLI)
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/reload_required | bool | Wether chaning of the field requires service restart (e.g. CLI will generate a promt to ask user to restart the service)
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/value/type | string | Type of value, one of "intrange"/"int"/"bool"/"choice"/"list"/"reference"/"ipaddress"
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/value/intmin | int | Minimum integer value, valid if type is "intrange"
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/value/intmax | int | Maximum integer value, valid if type is "intrange"
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/value/choice_list | array | List of choices, valid if type is "choice" or "list"
/dbs/{{index}}/tables/{{index}}/keys/{{index}}/fields/{{index}}/value/choice_table | string | Table name, user interface will generate a validation wether value is valid key inside given table, valid if type is "reference"

From the above data, it is possible to generate CLI commands. Example is given for an application that monitors CPU time for select processes and reports the statistics to the remote collectors:

```json
{
  "version": "1.0",
  "dbs": [
    {
      "name": "CONFIG_DB",
      "tables": [
        {
          "name": "CPU_REPORT",
          "keys": [
            {
              "type": "predefined",
              "name": "global",
              "operations": [
                "get",
                "set"
              ],
              "description": "Global configuration for cpu time report application",
              "fields": [
                {
                  "name": "enable",
                  "pretty_name": "Enabled",
                  "description": "Enable/disable the application",
                  "value": {
                    "type": "bool"
                  }
                },
                {
                  "name": "poll_interval",
                  "pretty_name": "Polling interval (s)",
                  "description": "Polling interval in seconds",
                  "value": {
                    "type": "int"
                  }
                }
              ]
            }
          ]
        },
        {
          "name": "CPU_REPORT_PROCESS",
          "keys": [
            {
              "type": "object",
              "name": "counter",
              "operations": [
                "get",
                "set",
                "mod",
                "del"
              ],
              "description": "Procecss CPU time counter",
              "fields": [
                 {
                  "name": "name",
                  "pretty_name": "Process name",
                  "description": "Process name to monitor",
                  "value": {
                    "type": "string"
                  }
                }
              ]
            }
          ]
        },
        {
          "name": "CPU_REPORT_COLLECTOR",
          "description": "Remote collector configuration",
          "keys": [
            {
              "type": "object",
              "name": "collector",
              "operations": [
                "get",
                "set",
                "mod",
                "del"
              ],
              "fields": [
                {
                  "name": "address",
                  "pretty_name": "Address",
                  "description": "Collector's IP address",
                  "value": {
                    "type": "ipaddress"
                  }
                },
                {
                  "name": "port",
                  "pretty_name": "Port",
                  "description": "Collector's port",
                  "value": {
                    "type": "intrange",
                    "intmin": "1024",
                    "intmax": "65000"
                  }
                }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

The above example generates:

For cpu-time-report global configuration:
```
admin@sonic:~$ sudo config cpu-time-report global enable
admin@sonic:~$ sudo config cpu-time-report global poll-interval 5
```

For tables containing objects, e.g. CPU_REPORT_PROCESS table:
```
admin@sonic:~$ sudo config cpu-time-report counter create orchagent-counter --name=orchagent
admin@sonic:~$ sudo config cpu-time-report counter create syncd-counter --name=syncd
admin@sonic:~$ sudo config cpu-time-report counter remove orchagent-counter
```

```
admin@sonic:~$ sudo config cpu-time-report collector create my_collector --address=10.0.0.1 --port=8888
admin@sonic:~$ sudo config cpu-time-report collector remove my_collector
```

Show for predefined keys:
```
admin@sonic:~$ show cpu-time-report global
Enabled    Polling interval (s)
---------  ---------
True       5
```

Show for process and collectors:

```
admin@sonic:~$ show cpu-time-report counter
Name                Process Name   
------------------  ---------------
orchagent-counter   orchagent
syncd-counter       syncd
```

```
admin@sonic:~$ show cpu-time-report collector
Name           Address     Port
-------------  ----------  ------
my_collector   10.0.0.1    8888
``` 

## Arbitary commands execution within docker container

Some features may require executing a seperate command within a docker container. This command should be a subcommand in SONiC command line interface.
Therefore, additional file is required to autogenerate such commands - *additional_commands.json*.

### additional_commands.json schema definition

additional_commands.json is used to generate CLI commands that wrap a binary inside a docker container.
Example usage for non-trivial show commands, which do not look at database.

The following table describes schema for version 1.0:

Path | Type | Description
--- | --- | ---
/version | string | Version of schema definition
/additional_commands | array | Array of additional commands
/additional_commands/{{index}}/executable | string | Executable name inside container, absolute path or name if binary is in the $PATH
/additional_commands/{{index}}/name | string | Name of the command, used for CLI command generation or REST API path
/additional_commands/{{index}}/description | string | Description of the command
/additional_commands/{{index}}/operation | string | Valid operations, "get", "del", which are mapped to CLI "show", "sonic-clear" or for REST API maps to GET, DELETE
/additional_commands/{{index}}/positional_arguments | array | Array of positional arguments passed to binary
/additional_commands/{{index}}/positional_arguments/{{index}}/name | string | Positional argument name
/additional_commands/{{index}}/positional_arguments/{{index}}/description | string | Positional argument description
/additional_commands/{{index}}/positional_arguments/{{index}}/value/type | string | Type of value, one of "intrange"/"int"/"bool"/"choice"/"list"/"reference"/"ipaddress"
/additional_commands/{{index}}/positional_arguments/{{index}}/value/intmin | int | Minimum integer value, valid if type is "intrange"
/additional_commands/{{index}}/positional_arguments/{{index}}/value/intmax | int | Maximum integer value, valid if type is "intrange"
/additional_commands/{{index}}/positional_arguments/{{index}}/value/choice_list | array | List of choices, valid if type is "choice" or "list"
/additional_commands/{{index}}/positional_arguments/{{index}}/value/choice_table | string | Table name, user interface will generate a validation wether value is valid key inside given table, valid if type is "reference"
/additional_commands/{{index}}/optional_arguments | array | Array of positional arguments passed to binary
/additional_commands/{{index}}/optional_arguments/{{index}}/name | string | Optional argument name (e.g. "-c", "--collector")
/additional_commands/{{index}}/optional_arguments/{{index}}/description | string | Optional argument description
/additional_commands/{{index}}/optional_arguments/{{index}}/value/type | string | Type of value, one of "intrange"/"int"/"bool"/"choice"/"list"/"reference"/"ipaddress"
/additional_commands/{{index}}/optional_arguments/{{index}}/value/intmin | int | Minimum integer value, valid if type is "intrange"
/additional_commands/{{index}}/optional_arguments/{{index}}/value/intmax | int | Maximum integer value, valid if type is "intrange"
/additional_commands/{{index}}/optional_arguments/{{index}}/value/choice_list | array | List of choices, valid if type is "choice" or "list"
/additional_commands/{{index}}/optional_arguments/{{index}}/value/choice_table | string | Table name, user interface will generate a validation wether value is valid key inside given table, valid if type is "reference"


Example for cpu-time-report:

```json
{
  "version": "1.0",
  "additional_commands": [
    {
      "executable": "/usr/bin/generate_dump",
      "positional_arguments": [
      ],
      "optional_arguements": [
      ],
      "name": "dump",
      "description": "generate process monitoring dump",
      "default": true,
      "operation": [
        "get"
      ]
    },
    {
      "executable": "/usr/bin/clear_stats",
      "positional_arguments": [
      ],
      "optional_arguements": [
      ],
      "name": "stats",
      "description": "Clear process statistic",
      "operation": [
        "del"
      ]
    }
  ]
}
```

Will generate the following command:

```
admin@sonic:~$ show cpu-time-report dump
admin@sonic:~$ show cpu-time-report      # same because 'dump' is a default subcommand
```

```
admin@sonic:~$ sonic-clear cpu-time-report stats
```

## SONiC application extention entrypoint and process management

SONiC application extention will be managed by SONiC supervisor, therefore it **must** provide a list of critical applications in processes.json, their arguments, and optional pre- and post-start commands to be executed within a container.

Path | Type | Description
--- | --- | ---
/prestart | array | List of pre-start commands to execute
/prestart/{{index}}/command | string | Command string
/processes | array | List processes for supervisor
/processes/{{index}}/path | string | Binary path
/processes/{{index}}/arguments | string | Process arguments
/poststart | array | List of post-start commands to execute
/poststart/{{index}}/command | string | Command string
/stop | array | List of stop commands to execute, NOTE: there is no pre-, post-stop. Stop is when supervisord terminates, no control wether command is run before processes are terminated or after.
/stop/{{index}}command | string | Command string

For example:
```
{
    "prestart": [
        "/usr/bin/command1",
        "/usr/bin/command2"
    ],
    "processes": [
        {
            "path": "/usr/bin/command3",
            "arguments": "--arg1 arg1"
        },
    ],
    "poststart": [
        "/usr/bin/command3",
        "/usr/bin/command4"
    ]
}
```

SONiC extention infrastructure will take care of generating all the `supervisord` files and start a container with the appropriate entrypoint to make `supervisord` manage the processes execution.

During installation procedure, SONiC extention installation tool will generate *supervisord.conf* based on *processes.json* file.

# SONiC build system support

SONiC build system has to be extended with support to build external extentions.
SONiC build system will provide tree docker images to be used as a base to build SONiC extentions - *sonic-sdk-buildenv*, *sonic-sdk* and *sonic-sdk-dbg*.

*sonic-sdk-buildenv* will have common SONiC packages required to build SONiC application extention and will be a minimal version of sonic-slave container:

- build-essential
- libhiredis-dev
- libnl*-dev
- libswsscommon-dev
- libsairedis-dev
- libsaimeta-dev
- libsaimetadata-dev

*sonic-sdk* will be based on *docker-config-engine* with addition of packages needed at run time:

- libhiredis
- libnl*
- libswsscommon
- libsairedis
- libsairedis
- libsaimeta
- libsaimetadata

Corresponding *-dbg* packages will be added for *sonic-sdk-dbg* image.
A list of packages will be extended on demand when a common package is required by community.
If a package is required but is specific to the SONiC application extention it should not be added to this list.

Developer of SONiC application extention can use those docker images to build his extention.
It would be highly encouraged to laverage docker multi-staged build feature in this case:

```Dockerfile

FROM sonic-sdk-buildenv:202006.1-583bfde4 as build-env

# Building packages here, like feature_1.0.0_amd64.deb

FROM sonic-sdk:202006.1-583bfde4 as run-env

# Copy artifacts from build environment 
COPY --from=build-env ["feature_1.0.0_amd64.deb", "/"]

# Installing packages

LABEL tag=202006.0-af12ae64
```

# SONiC extention versioning

*sonic-sdk* docker image will embed a label *sonic-version*, which *must* start with release number:

```Dockerfile
LABEL sonic-version=202006.0-af12ae64
```

This will give a way to failure or warn the user during extention installation in case *sonic-version* label does not match SONiC version release.


SONiC extention maintainer *must* be using the following convention when tagging docker images:

Tag | Comment
--- | ---
latest | The most recent version working for SONiC master
release-number (e.g. 202006) | The most recent version working for SONiC
anything-else | Specific version, version *may* be not neccesary to contain SONiC release number

# Open questions

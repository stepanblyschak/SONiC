# SONiC extentions HLD

## High Level Design Document

# Table of Contents

# List of Tables

# List of Figures

# About this Manual

This document provides an overview of the design of implementation of SONiC extentions.

# Scope

This document describes the high level design of SONiC extentions infrastructure.

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

```
name        = 1*ALPHA ;
repository  = 1*ALPHA ;
description = 1*ALPHA ;
status      = "installed"/"not installed"/"installation error"
```

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

sonic-extention CLI should prevent user from deleting a repository with extention installed.

Installation commands example:
```
admin@sonic:~$ # install guarantied compatible featureA
admin@sonic:~$ sudo sonic-extention install featureA
admin@sonic:~$ # install latest version of featureA
admin@sonic:~$ sudo sonic-extention install featureA --tag=latest
```

After installation, user can enable the feature:

```
admin@sonic:~$ sudo config feature featureA enabled 
```

An option "--tag" in CLI will allow user to use latest feature version.
```show version``` command can be used to display feature docker image version.

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

## manifest.json schema

```
; manifest schema definition
version       = 1*DIGIT.1*DIGIT ; version of manifest definition
name          = string          ; SONiC extention name
requires      = 1*string        ; List of required SONiC core services
after         = 1*string        ; List of SONiC core services to start after
dockerOptions = 1*string        ; List of docker container run options
```

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
  "dockerOptions": [
    "--net=host",
  ]
}
```

## schema.json format definition

schema.json is used to define the database tables that are relevant to the feature:

```
; common definitions
string = 1*ALPHA

; values definition
type                  = "intrange"/"int"/"choice"/"list"/"reference" ; type of field value, can be easy mapped to click.Type for CLI
intmax                = 1*DIGIT  ; valid for type="intrange", defines a maximum valid for this field
intmin                = 1*DIGIT  ; valid for type="intrange", defines a minimum valid for this field
choiceList            = 1*string ; valid for type="choice", a list of strings of valid choices for this field
choiceTable           = string   ; valid for type="reference", database table name which has valid object for this field value
valueType             = (type [intmax] [intmin] [choiceList] [choiceTable])

; cli options definition
commands   = 1*("show"/"config") ; which commands to generate
subcommand = string ; subcommand name; defines wether to create a sub-command
                    ; if not specified will use a root command for feature
defaultsubcommand = BOOL ; whether defined subcommand is default subcommand, default is FALSE
clioptions = (commands subcommand [defaultsubcommand])

; fields definition
name                  = string    ; DB field name
prettyName            = string    ; optional pretty name for this field; for example, used as a column name for show CLI commands
description           = string    ; optional description for the field, used for generating help message for CLI
value                 = valueType ;
requiresServiceReload = BOOL      ; whether changing this field value requires service restart, default is FALSE
                                  ; when True, CLI promts for service restart.
field                 = (name value [prettyName] [description] [requiresServiceReload])

; key definition
type        = "predefined"/"object" ; defines a type of keys in the table
                                    ; whether keys are object/entries names or a predefined key
                                    ; (e.g. "global" key for global configuration)
name        = string  ; valid for type="predefined", a name for predefined key
description = string  ; optional description used to generate CLI help for example
fields      = 1*field ; list of fields defined for this type of key
key         = (type name fields [description])

; table definition
name       = string ; table name
keys       = 1*key ; list of keys definition in the table
cli        = clioptions ; cli generation options
table      = (name keys cli)

; db definition
name   = "CONFIG_DB" ; only for CONFIG_DB CLI is generated for now
tables = 1*table ; list of table definitions
db     = (name tables)

; schema definition
version = [0-9]+.[0-9]+ ; version of schema definition
dbs = 1*db ; list of database definitions
schema = (version dbs)
```

From the above data, it is possible to generate CLI commands.
Following common python practice to develop plugins there will be an empty package ```show.plugins``` and ```config.plugins```.
Generated commands will be put in a module as part of plugins package. show and config commands should be updated to autodiscover those CLI plugins.

Based on the above, the following CLI commands will be generated:

```
# for table of predefined keys
config <feature> <predefined-key> <field-name> <field-value>

# for table of objects
config <feature> <table-name> create [NAME] [--<field-name>=<field-value>]*

# show  for predefined keys
show <feature> <predefined-key>
FIELD #1   FIELD #2   FIELD #3
---------  ---------  ---------
VALUE #1   VALUE #2   VALUE #3


# show for table objects
show <feature> <table-name>
Name   FIELD #1   FIELD #2   FIELD #3
-----  ---------  ---------  ---------
obj1   value 1    value 2    value #3
``` 

Example of schema.json for featureA:

```json
{
  "schema": {
    "dbs": [
      {
        "name": "CONFIG_DB",
        "tables": [
          {
            "name": "FEATUREA",
            "keys": [
              {
                "type": "predefined",
                "name": "global",
                "description": "Feature A global configuration",
                "fields": [
                  {
                    "name": "field_1",
                    "prettyName": "Field 1",
                    "description": "Field 1 configuration",
                    "value": {
                      "type": "int"
                    }
                  },
                  {
                    "name": "field_2",
                    "prettyName": "Field 2",
                    "description": "Field 2 configuration",
                    "value": {
                      "type": "choice",
                      "choiceList": [
                        "X",
                        "Y",
                      ]
                    }
                  }
                ]
              }
            ],
            "clioptions": {
              "commands": [
                "show",
                "config"
              ]
            }
          },
          {
            "name": "FEATUREA_OBJECT",
            "keys": [
              {
                "type": "object",
                "description": "Feature A objects configuration",
                "fields": [
                  {
                    "name": "obj_field_1",
                    "prettyName": "Object Field 1",
                    "description": "Field 1 configuration",
                    "value": {
                      "type": "int"
                    }
                  }
                ]
              }
            ],
            "clioptions": {
              "commands": [
                "show",
                "config"
              ],
              "subcommand": "object"
            }
          }
        ]
      }
    ]
  }
}
```

The above example generates:

Table of predefined keys:
```
admin@sonic:~$ sudo config featureA global field-1 5
admin@sonic:~$ sudo config featureA global field-2 X
```

Table of objects:
```
admin@sonic:~$ sudo config featureA object create obj1 --obj-field-1=7
admin@sonic:~$ sudo config featureA object remove obj1
```

Show for predefined keys:
```
admin@sonic:~$ show featureA global
FIELD 1    FIELD 2
---------  ---------
5          X
```

Show for table objects:
```
admin@sonic:~$ show featureA object
Name   Object field 1   
-----  ---------------
obj1   7
```

## additional_commands.json schema definition

additional_commands.json is used to generate CLI commands that wrap a binary inside a docker container.
Example usage for non-trivial show commands, which do not look at database.

```
; argument definition
value       = valueType ; defined previously
name        = string ; name for positional, option name for optional (e.g. '-o'/'--option')
description = string
argument    = (value name description)

; additional commands definition
version           = [0-9]+.[0-9]+ ; version of commands definition
cli               = clioptions ; cli options for additional command generation
executable        = string ; name of executable inside container
positionalArgs    = 1*argument
optionalArgs      = 1*argument
```

Example for featureA:

```json
{
  "version": "1.0",
  "additionalCommands": [
    {
      "executable": "/usr/bin/featureA",
      "positionalArgs": [
        {
          "value": {
            "type": "reference",
            "choiceTable": "FEATUREA_OBJECT"
          },
          "name": "object",
          "description": "existing object from FEATUREA_OBJECT table"
        }
      ],
      "clioptions": {
        "commands": [
          "show",
        ],
        "subcommand": "execute",
        "defaultsubcommand": true
      }
    }
  ]
}
```

Will generate the following command:

```
admin@sonic:~$ show featureA execute
admin@sonic:~$ show featureA  # same because 'execute' is a default subcommand
```

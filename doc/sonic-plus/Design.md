# SONiC+ HLD

## High Level Design Document

# Table of Contents

# List of Tables

# List of Figures

# About this Manual

This document provides an overview of the design of implementation of SONiC+

# Scope

This document describes the high level design of SONiC+ infrastructure

# Definitions/Abbreviation

# Overview

SONiC+ - a SONiC infrastructure for building and integrating third-party out-of-tree services/features that provide their functionality as a docker container managed as a systemd service.

# Requirements

- Extention provides its functionality as another docker container running alongside SONiC core services
- Extension is distributed as a docker image available though Docker Hub or private registry
- SONiC+ services is an extension to existing *feature* conecpt already available in SONiC, that means, a SONiC+ service is a subject to the same handlers as for *feature*s
- SONiC+ infrastructure provides a CLI interface to install, remove, list available SONiC+ extensions
- Installed SONiC+ extenstions migrate during SONiC-2-SONiC upgrades
- An extention is backward and forward compatible between SONiC releases

# Design

Besides of a docker image, there are the following parts of the host OS that are missing:

- systemd service
- CLI sub-commands for *show*, *config*, *sonic-clear*, etc.



---
layout: post
title: RedFone & FreeSWITCH high availability
---

{{ page.title }}

### Introduction

[High availability]https://en.wikipedia.org/wiki/High_availability is a system design and implementation approach geared towards keeping online resources available through node failure or system maintenance. Online services including their IP address can be migrated to backup nodes at any time, due to failures in the active node or the need of administrative maintenance. 

This article describes how to configure a simple active/passive failover HA setup for a Fonebridge-FreeSWITCH system using Heartbeat and Pacemaker. 

Heartbat is a daemon that allows the nodes to know about the presence or disappearance of peer processes on other machines and to easily exchange messages between them.

The heartbeat daemon needs to be combined with a cluster resource manager (CRM) which has the task of starting and stopping services (moving IP addresses, starting and stopping init scripts) so that the highly available setup can be sustained. [Pacemaker]http://clusterlabs.org/wiki/Main_Page is the cluster resource manager used in our implementation.

### Equipment Overview, Hardware Requirements

We will use two servers or Virtual Machines preferrably residing on diferent physical hosts and two network interfaces on each server or VM. A third ethernet interface could also be used to isolate heartbeat related traffic between the two nodes, but in our case this traffic is configured unicast between the nodes and a subinterface is used for the floating IP assignment which in turn will be the one to which services will bind.

In short the following schema will be used:

#### Nodes and OS

freeswitch-dev - Primary active node, Debian Squeeze 6.0 - 2.6.32-5-amd64
ubuntu-v20z    - Secondary backup node, Ubuntu 10.04.3 LTS - 2.6.32-21-generic-pae

#### IP schema

10.101.20.110 - eth0 on freeswitch-dev
10.101.20.220 - eth0 on ubuntu-v20z
10.101.20.219 - eth0:0 Floating IP on active node


### Required Packages: Installation

```
apt-get update
apt-get upgrade
apt-get install heartbeat pacemaker
```
On Debian Squeeze you may get the following right after installation

```
Setting up heartbeat (1:3.0.3-2) ...

Heartbeat not configured: /etc/ha.d/ha.cf not found. Heartbeat failure [rc=1]. Failed.
```

It only means that the config files were not found when trying to start the Heartbeat service, possibly a bug being worked on.





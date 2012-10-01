---
layout: post
title: RedFone & FreeSWITCH high availability
---

{{ page.title }}

### Introduction

[High availability](https://en.wikipedia.org/wiki/High_availability) is a system design and implementation approach geared towards keeping online resources available through node failure or system maintenance. Online services including their IP address can be migrated to backup nodes at any time, due to failures in the active node or the need of administrative maintenance. 

This article describes how to configure a simple active/passive failover HA setup for a Fonebridge-FreeSWITCH system using Heartbeat and Pacemaker. 

Heartbat is a daemon that allows the nodes to know about the presence or disappearance of peer processes on other machines and to easily exchange messages between them.

The heartbeat daemon needs to be combined with a cluster resource manager (CRM) which has the task of starting and stopping services (moving IP addresses, starting and stopping init scripts) so that the highly available setup can be sustained. [Pacemaker](http://clusterlabs.org/wiki/Main_Page) is the cluster resource manager used in our implementation.

### Equipment Overview, Hardware Requirements

We will use two servers or Virtual Machines preferrably residing on diferent physical hosts and two network interfaces on each server or VM. A third ethernet interface could also be used to isolate heartbeat related traffic between the two nodes, but in our case this traffic is configured unicast between the nodes and a subinterface is used for the floating IP assignment which in turn will be the one to which services will bind.

In short the following schema will be used:


|Node|Role|Distro|
|:--------------------|:--------------------|:---------------------|
|freeswitch-dev| Primary active node| Debian Squeeze 6.0
|ubuntu-v20z| Secondary backup node| Ubuntu 10.04.3 LT
.


#### IP schema

* 10.101.20.110 - eth0 on freeswitch-dev
* 10.101.20.220 - eth0 on ubuntu-v20z
* 10.101.20.219 - eth0:0 Floating IP on active node

.

### Required Packages: Installation


    #shell>apt-get update
    #shell>apt-get upgrade
    #shell>apt-get install heartbeat pacemaker


On Debian Squeeze you may get the following right after installation

    Setting up heartbeat (1:3.0.3-2) ...
    Heartbeat not configured: /etc/ha.d/ha.cf not found. Heartbeat failure [rc=1]. Failed.


It only means that the config files were not found when trying to start the Heartbeat service, possibly a bug being worked on.


### Configure Heartbeat

On the primary HA node, create a file named /etc/ha.d/ha.cf with the following contents. Replace the IP with the statically assigned IP of the secondary node. On Debian you can find a thoroughly documented ha.cf example at /usr/share/doc/heartbeat/ha.cf.gz

***File:***/etc/ha.d/ha.cf (on primary node)

    logfacility daemon
    keepalive 2
    deadtime 15
    warntime 5
    initdead 120
    udpport 694
    ucast eth0 10.101.20.220
    auto_failback on
    node freeswitch-dev
    node ubuntu-v20z
    use_logd yes
    crm respawn


On the secondary HA node create the equivalent /etc/ha.d/ha.cf file replacing the IP with the statically assigned one pointing to the primary node.


***File:***/etc/ha.d/ha.cf (on secondary node)

    logfacility daemon
    keepalive 2
    deadtime 15
    warntime 5
    initdead 120
    udpport 694
    ucast eth0 10.101.20.220
    auto_failback on
    node freeswitch-dev
    node ubuntu-v20z
    use_logd yes
    crm respawn


Again on primary HA node create the file /etc/ha.d/authkeys with the following content.


    auth 1
    1 sha1 VeryStrongPassword


***Note:*** The 'VeryStrongPassword' value is the secret key shared between nodes

Adjust file permissions as follows:

    #shell>chmod 600 /etc/ha.d/authkeys


Then copy this file to the secondary HA node:



    #shell>scp /etc/ha.d/authkeys root@ubuntu-v20z:/etc/ha.d/
    #shell>ssh root@ubuntu-v20z "chmod 600 /etc/ha.d/authkeys"


At this point you can go ahead and start the heartbeat services on both nodes



    #shell>service heartbeat start
    #shell>ssh root@ubuntu-v20z "service heartbeat start"
.


### Configure Cluster Resources

By making use of the Heartbeat messaging and membership capabilities the resource manager will execute scripts and move resources to its convenience in order to achieve maximum availability of services.

This is probably the most cryptic part of the configuration but you can find extensive quality documentation from [Clusterlabs] at [1]

[1]:http://www.clusterlabs.org/doc/en-US/Pacemaker/1.0/html/Pacemaker_Explained/s-resource-supported.html "ClusterLabs"

On the primary node issue the following command which will fire up the text editor set by the environment variable EDITOR. This will most likely be vi unless you have set it differently.

    crm configure edit

You will be presented with information resembling the one below, remember this would be a vi environment.

    node $id="819efb52-62e8-43f6-a9e5-b3ec34de868f" ubuntu-v20z
    node $id="a2a664c3-94b4-4b06-b664-c2aeebb23bba" freeswitch-dev
    property $id="cib-bootstrap-options" \
        dc-version="1.0.9-74392a28b7f31d7ddc86689598bd23114f58978b" \
        cluster-infrastructure="Heartbeat"

Edit the text above so that your complete configuration resembles the following:

    node $id="819efb52-62e8-43f6-a9e5-b3ec34de868f" ubuntu-v20z
    node $id="a2a664c3-94b4-4b06-b664-c2aeebb23bba" freeswitch-dev
    primitive fs lsb:FSSofia \
            op monitor interval="2" timeout="8" \
            meta target-role="Started"
    primitive ip_shared ocf:heartbeat:IPaddr2 \
            params ip="10.101.20.219" nic="eth0:0"
    primitive ip_shared_arp ocf:heartbeat:SendArp \
            params ip="10.101.20.219" nic="eth0:0"
    group VoiceServices ip_shared ip_shared_arp fs
    location cli-prefer-VoiceServices VoiceServices \
            rule $id="cli-prefer-rule-VoiceServices" inf: #uname eq ubuntu-v20z
    property $id="cib-bootstrap-options" \
            dc-version="1.0.9-74392a28b7f31d7ddc86689598bd23114f58978b" \
            cluster-infrastructure="Heartbeat" \
            expected-quorum-votes="1" \
            stonith-enabled="false" \
            no-quorum-policy="ignore" \
            last-lrm-refresh="1299964019"
    rsc_defaults $id="rsc-options" \
            resource-stickiness="100"

Of particular importance are the primitive clauses which define resource agents. You can think of a resource agent simply as a script (shell or even Phython, Perl) that presents a view of resources to the cluster, for example by providing a status CLI option. There are three classes of resources that Pacemaker support and we are using two of the three supported classes in our primitive definitions: Linux Standard Base (LSB) and Open Cluster Framework (OCF).

The first primitive definition uses an LSB resource, the kind of those found in /etc/init.d. We will conveniently turn to the community resources and look at the freeswitch-contrib repo.  There is an LSB resource named FSsofia in the directory ```ledr/various/ha.d/resources.d``` which we will use as a base for our LSB resource. Copy this script to /etc/init.d on our primary node and modify so that the final script looks as follows.

<script src="https://gist.github.com/3795649.js?file=FSSofia"></script>

So what will happen is that the freeswitch service will be started on both nodes and once a failure or sys admin switchover happens then Pacemaker will restart FreeSWITCH profiles move the floating IP to the new node and send an ARP accordingly to reflect the change on the IP.

You may need to allow the FreeSWITCH service to bind to a non local IP which on Debian can be achived as follows:

    echo 1 > /proc/sys/net/ipv4/ip_nonlocal_bind

or

    sysctl to set net.ipv4.ip_nonlocal_bind = 1

.


### Monitor Cluster Resources

At this time you can issue the ```crm_mon``` command to start the cluster monitor and you will be able to check the current state of the cluster. The DC (Designated Cluster) node is where all the decisions are made and if the current DC fails, a new one is elected from the remaining cluster nodes.

<script src="https://gist.github.com/3801277.js"> </script>



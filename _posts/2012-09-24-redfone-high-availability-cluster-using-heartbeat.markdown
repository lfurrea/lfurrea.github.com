---
layout: post
title: RedFone & FreeSWITCH high availability
---

{{ page.title }}

### Introduction

[High availability](https://en.wikipedia.org/wiki/High_availability) is a system design and implementation approach geared towards keeping online resources available through node failure or system maintenance. Online services including their IP address can be migrated to backup nodes at any time, due to failures in the active node or the need of administrative maintenance. 

This article describes how to configure a simple active/passive failover HA setup for a Fonebridge-FreeSWITCH system using Corosync and Pacemaker. 

[Corosync](http://www.corosync.org) is a daemon that allows member nodes to know about the presence or disappearance of peer processes on other machines and to easily exchange messages between them.

Corosync was chosen over [Heartbeat](http://linux-ha.org/wiki/Heartbeat) since development is very active for the project as opposed to Heartbeat for which development seems to have slowed down and seems to be only maintenance oriented, but this was only a personal choice.

The Corosync daemon needs to be combined with a cluster resource manager (CRM) which has the task of starting and stopping services (moving IP addresses, starting and stopping init scripts) so that the highly available setup can be sustained. [Pacemaker](http://clusterlabs.org/wiki/Main_Page) is the cluster resource manager used in our implementation.



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
    #shell>apt-get install corosync pacemaker


### Networking Config

freeswitch-dev node
    auto eth0
    iface eth0 inet static
        address 10.101.20.110
        netmask 255.255.255.0
        network 10.101.20.0
        broadcast 10.101.20.255
        gateway 10.101.20.1

ubuntu-v20z

    auto eth0
    iface eth0 inet static
        address 10.101.20.220
        netmask 255.255.255.0
        network 10.101.20.0
        broadcast 10.101.20.255
        gateway 10.101.20.1

There is no need for any configuration on eth0:0 as it will be managed by pacemaker.

Confirm that you can communicate between the two nodes:

    freeswitch-dev:~# ping -c 3 10.101.20.220
    PING 10.101.20.220 (10.101.20.220) 56(84) bytes of data.
    64 bytes from 10.101.20.220: icmp_req=1 ttl=64 time=0.156 ms
    64 bytes from 10.101.20.220: icmp_req=2 ttl=64 time=0.147 ms
    64 bytes from 10.101.20.220: icmp_req=3 ttl=64 time=0.148 ms

    --- 10.101.20.220 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 1998ms
    rtt min/avg/max/mdev = 0.147/0.150/0.156/0.010 ms

Now we need to make sure that we can communicate between nodes by their names. If using DNS make sure you add the proper entries, otherwise modify ```/etc/hosts``` to resemble the following on each node:

    10.101.20.110   freeswitch-dev.redfonelabs.com freeswitch-dev
    10.101.20.220   ubuntu-v20z.redfonelabs.com ubuntu-v20z

We can then verify communication by using names:

    freeswitch-dev:~# ping -c 3 ubuntu-v20z
    PING ubuntu-v20z.redfonelabs.com (10.101.20.220) 56(84) bytes of data.
    64 bytes from ubuntu-v20z.redfonelabs.com (10.101.20.220): icmp_req=1 ttl=64 time=0.139 ms
    64 bytes from ubuntu-v20z.redfonelabs.com (10.101.20.220): icmp_req=2 ttl=64 time=0.141 ms
    64 bytes from ubuntu-v20z.redfonelabs.com (10.101.20.220): icmp_req=3 ttl=64 time=0.136 ms

    --- ubuntu-v20z.redfonelabs.com ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 1998ms
    rtt min/avg/max/mdev = 0.136/0.138/0.141/0.013 ms



### Configure Corosync

On the primary HA node, use the corosync example file as base to create the  ```cp /etc/corosync/corosync.conf.example /etc/corosync/corosync.conf```. Replace the bind IP with the statically assigned IP of the local node. You can leave the default suggested Multicast address and port for communication between nodes.

***File:***/etc/corosync/corosync.conf (on primary node)

<script src="https://gist.github.com/3837137.js"> </script>


On the secondary HA node create the equivalent /etc/ha.d/ha.cf file replacing the bind IP with the statically assigned one of the secondary node.


***File:***/etc/corosync/corosync.conf (on secondary node)

<script src="https://gist.github.com/3837151.js"> </script>


Since we're on Debian based distros modify the /etc/default/corosync file to enable startup of the corosync daemon. Initially on the primary node:

    #shell>service start corosync


Then check if cluster communication is working:

    #shell>corosync-cfgtool -s

    freeswitch-dev:~# corosync-cfgtool -s
    Printing ring status.
    Local node ID 1846830346
    RING ID 0
        id      = 10.101.20.110
        status  = ring 0 active with no faults

Everything looks normal and our fixed IP is listed over the 127.0.0.x loopback, also the status reports '''no faults'''

If your output looks different initial checks would be on the network settings, firewall or script startup.

You can then start Corosync on the secondary node and run the same checks.

    #shell>ssh ubuntu-v20z "service start corosync"
    #shell>ssh ubuntu-v20z "corosync-cfgtool -s"

    Printing ring status.
    Local node ID 1544840458
    RING ID 0
        id      = 10.101.20.220
        status  = ring 0 active with no faults

Everything looking healthy and therefore we can go ahead and check the cluster membership.

    freeswitch-dev:~# corosync-quorumtool -l
    Nodeid     Votes  Name
    1846830346     1  freeswitch-dev.redfonelabs.com
    1544840458     1  ubuntu-v20z.redfonelabs.com
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



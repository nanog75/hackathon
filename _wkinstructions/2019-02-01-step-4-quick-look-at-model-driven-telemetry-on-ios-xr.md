---
published: true
date: '2019-02-01 05:03 -0400'
title: 'Step 4: Quick look at Model Driven Telemetry on IOS-XR'
author: Akshat Sharma
excerpt: >-
  Short tutorial on Model Driven Telemetry on IOS-XR  - setting up a connection,
  testing onbox and trying out pipeline
tags:
  - iosxr
  - cisco
  - telemetry
  - lab
---

{% include toc %} 

>Have a look at the set of Telemetry learning labs on DevNet for more details on Telemetry,
>available workflows, configurations and usage of tools:
><https://learninglabs.cisco.com/modules/iosxr-streaming-telemetry>


# Setting up a Telemetry Client/Collector


Let's consider the options available to set up a Telemetry client/collector that can receive Streaming data from the router.

## Open-Source tool: Pipeline

Pipeline is a flexible, multi-function collection service that is written in Go. It can ingest telemetry data from any XR release starting from 6.0.1. Pipeline’s input stages support raw UDP and TCP, as well as gRPC dial-in and dial-out capability. For encoding, Pipeline can consume JSON, compact GPB and self-describing GPB. On the output side, Pipeline can write the telemetry data to a text file as a JSON object, push the data to a Kafka bus and/or format it for consumption by open source stacks. Pipeline can easily be extended to include other output stages and we encourage contributions from anyone who wants to get involved.



<img src="{{site.baseurl}}/images/telemetry_stack.png" alt="model-driven"  style=" margin-left: auto; margin-right: auto; display: block;" align="middle"/>

It’s important to understand that Pipeline is not a complete big data analytics stack. Think of it as the first layer in a scalable, modular, analytics architecture. Depending on your use case, that architecture would also include separate components for big data storage, stream processing, analysis, alerting and visualization.


### Supported Capabilities

Pipeline is the most comprehensive tool available for IOS-XR telemetry data consumption. It is Golang–based code which consumes IOS XR telemetry streams directly from routers or indirectly from a pub/sub bus (Kafka). Once collected, Pipeline performs transformations of the data and forwards the result to the configured consumer.

Pipeline supports different input transport formats from routers (please be aware that multiple input modules of any type can run in parallel):

* TCP
* gRPC
* UDP
* Apache Kafka

Pipeline can support different encodings as well:

* (compact) GPB
* KV-GPB
* JSON

Pipeline can stream data to several different consumers. Supported downstream consumers include:

* InfluxDB (TSDB)
* Prometheus (TSDB)
* Apache Kafka
* dump-to-file (mostly for diagnostics purposes)

>Connect to your Pod first! Make sure your Anyconnect VPN connection to the Pod assigned to you is active. 
>
> If you haven't connected yet, check out the instructions to do so here: 
><https://iosxr-lab-ciscolive.github.io/LTRSPG-2414-cleur2019/assets/CLEUR19-AkshatSharma-IOS-XR-Programmability-Session-1-Friday.pdf>
>
>
> Once you're connected, use the following instructions to connect to the individual nodes.
> The instructions in the workshop will simply refer to the Name of the box to connect without
> repeating the connection details and credentials. So refer back to this list when you need it.
>  
>
> The 3 nodes in the topology are: 
> 
><p style="font-size: 16px;"><b>Development Linux System (DevBox)</b></p> 
>      IP Address: 10.10.20.170
>      Username/Password: [admin/admin]
>      SSH Port: 2211
> 
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R1: (Router r1)</b></p> 
>
>     IP Address: 10.10.20.170  
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170  
>     XR SSH Port: 2221    
>     NETCONF Port: 8321   
>     gRPC Port: 57021  
>     XR-Bash SSH Port: 2222    
>
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R2:  (Router r2)</b></p> 
>
>     IP Address: 10.10.20.170   
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170   
>     XR SSH Port: 2231    
>     NETCONF Port: 8331   
>     gRPC Port: 57031    
>     XR-Bash SSH Port: 2232
{: .notice--info}



The Topology in use is shown below:
![topology_devnet.png]({{site.baseurl}}/images/topology_devnet.png) 



## Testing Telemetry Yang Paths on-box

In this section, we intend to enable IPv6 on b2b interfaces on the two routers (`r1` and `r2`) and stream IPv6 neighbor information to an onbox utility to verify the data.   

We will construct a `dial-in` Telemetry collector/client wherein the client initiates a connection to the router and then subscribes to a pre-configured subscription on the router.    

### Set up IPv6 Neighbors

To start off, let's enable IPv6 and configure IPv6 addresses on the b2b interfaces of r1 and r2.


#### Configure IPv6 on router r1  

Connect to router `r1`:  

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Username</b>: admin<br/><b>Password</b>: admin<br/><b>SSH port</b>: 2221<br/><b>IP</b>: 10.10.20.170
</p>  


```
Laptop-terminal:$ ssh -p 2221 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password:


RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```

Apply the following configuration on router `r1`:

```
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#conf t
Mon Sep  3 09:00:33.672 UTC
RP/0/RP0/CPU0:r1(config)#interface GigabitEthernet 0/0/0/0
RP/0/RP0/CPU0:r1(config-if)#ipv6 enable
RP/0/RP0/CPU0:r1(config-if)#ipv6 address 1010:1010::10/64
RP/0/RP0/CPU0:r1(config-if)#no shut
RP/0/RP0/CPU0:r1(config-if)#      
RP/0/RP0/CPU0:r1(config-if)#
RP/0/RP0/CPU0:r1(config-if)#exit
RP/0/RP0/CPU0:r1(config)#
RP/0/RP0/CPU0:r1(config)#interface GigabitEthernet 0/0/0/1
RP/0/RP0/CPU0:r1(config-if)#ipv6 enable                      
RP/0/RP0/CPU0:r1(config-if)#ipv6 address 2020:2020::10/64    
RP/0/RP0/CPU0:r1(config-if)#no shut
RP/0/RP0/CPU0:r1(config-if)#commit
Mon Sep  3 09:01:59.484 UTC
RP/0/RP0/CPU0:r1(config-if)#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show  configuration commit changes last 1
Mon Sep  3 09:02:08.867 UTC
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface GigabitEthernet0/0/0/0
 ipv6 address 1010:1010::10/64
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv6 address 2020:2020::10/64
 ipv6 enable
 no shutdown
!
end

RP/0/RP0/CPU0:r1#

```

#### Configure IPv6 on router r2

Similarly, connect to router `r2`:

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Username</b>: admin<br/><b>Password</b>: admin<br/><b>SSH port</b>: 2231<br/><b>IP</b>: 10.10.20.170
</p>  


```
Laptop-terminal:$ ssh -p 2231 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 2 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password:


RP/0/RP0/CPU0:r2#
RP/0/RP0/CPU0:r2#

```

Apply the following configuration to router `r2`:

```
RP/0/RP0/CPU0:r2#
RP/0/RP0/CPU0:r2#conf t
Mon Sep  3 09:04:32.472 UTC
RP/0/RP0/CPU0:r2(config)#interface GigabitEthernet 0/0/0/0
RP/0/RP0/CPU0:r2(config-if)#ipv6 enable
RP/0/RP0/CPU0:r2(config-if)#ipv6 address 1010:1010::20/64
RP/0/RP0/CPU0:r2(config-if)#no shut
RP/0/RP0/CPU0:r2(config-if)#exit
RP/0/RP0/CPU0:r2(config)#interface GigabitEthernet 0/0/0/1
RP/0/RP0/CPU0:r2(config-if)#ipv6 enable
RP/0/RP0/CPU0:r2(config-if)#ipv6 address 2020:2020::20/64
RP/0/RP0/CPU0:r2(config-if)#no shut
RP/0/RP0/CPU0:r2(config-if)#commit
Mon Sep  3 09:05:37.371 UTC
RP/0/RP0/CPU0:r2(config-if)#
RP/0/RP0/CPU0:r2#
RP/0/RP0/CPU0:r2#show  configuration commit changes last 1
Mon Sep  3 09:05:51.846 UTC
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface GigabitEthernet0/0/0/0
 ipv6 address 1010:1010::20/64
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv6 address 2020:2020::20/64
 ipv6 enable
 no shutdown
!
end

RP/0/RP0/CPU0:r2#

```


#### Bring up IPv6 neighbors

IPv6 Neighbor Discovery (ND) works much the same way as ARP. Neighbor discovery messages are not sent out until traffic flow for a particular IPv6 destination needs to traverse the IPv6 enabled interface.  
So if we dump the `ipv6 neighbor` information on the two routers, we don't notice any reachable neighbors just yet.



Router r1:

```
RP/0/RP0/CPU0:r1#show  ipv6 neighbors
Mon Sep  3 09:13:17.796 UTC
IPv6 Address                             Age  Link-layer Add State Interface            Location
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/0            0/0/CPU0       
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/1            0/0/CPU0       
RP/0/RP0/CPU0:r1#
```

Router r2:

```
RP/0/RP0/CPU0:r2#show  ipv6 neighbors
Mon Sep  3 09:12:52.780 UTC
IPv6 Address                             Age  Link-layer Add State Interface            Location
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/0            0/0/CPU0       
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/1            0/0/CPU0       
RP/0/RP0/CPU0:r2#
```  

#### Ping the reachable Neighbors

We'll use pings (icmpv6) as the trigger to generate IPv6 ND messages and establish neighbors dynamically:  

On router `r1`:

```
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#ping 1010:1010::20
Mon Sep  3 09:25:45.079 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1010:1010::20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 95/125/231 ms
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#ping 2020:2020::20
Mon Sep  3 09:25:53.627 UTC
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2020:2020::20, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 97/125/214 ms
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
```

#### Verify neighbors are up  

The IPv6 neighbors should now be up!

```
RP/0/RP0/CPU0:r1#show  ipv6 neighbors
Mon Sep  3 09:26:07.306 UTC
IPv6 Address                             Age  Link-layer Add State Interface            Location
1010:1010::20                            21   5254.0093.8ab0 REACH Gi0/0/0/0            0/0/CPU0       
fe80::5054:ff:fe93:8ab0                  11   5254.0093.8ab0 REACH Gi0/0/0/0            0/0/CPU0       
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/0            0/0/CPU0       
2020:2020::20                            13   5254.0093.8ab1 REACH Gi0/0/0/1            0/0/CPU0       
fe80::5054:ff:fe93:8ab1                  2    5254.0093.8ab1 REACH Gi0/0/0/1            0/0/CPU0       
[Mcast adjacency]                           - 0000.0000.0000 REACH Gi0/0/0/1            0/0/CPU0       
RP/0/RP0/CPU0:r1#

```


### Configure Telemetry to Stream IPv6 neighbor information

The IPv6 neighbors displayed using `show ipv6 neighbors` can be streamed using the IPv6-ND oper YANG model in IOS-XR.  

<div style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"> The Yang models for IOS-XR per release can be found on Github here: <br/>
<blockquote>
<https://github.com/YangModels/yang/tree/master/vendor/cisco/xr>
</blockquote>  

In this lab we're working with IOS-XR release `6.4.1`:  

<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show  version
Mon Sep  3 09:46:31.320 UTC

Cisco IOS XR Software, Version 6.4.1
Copyright (c) 2013-2017 by Cisco Systems, Inc.

Build Information:
 Built By     : nkhai
 Built On     : Wed Mar 28 19:20:20 PDT 2018
 Build Host   : iox-lnx-090
 Workspace    : /auto/srcarchive14/prod/6.4.1/xrv9k/ws
 Version      : 6.4.1
 Location     : /opt/cisco/XR/packages/

cisco IOS-XRv 9000 () processor
System uptime is 2 hours, 28 minutes

RP/0/RP0/CPU0:r1#
</code></pre></div>  

The required Yang models for this release are thus available here:
<blockquote>
<https://github.com/YangModels/yang/tree/master/vendor/cisco/xr/641>
</blockquote>  
</div>

### Figure out the Yang Model/Path to use

The Yang model we're interested in is `Cisco-IOS-XR-ipv6-nd-oper.yang`, located here:
><https://github.com/YangModels/yang/blob/master/vendor/cisco/xr/641/Cisco-IOS-XR-ipv6-nd-oper.yang>  

The easiest way to understand the information available within the Yang model is to display the YANG model in a tree format using a tool called `pyang`.  

For this purpose, connect to the devbox:

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Username</b>: admin<br/><b>Password</b>: admin<br/><b>SSH port</b>: 2211<br/><b>IP</b>: 10.10.20.170
</p>  

```
Laptop-terminal:$ ssh -p 2211 admin@10.10.20.170
admin@10.10.20.170's password:
Last login: Mon Sep  3 01:28:30 2018 from 192.168.122.1
admin@devbox:~$
admin@devbox:~$

```

Clone the Yang git repo:  


```
admin@devbox:~$ git clone https://github.com/YangModels/yang
Cloning into 'yang'...
remote: Counting objects: 21847, done.
remote: Compressing objects: 100% (218/218), done.
remote: Total 21847 (delta 512), reused 709 (delta 505), pack-reused 21124
Receiving objects: 100% (21847/21847), 42.52 MiB | 13.17 MiB/s, done.
Resolving deltas: 100% (16807/16807), done.
Checking connectivity... done.
Checking out files: 100% (20153/20153), done.
admin@devbox:~$
admin@devbox:~$
admin@devbox:~$ cd yang/vendor/cisco/xr/641/
admin@devbox:641$ pwd
/home/admin/yang/vendor/cisco/xr/641
admin@devbox:641$

```


Dump the `Cisco-IOS-XR-ipv6-nd-oper.yang` model in a tree format:
  

```
admin@devbox:641$ pyang -f tree Cisco-IOS-XR-ipv6-nd-oper.yang
module: Cisco-IOS-XR-ipv6-nd-oper
   +--ro ipv6-node-discovery
      +--ro nodes
         +--ro node* [node-name]
            +--ro neighbor-interfaces
            |  +--ro neighbor-interface* [interface-name]
            |     +--ro host-addresses
            |     |  +--ro host-address* [host-address]
            |     |     +--ro host-address              inet:ipv6-address-no-zone
            |     |     +--ro last-reached-time
            |     |     |  +--ro seconds?   uint32
            |     |     +--ro reachability-state?       Ipv6-nd-sh-state
            |     |     +--ro link-layer-address?       yang:mac-address
            |     |     +--ro encapsulation?            Ipv6-nd-media-encap
            |     |     +--ro selected-encapsulation?   Ipv6-nd-media-encap
            |     |     +--ro origin-encapsulation?     Ipv6-nd-neighbor-origin
            |     |     +--ro interface-name?           string
            |     |     +--ro location?                 xr:Node-id
            |     |     +--ro is-router?                boolean
            |     |     +--ro serg-flags?               uint32
            |     |     +--ro vrfid?                    uint32
            |     +--ro interface-name    xr:Interface-name
            +--ro neighbor-summary
            |  +--ro multicast
            |  |  +--ro incomplete-entries?          uint32
            |  |  +--ro reachable-entries?           uint32
            |  |  +--ro stale-entries?               uint32
            |  |  +--ro delayed-entries?             uint32
            |  |  +--ro probe-entries?               uint32
            |  |  +--ro deleted-entries?             uint32
            |  |  +--ro subtotal-neighbor-entries?   uint32
            |  +--ro static
            |  |  +--ro incomplete-entries?          uint32
            |  |  +--ro reachable-entries?           uint32
            |  |  +--ro stale-entries?               uint32
            |  |  +--ro delayed-entries?             uint32
            |  |  +--ro probe-entries?               uint32
            |  |  +--ro deleted-entries?             uint32
            |  |  +--ro subtotal-neighbor-entries?   uint32
            |  +--ro dynamic
            |  |  +--ro incomplete-entries?          uint32
            |  |  +--ro reachable-entries?           uint32
            |  |  +--ro stale-entries?               uint32
            |  |  +--ro delayed-entries?             uint32
            |  |  +--ro probe-entries?               uint32
            |  |  +--ro deleted-entries?             uint32
            |  |  +--ro subtotal-neighbor-entries?   uint32
            |  +--ro total-neighbor-entries?   uint32
            +--ro bundle-nodes
            |  +--ro bundle-node* [node-name]
            |     +--ro node-name                   xr:Node-id
            |     +--ro age
            |     |  +--ro seconds?   uint32
            |     +--ro group-id?                   uint32
            |     +--ro process-name?               string
            |     +--ro sent-sequence-number?       uint32
            |     +--ro received-sequence-number?   uint32
            |     +--ro state?                      Ipv6-nd-bndl-state
            |     +--ro state-changes?              uint32
            |     +--ro sent-packets?               uint32
            |     +--ro received-packets?           uint32
            +--ro bundle-interfaces
            |  +--ro bundle-interface* [interface-name]
            |     +--ro interface-name           xr:Interface-name
            |     +--ro nd-parameters
            |     |  +--ro is-dad-enabled?               boolean
            |     |  +--ro dad-attempts?                 uint32
            |     |  +--ro is-icm-pv6-redirect?          boolean
            |     |  +--ro is-dhcp-managed?              boolean
            |     |  +--ro is-route-address-managed?     boolean
            |     |  +--ro is-suppressed?                boolean
            |     |  +--ro send-unicast-ra?              boolean
            |     |  +--ro nd-retransmit-interval?       uint32
            |     |  +--ro nd-min-transmit-interval?     uint32
            |     |  +--ro nd-max-transmit-interval?     uint32
            |     |  +--ro nd-advertisement-lifetime?    uint32
            |     |  +--ro nd-reachable-time?            uint32
            |     |  +--ro nd-cache-limit?               uint32
            |     |  +--ro complete-protocol-count?      uint32
            |     |  +--ro complete-glean-count?         uint32
            |     |  +--ro incomplete-protocol-count?    uint32
            |     |  +--ro incomplete-glean-count?       uint32
            |     |  +--ro dropped-protocol-req-count?   uint32
            |     |  +--ro dropped-glean-req-count?      uint32
            |     +--ro local-address
            |     |  +--ro ipv6-address?     inet:ipv6-address
            |     |  +--ro valid-lifetime?   uint32
            |     |  +--ro pref-lifetime?    uint32
            |     +--ro parent-interface-name?   xr:Interface-name
            |     +--ro iftype?                  uint32
            |     +--ro mtu?                     uint32
            |     +--ro etype?                   uint32
            |     +--ro vlan-tag?                uint16
            |     +--ro mac-addr-size?           uint32
            |     +--ro mac-addr?                yang:mac-address
            |     +--ro is-interface-enabled?    boolean
            |     +--ro is-ipv6-enabled?         boolean
            |     +--ro is-mpls-enabled?         boolean
            |     +--ro member-link*             uint32
            |     +--ro global-address*
            |     |  +--ro ipv6-address?     inet:ipv6-address
            |     |  +--ro valid-lifetime?   uint32
            |     |  +--ro pref-lifetime?    uint32
            |     +--ro member-node*
            |        +--ro node-name?     xr:Node-id
            |        +--ro total-links?   uint32
            +--ro interfaces
            |  +--ro interface* [interface-name]
            |     +--ro interface-name                xr:Interface-name
            |     +--ro is-dad-enabled?               boolean
            |     +--ro dad-attempts?                 uint32
            |     +--ro is-icm-pv6-redirect?          boolean
            |     +--ro is-dhcp-managed?              boolean
            |     +--ro is-route-address-managed?     boolean
            |     +--ro is-suppressed?                boolean
            |     +--ro send-unicast-ra?              boolean
            |     +--ro nd-retransmit-interval?       uint32
            |     +--ro nd-min-transmit-interval?     uint32
            |     +--ro nd-max-transmit-interval?     uint32
            |     +--ro nd-advertisement-lifetime?    uint32
            |     +--ro nd-reachable-time?            uint32
            |     +--ro nd-cache-limit?               uint32
            |     +--ro complete-protocol-count?      uint32
            |     +--ro complete-glean-count?         uint32
            |     +--ro incomplete-protocol-count?    uint32
            |     +--ro incomplete-glean-count?       uint32
            |     +--ro dropped-protocol-req-count?   uint32
            |     +--ro dropped-glean-req-count?      uint32
            +--ro nd-virtual-routers
            |  +--ro nd-virtual-router* [interface-name]
            |     +--ro interface-name        xr:Interface-name
            |     +--ro local-address
            |     |  +--ro ipv6-address?     inet:ipv6-address
            |     |  +--ro valid-lifetime?   uint32
            |     |  +--ro pref-lifetime?    uint32
            |     +--ro link-layer-address?   yang:mac-address
            |     +--ro context?              uint32
            |     +--ro state?                Ipv6-nd-sh-vr-state
            |     +--ro flags?                Ipv6-nd-sh-vr-flags
            |     +--ro vr-gl-addr-ct?        uint32
            |     +--ro vr-global-address*
            |        +--ro ipv6-address?     inet:ipv6-address
            |        +--ro valid-lifetime?   uint32
            |        +--ro pref-lifetime?    uint32
            +--ro slaac-interfaces
            |  +--ro slaac-interface* [interface-name]
            |     +--ro router-advert-detail
            |     |  +--ro idb?   xr:Interface-name
            |     |  +--ro ra*
            |     |     +--ro elapsed-ra-time
            |     |     |  +--ro seconds?   uint32
            |     |     +--ro reachable-time
            |     |     |  +--ro seconds?   uint32
            |     |     +--ro retrans-time
            |     |     |  +--ro seconds?   uint32
            |     |     +--ro address?           inet:ipv6-address
            |     |     +--ro hops?              uint32
            |     |     +--ro flags?             uint32
            |     |     +--ro life-time?         uint32
            |     |     +--ro mtu?               uint32
            |     |     +--ro err-msg?           boolean
            |     |     +--ro vrf-id?            uint32
            |     |     +--ro u6-tbl-id?         uint32
            |     |     +--ro rib-protoid?       uint16
            |     |     +--ro default-router?    boolean
            |     |     +--ro reachability?      uint32
            |     |     +--ro prefix-q*
            |     |        +--ro prefix-address?        inet:ipv6-address
            |     |        +--ro eui64?                 inet:ipv6-address
            |     |        +--ro valid-life-time?       uint32
            |     |        +--ro preferred-life-time?   uint32
            |     |        +--ro prefix-len?            uint32
            |     |        +--ro flags?                 uint32
            |     |        +--ro pfx-flags?             uint32
            |     +--ro interface-name          xr:Interface-name
            +--ro node-name              xr:Node-id
admin@devbox:641$

```

From the above dump, it becomes fairly clear that we intend to extract the following set of nodes from the model to match the information in the `show ipv6 neighbors` output:  

<div class="highlighter-rouge"><pre class="highlight"><code>
<mark>module: Cisco-IOS-XR-ipv6-nd-oper</mark>
   <mark>+--ro ipv6-node-discovery</mark>
      <mark>+--ro nodes</mark>
         <mark>+--ro node* [node-name]</mark>
            <mark>+--ro neighbor-interfaces</mark>
            <mark>|  +--ro neighbor-interface* [interface-name]</mark>
            <mark>|     +--ro host-addresses</mark>
            <mark>|     |  +--ro host-address* [host-address]</mark>
            |     |     +--ro host-address              inet:ipv6-address-no-zone
            |     |     +--ro last-reached-time
            |     |     |  +--ro seconds?   uint32
            |     |     +--ro reachability-state?       Ipv6-nd-sh-state
            |     |     +--ro link-layer-address?       yang:mac-address
            |     |     +--ro encapsulation?            Ipv6-nd-media-encap
            |     |     +--ro selected-encapsulation?   Ipv6-nd-media-encap
            |     |     +--ro origin-encapsulation?     Ipv6-nd-neighbor-origin
            |     |     +--ro interface-name?           string
            |     |     +--ro location?                 xr:Node-id
            |     |     +--ro is-router?                boolean
            |     |     +--ro serg-flags?               uint32
            |     |     +--ro vrfid?                    uint32

</code></pre></div>


### Setting up the Telemetry configuration  

The container path that we need to set up as part of the Telemetry configuration is highlighted above.
On the router, this becomes the following `sensor-path`:

```
!
telemetry model-driven
 sensor-group IPV6Neighbor
  sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
 !
```

Creating a subscription out of this path to stream the data every 15 seconds, we get the following additional configuration:

<div class="highlighter-rouge"><pre class="highlight"><code>
telemetry model-driven
 sensor-group IPV6Neighbor
  sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
 !
 <mark>subscription IPV6
  sensor-group-id IPV6Neighbor sample-interval 15000</mark>
 !
!
</code></pre></div>



If we wait a few seconds, we will notice that this sensor-path gets `Resolved` indicating that the router is now ready to send Telemetry data to an external collector.


<div class="highlighter-rouge"><pre class="highlight"><code>
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#show  telemetry model-driven subscription IPV6 internal
Mon Sep  3 10:50:20.451 UTC
Subscription:  IPV6
-------------
  State:       NA
  Sensor groups:
  Id: IPV6Neighbor
    Sample Interval:      15000 ms
    Sensor Path:          Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
    <mark>Sensor Path State:    Resolved</mark>

  Collection Groups:
  ------------------
  No active collection groups

RP/0/RP0/CPU0:r1#

</code></pre></div>


## Testing the Sensor path

Before we proceed, let's actually test the sensor path to make sure we're getting the relevant data. Starting with release 6.3.1, this can be done on the router itself  by using the following CLI command:

<div class="highlighter-rouge"><pre class="highlight"><code>
run mdt_exec -s &lt;your_sensor_path&gt; -c &lt;cadence&gt;
</code></pre></div>

Trying this out on router `r1` for the sensor_path=`Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address` and cadence=`2000` (2 seconds), we get:

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Just press `Enter` to terminate.
</p>  

```
RP/0/RP0/CPU0:r1#run mdt_exec -s Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address -c 2000
Mon Sep  3 17:54:56.475 UTC
Enter any key to exit...
 Sub_id 200000001, flag 0, len 0
 Sub_id 200000001, flag 4, len 3072
--------
{"node_id_str":"r1","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address","collection_id":4,"collection_start_time":1535997296669,"msg_timestamp":1535997296675,"data_json":[{"timestamp":1535997296674,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"1010:1010::20"},"content":{"last-reached-time":{"seconds":38},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997296674,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"fe80::5054:ff:fe93:8ab0"},"content":{"last-reached-time":{"seconds":179},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997296675,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997296676,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"2020:2020::20"},"content":{"last-reached-time":{"seconds":170},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997296676,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"fe80::5054:ff:fe93:8ab1"},"content":{"last-reached-time":{"seconds":165},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997296676,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}}],"collection_end_time":1535997296677}
--------
 Sub_id 200000001, flag 4, len 3068
--------
{"node_id_str":"r1","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address","collection_id":5,"collection_start_time":1535997298677,"msg_timestamp":1535997298684,"data_json":[{"timestamp":1535997298683,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"1010:1010::20"},"content":{"last-reached-time":{"seconds":40},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997298683,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"fe80::5054:ff:fe93:8ab0"},"content":{"last-reached-time":{"seconds":181},"reachability-state":"delay","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997298683,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997298685,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"2020:2020::20"},"content":{"last-reached-time":{"seconds":172},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997298685,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"fe80::5054:ff:fe93:8ab1"},"content":{"last-reached-time":{"seconds":167},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997298685,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}}],"collection_end_time":1535997298686}
--------

RP/0/RP0/CPU0:r1#

```


Let's just structure the data above a little better to see what we're
receiving. Drop into router `r1`'s  python shell and use the json module to "pretty" print it.

```
RP/0/RP0/CPU0:r1#bash
Mon Sep  3 17:58:00.770 UTC
[r1:~]$
[r1:~]$ python
Python 2.7.3 (default, Dec 12 2017, 08:22:03)
[GCC 4.9.1] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import json
>>>
>>> data="""{"node_id_str":"r1","subscription_id_str":"app_TEST_200000001","encoding_path":"Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address","collection_id":6,"collection_start_time":1535997300686,"msg_timestamp":1535997300703,"data_json":[{"timestamp":1535997300701,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"1010:1010::20"},"content":{"last-reached-time":{"seconds":42},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997300701,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"fe80::5054:ff:fe93:8ab0"},"content":{"last-reached-time":{"seconds":183},"reachability-state":"delay","link-layer-address":"5254.0093.8ab0","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997300701,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/0","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/0","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997300707,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"2020:2020::20"},"content":{"last-reached-time":{"seconds":174},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997300707,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"fe80::5054:ff:fe93:8ab1"},"content":{"last-reached-time":{"seconds":169},"reachability-state":"reachable","link-layer-address":"5254.0093.8ab1","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"dynamic","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":true,"serg-flags":255,"vrfid":1610612736}},{"timestamp":1535997300707,"keys":{"node-name":"0/0/CPU0","interface-name":"GigabitEthernet0/0/0/1","host-address":"ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"},"content":{"last-reached-time":{"seconds":0},"reachability-state":"reachable","link-layer-address":"0000.0000.0000","encapsulation":"arpa","selected-encapsulation":"arpa","origin-encapsulation":"static","interface-name":"Gi0/0/0/1","location":"0/0/CPU0","is-router":false,"serg-flags":255,"vrfid":1610612736}}],"collection_end_time":1535997300709}"""
>>>
>>>
>>> print json.dumps(json.loads(data), indent=4)
{
   "encoding_path": "Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address",
   "subscription_id_str": "app_TEST_200000001",
   "collection_start_time": 1535997300686,
   "msg_timestamp": 1535997300703,
   "collection_end_time": 1535997300709,
   "node_id_str": "r1",
   "data_json": [
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/0",
               "host-address": "1010:1010::20"
           },
           "timestamp": 1535997300701,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/0",
               "last-reached-time": {
                   "seconds": 42
               },
               "link-layer-address": "5254.0093.8ab0",
               "selected-encapsulation": "arpa",
               "is-router": true,
               "serg-flags": 255,
               "reachability-state": "reachable",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "dynamic"
           }
       },
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/0",
               "host-address": "fe80::5054:ff:fe93:8ab0"
           },
           "timestamp": 1535997300701,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/0",
               "last-reached-time": {
                   "seconds": 183
               },
               "link-layer-address": "5254.0093.8ab0",
               "selected-encapsulation": "arpa",
               "is-router": true,
               "serg-flags": 255,
               "reachability-state": "delay",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "dynamic"
           }
       },
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/0",
               "host-address": "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"
           },
           "timestamp": 1535997300701,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/0",
               "last-reached-time": {
                   "seconds": 0
               },
               "link-layer-address": "0000.0000.0000",
               "selected-encapsulation": "arpa",
               "is-router": false,
               "serg-flags": 255,
               "reachability-state": "reachable",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "static"
           }
       },
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/1",
               "host-address": "2020:2020::20"
           },
           "timestamp": 1535997300707,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/1",
               "last-reached-time": {
                   "seconds": 174
               },
               "link-layer-address": "5254.0093.8ab1",
               "selected-encapsulation": "arpa",
               "is-router": true,
               "serg-flags": 255,
               "reachability-state": "reachable",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "dynamic"
           }
       },
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/1",
               "host-address": "fe80::5054:ff:fe93:8ab1"
           },
           "timestamp": 1535997300707,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/1",
               "last-reached-time": {
                   "seconds": 169
               },
               "link-layer-address": "5254.0093.8ab1",
               "selected-encapsulation": "arpa",
               "is-router": true,
               "serg-flags": 255,
               "reachability-state": "reachable",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "dynamic"
           }
       },
       {
           "keys": {
               "node-name": "0/0/CPU0",
               "interface-name": "GigabitEthernet0/0/0/1",
               "host-address": "ffff:ffff:ffff:ffff:ffff:ffff:ffff:ffff"
           },
           "timestamp": 1535997300707,
           "content": {
               "vrfid": 1610612736,
               "interface-name": "Gi0/0/0/1",
               "last-reached-time": {
                   "seconds": 0
               },
               "link-layer-address": "0000.0000.0000",
               "selected-encapsulation": "arpa",
               "is-router": false,
               "serg-flags": 255,
               "reachability-state": "reachable",
               "location": "0/0/CPU0",
               "encapsulation": "arpa",
               "origin-encapsulation": "static"
           }
       }
   ],
   "collection_id": 6
}
>>>
>>>
>>> exit()
[r1:~]$
[r1:~]$ exit
logout

RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```


<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #eff9ef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Perfect! We're receiving all the fields we need as per the original show command. You're now ready to start building your own Telemetry collector.
</p>  



## Building your own clients

We learnt in the earlier sections that IOS-XR supports streaming Telemetry data over raw TCP (dial-out) and gRPC (dial-in and dial-out). The structure of the streamable data is derived from Oper YANG models.

Now, these Oper Yang-models can also be mapped to equivalent protobuf models, represented in `.proto` files.
This is especially useful when we need to write a Telemetry client code from scratch.
By exposing these protobuf-based capabilities over a gRPC connection, it enables a user to utilize gRPC's intrinsic architecture to generate bindings(code/libraries) in a language of choice (python, c++, golang etc.).

To view the .proto files corresponding to the Oper Yang models in XR, clone the following git repo on the `devbox` and peek into the `proto_archive/` directory:

><https://github.com/cisco/bigmuddy-network-telemetry-proto>  



This represents all the available Oper Yang models arranged as folders containing the .proto files.
We can generate bindings in the language of choice (python, c++, golang etc.) using these .proto files and leverage the bindings to subscribe to the router's Telemetry stream as well as decode the data received. We will delve into these scenarios and write our own Telemetry client/collector in subsequent labs in this module.





## Clean up!

Finally, clean up the telemetry configuration from router r1 as we progress to the next stage of the lab:


Remove existing telemetry configurations from `r1`:

```
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#conf t
Tue Sep  4 08:01:03.269 UTC
RP/0/RP0/CPU0:r1(config)#no telemetry model-driven
RP/0/RP0/CPU0:r1(config)#commit
Tue Sep  4 08:01:06.422 UTC
RP/0/RP0/CPU0:r1(config)#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```

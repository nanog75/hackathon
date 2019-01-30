---
published: true
date: '2018-06-12 09:51 -0400'
title: 'Step 2: Using ncclient  to interact over Netconf using YANG models'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - clus2018
  - devnet
excerpt: Using ncclient to work with IOS-XR netconf
---


{% include toc %}

## To Know More

ncclient is a python library available to ease interaction over Netconf with devices that support it. Utilize ncclient to setup netconf sessions with the device and send/receive YANG data encoded in XML.

To learn more about ncclient, head here:

><https://github.com/ncclient/ncclient>



>The vagrant environment consisting of two IOS-XRv9k instances must already be running. Verify >this by opening up a terminal and running the following commands:
>
>  ```
>  cd ~/topology
>  vagrant status
>
>  ```
> You should then see something like this:
>  ```
>  cisco@pod2:~/topology$ vagrant status
>  Current machine states:
>
>  rtr1                      running (virtualbox)
>  rtr2                      running (virtualbox)
>  devbox                    running (virtualbox)
>
>   This environment represents multiple VMs. The VMs are all listed
>   above with their current state. For more information about a specific
>   VM, run `vagrant status NAME`.
>   cisco@pod2:~/topology$ 
>  ```

{: .notice--info}


The topology we will be dealing with looks something like this:

![topology.png]({{site.baseurl}}/images/topology.png)


Ask the proctor for your credentials to access the pod. Click on the "VNC" option under Devnet_Pods/Podx where Podx is the pod assigned to you.
{: .notice--warning}


## Open up Sublime to view the code pieces we will be dealing with
![sublime.png]({{site.baseurl}}/images/sublime.png)

Keep this open to view the code at any point. Drop back into the terminal to continue.



## Execute ncclient code

Hop back over into the `vagrant/code` directory in the devbox and view the available scripts (in sublime or in bash):


```
vagrant@devbox:code$ cd ncclient/
vagrant@devbox:ncclient$ ls
configure-telemetry.py  dump_pyang_tree.py
vagrant@devbox:ncclient$ 

```


### Use pyang to dump YANG model

We execute this script to view the YANG model that we will be utilizing. This script interacts with the device using ncclient to fetch the YANG model and then dumps it in a tree format using pyang for easy viewing.


```
vagrant@devbox:ncclient$ 
vagrant@devbox:ncclient$ ./dump_pyang_tree.py 
module: openconfig-telemetry
   +--rw telemetry-system
      +--rw sensor-groups
      |  +--rw sensor-group* [sensor-group-id]
      |     +--rw sensor-group-id    -> ../config/sensor-group-id
      |     +--rw config
      |     |  +--rw sensor-group-id?   string
      |     +--ro state
      |     |  +--ro sensor-group-id?   string
      |     +--rw sensor-paths
      |        +--rw sensor-path* [path]
      |           +--rw path      -> ../config/path
      |           +--rw config
      |           |  +--rw path?             string
      |           |  +--rw exclude-filter?   string
      |           +--ro state
      |              +--ro path?             string
      |              +--ro exclude-filter?   string
      +--rw destination-groups
      |  +--rw destination-group* [group-id]
      |     +--rw group-id        -> ../config/group-id
      |     +--rw config
      |     |  +--rw group-id?   string
      |     +--ro state
      |     |  +--ro group-id?   string
      |     +--rw destinations
      |        +--rw destination* [destination-address destination-port]
      |           +--rw destination-address    -> ../config/destination-address
      |           +--rw destination-port       -> ../config/destination-port
      |           +--rw config
      |           |  +--rw destination-address?    inet:ip-address
      |           |  +--rw destination-port?       uint16
      |           |  +--rw destination-protocol?   telemetry-stream-protocol
      |           +--ro state
      |              +--ro destination-address?    inet:ip-address
      |              +--ro destination-port?       uint16
      |              +--ro destination-protocol?   telemetry-stream-protocol
      +--rw subscriptions
         +--rw persistent
         |  +--rw subscription* [subscription-id]
         |     +--rw subscription-id       -> ../config/subscription-id
         |     +--rw config
         |     |  +--rw subscription-id?          uint64
         |     |  +--rw local-source-address?     inet:ip-address
         |     |  +--rw originated-qos-marking?   inet:dscp
         |     +--ro state
         |     |  +--ro subscription-id?          uint64
         |     |  +--ro local-source-address?     inet:ip-address
         |     |  +--ro originated-qos-marking?   inet:dscp
         |     +--rw sensor-profiles
         |     |  +--rw sensor-profile* [sensor-group]
         |     |     +--rw sensor-group    -> ../config/sensor-group
         |     |     +--rw config
         |     |     |  +--rw sensor-group?         -> /telemetry-system/sensor-groups/sensor-group/config/sensor-group-id
         |     |     |  +--rw sample-interval?      uint64
         |     |     |  +--rw heartbeat-interval?   uint64
         |     |     |  +--rw suppress-redundant?   boolean
         |     |     +--ro state
         |     |        +--ro sensor-group?         -> /telemetry-system/sensor-groups/sensor-group/config/sensor-group-id
         |     |        +--ro sample-interval?      uint64
         |     |        +--ro heartbeat-interval?   uint64
         |     |        +--ro suppress-redundant?   boolean
         |     +--rw destination-groups
         |        +--rw destination-group* [group-id]
         |           +--rw group-id    -> ../config/group-id
         |           +--rw config
         |           |  +--rw group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
         |           +--rw state
         |              +--rw group-id?   -> ../../../../../../../destination-groups/destination-group/group-id
         +--rw dynamic
            +--ro subscription* [subscription-id]
               +--ro subscription-id    -> ../state/subscription-id
               +--ro state
               |  +--ro subscription-id?          uint64
               |  +--ro destination-address?      inet:ip-address
               |  +--ro destination-port?         uint16
               |  +--ro destination-protocol?     telemetry-stream-protocol
               |  +--ro sample-interval?          uint64
               |  +--ro heartbeat-interval?       uint64
               |  +--ro suppress-redundant?       boolean
               |  +--ro originated-qos-marking?   inet:dscp
               +--ro sensor-paths
                  +--ro sensor-path* [path]
                     +--ro path     -> ../state/path
                     +--ro state
                        +--ro path?             string
                        +--ro exclude-filter?   string

vagrant@devbox:ncclient$ 

```



## Use Openconfig Telemetry YANG model to configure Telemetry


### View the Model

The model we intend to use can be viewed in the configure_telemetry.py file before we execute it.

```
#!/usr/bin/env python

from ncclient import manager
import re
from subprocess import Popen, PIPE, STDOUT
    
xr = manager.connect(host='20.1.1.10', port=830, username='vagrant', password='vagrant',
	allow_agent=False,
	look_for_keys=False,
	hostkey_verify=False,
	unknown_host_cb=True)


edit_data = '''
<config>
  <telemetry-system xmlns="http://openconfig.net/yang/telemetry">
   <sensor-groups>
    <sensor-group>
     <sensor-group-id>IPV6Neighbor</sensor-group-id>
     <sensor-paths nc:operation="replace">
      <sensor-path>
       <path>Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address</path>
      </sensor-path>
     </sensor-paths>
    </sensor-group>
   </sensor-groups>
   <subscriptions>
    <persistent>
     <subscription>
      <subscription-id>IPV6</subscription-id>
      <config>
       <subscription-id>IPV6</subscription-id>
      </config>
      <sensor-profiles>
       <sensor-profile>
        <sensor-group>IPV6Neighbor</sensor-group>
        <config>
         <sensor-group>IPV6Neighbor</sensor-group>
         <sample-interval>10000</sample-interval>
        </config>
       </sensor-profile>
      </sensor-profiles>
     </subscription>
    </persistent>
   </subscriptions>
  </telemetry-system>
</config>
'''


xr.edit_config(edit_data, default_operation='replace', target='candidate', format='xml')
xr.commit()


```


### Execute the ncclient script

```
vagrant@devbox:ncclient$ 
vagrant@devbox:ncclient$ ./configure-telemetry.py 
vagrant@devbox:ncclient$ 

```


### ssh into rtr1 to see the resulting configuration


Use `vagrant port rtr1` in the Pod's shell to determine the port being utilized for rtr1's ssh port (22):

```


```

```
cisco@pod2:~/topology$ ssh -p 2223 vagrant@localhost
The authenticity of host '[localhost]:2223 ([127.0.0.1]:2223)' can't be established.
RSA key fingerprint is SHA256:IhWg/fJ3DmarPxUK+AvUcKJ9sgW4D2U4y+gqrtmtOpc.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[localhost]:2223' (RSA) to the list of known hosts.
Password: 


RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#   
RP/0/RP0/CPU0:rtr1#show  running-config  telemetry model-driven 
Tue Jun 12 04:47:56.149 UTC
telemetry model-driven
 sensor-group IPV6Neighbor
  sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
 !
 subscription IPV6
  sensor-group-id IPV6Neighbor sample-interval 10000
 !
!

RP/0/RP0/CPU0:rtr1#

```

### Check that the telemetry stream is ready to go


In the rtr1 CLI, restart the emsd process to help resolve the sensor-path we just configured earlier.

```

RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#process restart emsd
Tue Jun 12 04:53:03.547 UTC
RP/0/RP0/CPU0:rtr1#

```

Now check the telemetry subscription state to make sure the sensor-path is resolved.

```
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show telemetry model-driven subscription IPV6 
Tue Jun 12 04:54:48.409 UTC
Subscription:  IPV6
-------------
  State:       NA
  Sensor groups:
  Id: IPV6Neighbor
    Sample Interval:      10000 ms
    Sensor Path:          Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address
    Sensor Path State:    Resolved

  Collection Groups:
  ------------------
  No active collection groups

RP/0/RP0/CPU0:rtr1#
```

Excellent! We're good to go.
{: .notice--success}

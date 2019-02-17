---
published: true
date: '2019-02-17 02:55 -0800'
title: 'Step 2:  Playing with YDK for config and gNMI Telemetry'
author: Nanog75 Hackathon Team
excerpt: >-
  Understand how to build YDK code to configure BGP  and try out  the existing
  Telemetry client
tags:
  - nanog75
---


{% include base_path %}
{% include toc %} 


## Analyse the provided ydk code


In the previous step: [Step 1]({{ base_path }}), we cloned the `code-samples` repository onto dev1.

Keep yourself connected to dev1 and drop into the `code-samples/ydk` directory:


```
tesuto@dev1:~$ cd code-samples/ydk/
tesuto@dev1:~/code-samples/ydk$ ls
config_oc_bgp_ydk.py
tesuto@dev1:~/code-samples/ydk$ 
```

Dump the contents of the `config_oc_bgp_ydk.py` script:


```python
tesuto@dev1:~/code-samples/ydk$ cat config_oc_bgp_ydk.py 
#!/usr/bin/env python

import logging,os
from argparse import ArgumentParser
from ydk.gnmi.providers import gNMIServiceProvider
from ydk.path import Repository
from ydk.models.openconfig import openconfig_network_instance as oc_ni
from ydk.services import CRUDService
from ydk.models.openconfig import openconfig_policy_types as oc_policy_types
from ydk.models.openconfig import openconfig_bgp_types as oc_bgp_types



def config_oc_bgp_ipv4():

    ni = oc_ni.NetworkInstances.NetworkInstance()
    ni.name = "default"
    
    protocol = ni.protocols.Protocol()

    protocol.identifier =  oc_policy_types.BGP()
    protocol.name = "default"
    protocol.config.identifier = oc_policy_types.BGP()
    protocol.config.name = "default"

    protocol.bgp.global_.config.as_ = 65000

    ni.protocols.protocol.append(protocol)

    return ni


if __name__ == "__main__":
    """Execute main program."""
    parser = ArgumentParser()
    parser.add_argument("-v", "--verbose", help="print debugging messages",
                        action="store_true")
    parser.add_argument('-i', '--grpc-ip', action='store', dest='grpc_ip',
                    help='Specify the IOS-XR GRPC server IP address', required=True)
    parser.add_argument('-g', '--grpc-port', action='store', dest='grpc_port',
                    help='Specify the IOS-XR GRPC server port', required=True)
    parser.add_argument('-u', '--username', action='store', dest='username',
                    help='Specify username to connect to gRPC server on XR', required=True)
    parser.add_argument('-p', '--password', action='store', dest='password',
                    help='Specify password to connect to gRPC server on XR', required=True)
    parser.add_argument('-y', '--yang-repo-path', action='store', dest='yang_repo_location',
                    help='Specify yang repo path', required=True)
    parser.add_argument('-e', '--extra-verbose', action='store_true',
                    help='Extra Verbose logs')
    args = parser.parse_args()

    # log debug messages if verbose argument specified
    if args.verbose:
        logger = logging.getLogger("ydk")
        logger.setLevel(logging.INFO)
        handler = logging.StreamHandler()
        formatter = logging.Formatter(("%(asctime)s - %(name)s - "
                                      "%(levelname)s - %(message)s"))
        handler.setFormatter(formatter)
        logger.addHandler(handler)
        if args.extra_verbose:
            os.environ['GRPC_TRACE'] = 'all'
            os.environ['GRPC_VERBOSITY'] = 'DEBUG'

    repository = Repository(args.yang_repo_location)
    provider = gNMIServiceProvider(repo=repository, 
                                   address=args.grpc_ip,
                                   port=int(args.grpc_port), 
                                   username=args.username, 
                                   password=args.password)

    crud = CRUDService()

    bgp_ni = config_oc_bgp_ipv4()

    crud.create(provider, bgp_ni)
tesuto@dev1:~/code-samples/ydk$ 
tesuto@dev1:~/code-samples/ydk$ 
```



Notice the basic structure of the YDK script:

1. It has 3 major components:  Provider, CRUDService and the Model objects.

2. The provider object is associated with the type of protocol and RPC you want to utilize for the connection with the router. In today's integration, we intend to use gNMI, hence the gNMIServiceProvider. gNMI does not implement the fetch capabilities automatically unlike netconf, therefore for validation purposes (which YDK does by default), you need to specify the location of a local directory containing the relevant yang models.

3.  The CRUDService helps implement a common CRUD (Create, Read, Update, Delete) API for all providers (today Netconf and gNMI are supported).

4. The Model objects are the highlight of YDK. It translates Yang models into objects in different programming languages (here we use python objects) making it very intuitive to use yang models if you know a bit of python.



## Execute the YDK script

Provide the relevant options to the YDK script (gRPC on all the routers will be enabled on port 57777 which we use for gNMI):


```
tesuto@dev1:~/code-samples/ydk$ 
tesuto@dev1:~/code-samples/ydk$ python config_oc_bgp_ydk.py -h
usage: config_oc_bgp_ydk.py [-h] [-v] -i GRPC_IP -g GRPC_PORT -u USERNAME -p
                            PASSWORD -y YANG_REPO_LOCATION [-e]

optional arguments:
  -h, --help            show this help message and exit
  -v, --verbose         print debugging messages
  -i GRPC_IP, --grpc-ip GRPC_IP
                        Specify the IOS-XR GRPC server IP address
  -g GRPC_PORT, --grpc-port GRPC_PORT
                        Specify the IOS-XR GRPC server port
  -u USERNAME, --username USERNAME
                        Specify username to connect to gRPC server on XR
  -p PASSWORD, --password PASSWORD
                        Specify password to connect to gRPC server on XR
  -y YANG_REPO_LOCATION, --yang-repo-path YANG_REPO_LOCATION
                        Specify yang repo path
  -e, --extra-verbose   Extra Verbose logs
tesuto@dev1:~/code-samples/ydk$ 
```

We need to run this script against rtr1 (IP address `100.96.0.14`):


```
tesuto@dev1:~/code-samples/ydk$ 
tesuto@dev1:~/code-samples/ydk$ python config_oc_bgp_ydk.py -i 100.96.0.14 -g 57777 -u rtrdev -p nanog75sf -y /home/tesuto/yang -v
2019-02-17 11:12:08,173 - ydk - INFO - gNMIServiceProvider Connected to 100.96.0.14 via Insecure Channel
2019-02-17 11:12:08,214 - ydk - INFO - Executing CRUD create operation on [network-instance[name='default']]
2019-02-17 11:12:08,214 - ydk - INFO - Executing set gRPC operation 'update' on entity 'network-instance[name='default']'
2019-02-17 11:12:08,227 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,236 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,242 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,248 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,254 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,261 - ydk - ERROR - Cannot find model with module name 'openconfig-bgp'
2019-02-17 11:12:08,284 - ydk - ERROR - Cannot find model with module name 'openconfig-mpls'
2019-02-17 11:12:08,293 - ydk - ERROR - Cannot find model with module name 'openconfig-mpls'
2019-02-17 11:12:08,298 - ydk - ERROR - Cannot find model with module name 'openconfig-mpls'
2019-02-17 11:12:08,317 - ydk - ERROR - Cannot find model with module name 'openconfig-isis'
2019-02-17 11:12:08,331 - ydk - ERROR - Cannot find model with module name 'openconfig-isis'
2019-02-17 11:12:08,357 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,362 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,369 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,375 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,380 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,387 - ydk - ERROR - Cannot find model with module name 'openconfig-aft'
2019-02-17 11:12:08,393 - ydk - ERROR - Cannot find model with module name 'openconfig-network-instance'
2019-02-17 11:12:08,442 - ydk - INFO - 
=============== Set Request Sent ================
update {
  path {
    origin: "openconfig-network-instance"
    elem {
      name: "network-instances"
    }
  }
  val {
    json_ietf_val: "{\"openconfig-network-instance:network-instance\":[{\"name\":\"default\",\"protocols\":{\"protocol\":[{\"identifier\":\"openconfig-policy-types:BGP\",\"name\":\"default\",\"config\":{\"identifier\":\"openconfig-policy-types:BGP\",\"name\":\"default\"},\"bgp\":{\"global\":{\"config\":{\"as\":65000}}}}]}}]}"
  }
}


2019-02-17 11:12:09,076 - ydk - INFO - 
============= Set Response Received =============
response {
  path {
    origin: "openconfig-network-instance"
    elem {
      name: "network-instances"
    }
  }
  message {
  }
  op: UPDATE
}
message {
}
timestamp: 1550401927704781395


2019-02-17 11:12:09,076 - ydk - INFO - Set Operation Succeeded
2019-02-17 11:12:09,076 - ydk - INFO - Operation succeeded
2019-02-17 11:12:09,077 - ydk - INFO - Disconnected from device
tesuto@dev1:~/code-samples/ydk$ 


```


Here we notice how YDK converts the CRUDService call into a request over gRPC to the gNMI server running on the router.

Now connect to router `rtr1` over SSH to see the effect of this YDK script:

**Username**: rtrdev
**Password**: nanog75sf
{: .notice--info}

```
tesuto@dev1:~/code-samples/ydk$ ssh rtrdev@100.96.0.14
Password: 


RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show  running-config  router bgp
Sun Feb 17 11:15:35.385 UTC
router bgp 65000
!

RP/0/RP0/CPU0:rtr1#


```


Perfect! Notice that the YDK script is using the openconfig-network-instance Yang model. These models are available on dev1 at `/home/tesuto/yang`.

Let's use the pyang tool to get a tree output of this model:


```
tesuto@dev1:~$ cd /home/tesuto/yang/
tesuto@dev1:~/yang$ 
tesuto@dev1:~/yang$ pyang -f tree openconfig-network-instance.yang 
module: openconfig-network-instance
  +--rw network-instances
     +--rw network-instance* [name]
        +--rw name                       -> ../config/name
        +--rw fdb
        |  +--rw config
        |  |  +--rw mac-learning?      boolean
        |  |  +--rw mac-aging-time?    uint16
        |  |  +--rw maximum-entries?   uint16
        |  +--ro state
        |  |  +--ro mac-learning?      boolean
        |  |  +--ro mac-aging-time?    uint16
        |  |  +--ro maximum-entries?   uint16
        |  +--rw mac-table
        |     +--rw entries
        |        +--rw entry* [mac-address]
        |           +--rw mac-address    -> ../config/mac-address
        |           +--rw config
        |           |  +--rw mac-address?   yang:mac-address
        |           |  +--rw vlan?          -> ../../../../../../vlans/vlan/config/vlan-id





#########################  Output Snipped ##################################




              +--rw bgp
              |  +--rw global
              |  |  +--rw config
              |  |  |  +--rw as           oc-inet:as-number
              |  |  |  +--rw router-id?   oc-yang:dotted-quad
              |  |  +--ro state
              |  |  |  +--ro as                oc-inet:as-number
              |  |  |  +--ro router-id?        oc-yang:dotted-quad
              |  |  |  +--ro total-paths?      uint32
              |  |  |  +--ro total-prefixes?   uint32
              |  |  +--rw default-route-distance
              |  |  |  +--rw config
              |  |  |  |  +--rw external-route-distance?   uint8
              |  |  |  |  +--rw internal-route-distance?   uint8
              |  |  |  +--ro state
              |  |  |     +--ro external-route-distance?   uint8
              |  |  |     +--ro internal-route-distance?   uint8
              |  |  +--rw confederation
              |  |  |  +--rw config
              |  |  |  |  +--rw enabled?      boolean
              |  |  |  |  +--rw identifier?   oc-inet:as-number
              |  |  |  |  +--rw member-as*    oc-inet:as-number
              |  |  |  +--ro state
              |  |  |     +--ro enabled?      boolean
              |  |  |     +--ro identifier?   oc-inet:as-number
              |  |  |     +--ro member-as*    oc-inet:as-number
              |  |  +--rw graceful-restart
              |  |  |  +--rw config
              |  |  |  |  +--rw enabled?             boolean
              |  |  |  |  +--rw restart-time?        uint16
              |  |  |  |  +--rw stale-routes-time?   decimal64
              |  |  |  |  +--rw helper-only?         boolean
              |  |  |  +--ro state
              |  |  |     +--ro enabled?             boolean
              |  |  |     +--ro restart-time?        uint16
              |  |  |     +--ro stale-routes-time?   decimal64
              |  |  |     +--ro helper-only?         boolean
              |  |  +--rw use-multiple-paths
              |  |  |  +--rw config
              |  |  |  |  +--rw enabled?   boolean
              |  |  |  +--ro state
              |  |  |  |  +--ro enabled?   boolean
              |  |  |  +--rw ebgp






#########################  Output Snipped ##################################





                       |     |        |     |  +--rw sid-id?          sr-sid-type
                       |     |        |     |  +--rw label-options?   enumeration
                       |     |        |     +--ro state
                       |     |        |        +--ro prefix?          inet:ip-prefix
                       |     |        |        +--ro sid-id?          sr-sid-type
                       |     |        |        +--ro label-options?   enumeration
                       |     |        +--rw adjacency-sids
                       |     |           +--rw adjacency-sid* [neighbor sid-id]
                       |     |              +--rw sid-id      -> ../config/sid-id
                       |     |              +--rw neighbor    -> ../config/neighbor
                       |     |              +--rw config
                       |     |              |  +--rw sid-id?                union
                       |     |              |  +--rw protection-eligible?   boolean
                       |     |              |  +--rw group?                 boolean
                       |     |              |  +--rw neighbor?              inet:ip-address
                       |     |              +--ro state
                       |     |                 +--ro sid-id?                    union
                       |     |                 +--ro protection-eligible?       boolean
                       |     |                 +--ro group?                     boolean
                       |     |                 +--ro neighbor?                  inet:ip-address
                       |     |                 +--ro allocated-dynamic-local?   sr-sid-type
                       |     +--rw hello-authentication
                       |        +--rw config
                       |        |  +--rw hello-authentication?   boolean
                       |        +--ro state
                       |        |  +--ro hello-authentication?   boolean
                       |        +--rw key
                       |        |  +--rw config
                       |        |  |  +--rw auth-password?   oc-types:routing-password
                       |        |  +--ro state
                       |        |     +--ro auth-password?   oc-types:routing-password
                       |        +--rw keychain
                       +--rw timers
                       |  +--rw config
                       |  |  +--rw csnp-interval?         uint16
                       |  |  +--rw lsp-pacing-interval?   uint64
                       |  +--ro state
                       |     +--ro csnp-interval?         uint16
                       |     +--ro lsp-pacing-interval?   uint64
                       +--rw bfd
                       |  +--rw config
                       |  |  +--rw bfd-tlv?   boolean
                       |  +--ro state
                       |     +--ro bfd-tlv?   boolean
                       +--rw interface-ref
                          +--rw config
                          |  +--rw interface?      -> /oc-if:interfaces/interface/name
                          |  +--rw subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
                          +--ro state
                             +--ro interface?      -> /oc-if:interfaces/interface/name
                             +--ro subinterface?   -> /oc-if:interfaces/interface[oc-if:name=current()/../interface]/subinterfaces/subinterface/index
tesuto@dev1:~/yang$ 


```


Alright, this is a huge model and we are only interested in the BGP portion of the model.
Since this model includes the openconfig-bgp.yang model, let's dump the openconfig-bgp yang model instead. Notice the highlighted entries below. That is the extent to which the given YDK script traverses the model and creates the object for the gNMI CRUDService call.


<div class="highlighter-rouge">
<pre class="highlight">
<code>
tesuto@dev1:~/yang$ 
tesuto@dev1:~/yang$ pyang -f tree openconfig-bgp.yang 
module: openconfig-bgp
  <mark>+--rw bgp
     +--rw global
     |  +--rw config
     |  |  +--rw as           oc-inet:as-number</mark>
     |  |  +--rw router-id?   oc-yang:dotted-quad
     |  +--ro state
     |  |  +--ro as                oc-inet:as-number
     |  |  +--ro router-id?        oc-yang:dotted-quad
     |  |  +--ro total-paths?      uint32
     |  |  +--ro total-prefixes?   uint32
     |  +--rw default-route-distance
     |  |  +--rw config
     |  |  |  +--rw external-route-distance?   uint8
     |  |  |  +--rw internal-route-distance?   uint8
     |  |  +--ro state
     |  |     +--ro external-route-distance?   uint8
     |  |     +--ro internal-route-distance?   uint8
     |  +--rw confederation
     |  |  +--rw config
     |  |  |  +--rw enabled?      boolean
     |  |  |  +--rw identifier?   oc-inet:as-number
     |  |  |  +--rw member-as*    oc-inet:as-number
     |  |  +--ro state
     |  |     +--ro enabled?      boolean
     |  |     +--ro identifier?   oc-inet:as-number
     |  |     +--ro member-as*    oc-inet:as-number
     |  +--rw graceful-restart
     |  |  +--rw config
     |  |  |  +--rw enabled?             boolean
     |  |  |  +--rw restart-time?        uint16
     |  |  |  +--rw stale-routes-time?   decimal64
     |  |  |  +--rw helper-only?         boolean
     |  |  +--ro state
     |  |     +--ro enabled?             boolean
     |  |     +--ro restart-time?        uint16
     |  |     +--ro stale-routes-time?   decimal64
     |  |     +--ro helper-only?         boolean
     |  +--rw use-multiple-paths
     |  |  +--rw config
     |  |  |  +--rw enabled?   boolean
     |  |  +--ro state
     |  |  |  +--ro enabled?   boolean
     |  |  +--rw ebgp
     |  |  |  +--rw config
     |  |  |  |  +--rw allow-multiple-as?   boolean
     |  |  |  |  +--rw maximum-paths?       uint32
     |  |  |  +--ro state
     |  |  |     +--ro allow-multiple-as?   boolean
     |  |  |     +--ro maximum-paths?       uint32
     |  |  +--rw ibgp
     |  |     +--rw config
     |  |     |  +--rw maximum-paths?   uint32
     |  |     +--ro state
     |  |        +--ro maximum-paths?   uint32
     |  +--rw route-selection-options
     |  |  +--rw config
     |  |  |  +--rw always-compare-med?           boolean
     |  |  |  +--rw ignore-as-path-length?        boolean
     |  |  |  +--rw external-compare-router-id?   boolean
     |  |  |  +--rw advertise-inactive-routes?    boolean
     |  |  |  +--rw enable-aigp?                  boolean
     |  |  |  +--rw ignore-next-hop-igp-metric?   boolean
     |  |  +--ro state
     |  |     +--ro always-compare-med?           boolean
     |  |     +--ro ignore-as-path-length?        boolean
     |  |     +--ro external-compare-router-id?   boolean
     |  |     +--ro advertise-inactive-routes?    boolean
     |  |     +--ro enable-aigp?                  boolean
     |  |     +--ro ignore-next-hop-igp-metric?   boolean
     |  +--rw afi-safis
     |  |  +--rw afi-safi* [afi-safi-name]
     |  |     +--rw afi-safi-name              -> ../config/afi-safi-name
     |  |     +--rw config
     |  |     |  +--rw afi-safi-name?   identityref
     |  |     |  +--rw enabled?         boolean
     |  |     +--ro state
     |  |     |  +--ro afi-safi-name?    identityref
     |  |     |  +--ro enabled?          boolean
     |  |     |  +--ro total-paths?      uint32
     |  |     |  +--ro total-prefixes?   uint32
     |  |     +--rw graceful-restart
     |  |     |  +--rw config
     |  |     |  |  +--rw enabled?   boolean
     |  |     |  +--ro state
     |  |     |     +--ro enabled?   boolean
     |  |     +--rw route-selection-options
     |  |     |  +--rw config
     |  |     |  |  +--rw always-compare-med?           boolean
     |  |     |  |  +--rw ignore-as-path-length?        boolean
     |  |     |  |  +--rw external-compare-router-id?   boolean
     |  |     |  |  +--rw advertise-inactive-routes?    boolean
     |  |     |  |  +--rw enable-aigp?                  boolean
     |  |     |  |  +--rw ignore-next-hop-igp-metric?   boolean
     |  |     |  +--ro state
     |  |     |     +--ro always-compare-med?           boolean
     |  |     |     +--ro ignore-as-path-length?        boolean
     |  |     |     +--ro external-compare-router-id?   boolean
     |  |     |     +--ro advertise-inactive-routes?    boolean
     |  |     |     +--ro enable-aigp?                  boolean
     |  |     |     +--ro ignore-next-hop-igp-metric?   boolean
     |  |     +--rw use-multiple-paths
     |  |     |  +--rw config
     |  |     |  |  +--rw enabled?   boolean
     |  |     |  +--ro state
     |  |     |  |  +--ro enabled?   boolean
     |  |     |  +--rw ebgp
     |  |     |  |  +--rw config
     |  |     |  |  |  +--rw allow-multiple-as?   boolean
     |  |     |  |  |  +--rw maximum-paths?       uint32
     |  |     |  |  +--ro state
     |  |     |  |     +--ro allow-multiple-as?   boolean
     |  |     |  |     +--ro maximum-paths?       uint32
     |  |     |  +--rw ibgp
     |  |     |     +--rw config
     |  |     |     |  +--rw maximum-paths?   uint32
     |  |     |     +--ro state
     |  |     |        +--ro maximum-paths?   uint32
     |  |     +--rw ipv4-unicast
     |  |     |  +--rw prefix-limit
     |  |     |  |  +--rw config
     |  |     |  |  |  +--rw max-prefixes?             uint32
     |  |     |  |  |  +--rw prevent-teardown?         boolean
     |  |     |  |  |  +--rw shutdown-threshold-pct?   oc-types:percentage
     |  |     |  |  |  +--rw restart-timer?            decimal64
     |  |     |  |  +--ro state
     |  |     |  |     +--ro max-prefixes?             uint32
     |  |     |  |     +--ro prevent-teardown?         boolean
     |  |     |  |     +--ro shutdown-threshold-pct?   oc-types:percentage
     |  |     |  |     +--ro restart-timer?            decimal64
     |  |     |  +--rw config
    
    
    #########################  Output Snipped ##################################
    
    
    
     |        +--ro state
     |           +--ro prefix?       oc-inet:ip-prefix
     |           +--ro peer-group?   -> ../../../../../peer-groups/peer-group/config/peer-group-name
     +--rw neighbors
     |  +--rw neighbor* [neighbor-address]
     |     +--rw neighbor-address      -> ../config/neighbor-address
     |     +--rw config
     |     |  +--rw peer-group?           -> ../../../../peer-groups/peer-group/peer-group-name
     |     |  +--rw neighbor-address?     oc-inet:ip-address
     |     |  +--rw enabled?              boolean
     |     |  +--rw peer-as?              oc-inet:as-number
     |     |  +--rw local-as?             oc-inet:as-number
     |     |  +--rw peer-type?            oc-bgp-types:peer-type
     |     |  +--rw auth-password?        oc-types:routing-password
     |     |  +--rw remove-private-as?    uint32
     |     |  +--rw route-flap-damping?   boolean
     |     |  +--rw send-community?       oc-bgp-types:community-type
     |     |  +--rw description?          string
     |     +--ro state
     |     |  +--ro peer-group?                -> ../../../../peer-groups/peer-group/peer-group-name
     |     |  +--ro neighbor-address?          oc-inet:ip-address
     |     |  +--ro enabled?                   boolean
     |     |  +--ro peer-as?                   oc-inet:as-number
     |     |  +--ro local-as?                  oc-inet:as-number
     |     |  +--ro peer-type?                 oc-bgp-types:peer-type
     |     |  +--ro auth-password?             oc-types:routing-password
     |     |  +--ro remove-private-as?         uint32
     |     |  +--ro route-flap-damping?        boolean
     |     |  +--ro send-community?            oc-bgp-types:community-type
     |     |  +--ro description?               string
     |     |  +--ro session-state?             enumeration
     |     |  +--ro last-established?          oc-types:timeticks64
     |     |  +--ro established-transitions?   oc-yang:counter64
     |     |  +--ro supported-capabilities*    identityref
     |     |  +--ro messages
     |     |  |  +--ro sent
     |     |  |  |  +--ro UPDATE?         uint64
     |     |  |  |  +--ro NOTIFICATION?   uint64
     |     |  |  +--ro received
     |     |  |     +--ro UPDATE?         uint64
     |     |  |     +--ro NOTIFICATION?   uint64
     |     |  +--ro queues
     |     |  |  +--ro input?    uint32
     |     |  |  +--ro output?   uint32
     |     |  +--ro dynamically-configured?    boolean
     |     +--rw timers
     |     |  +--rw config
     |     |  |  +--rw connect-retry?                    decimal64
     |     |  |  +--rw hold-time?                        decimal64
     |     |  |  +--rw keepalive-interval?               decimal64
     |     |  |  +--rw minimum-advertisement-interval?   decimal64
     |     |  +--ro state
     |     |     +--ro connect-retry?                    decimal64
     |     |     +--ro hold-time?                        decimal64
     |     |     +--ro keepalive-interval?               decimal64
     |     |     +--ro minimum-advertisement-interval?   decimal64
     |     |     +--ro negotiated-hold-time?             decimal64
     |     +--rw transport
  
  
   #########################  Output Snipped ##################################
  
  
                 +--rw l2vpn-vpls
                 |  +--rw prefix-limit
                 |     +--rw config
                 |     |  +--rw max-prefixes?             uint32
                 |     |  +--rw prevent-teardown?         boolean
                 |     |  +--rw shutdown-threshold-pct?   oc-types:percentage
                 |     |  +--rw restart-timer?            decimal64
                 |     +--ro state
                 |        +--ro max-prefixes?             uint32
                 |        +--ro prevent-teardown?         boolean
                 |        +--ro shutdown-threshold-pct?   oc-types:percentage
                 |        +--ro restart-timer?            decimal64
                 +--rw l2vpn-evpn
                    +--rw prefix-limit
                       +--rw config
                       |  +--rw max-prefixes?             uint32
                       |  +--rw prevent-teardown?         boolean
                       |  +--rw shutdown-threshold-pct?   oc-types:percentage
                       |  +--rw restart-timer?            decimal64
                       +--ro state
                          +--ro max-prefixes?             uint32
                          +--ro prevent-teardown?         boolean
                          +--ro shutdown-threshold-pct?   oc-types:percentage
                          +--ro restart-timer?            decimal64
tesuto@dev1:~/yang$ 

</code>
</pre>
</div>




## Your Tasks


Your task is to get the following configuration on rtr1 and rtr2 using YDK.

**rtr1 BGP configuration**:

```
router bgp 65000
 bgp router-id 172.16.1.1
 address-family ipv4 unicast
 !
 neighbor-group IBGP
  remote-as 65000
  local address 172.16.1.1
  address-family ipv4 unicast
  !
 !
 neighbor 172.16.4.1
  use neighbor-group IBGP
  address-family ipv4 unicast
  !
 !
!



```

**rtr2 BGP configuration**:

```
router bgp 65000
 bgp router-id 172.16.4.1
 address-family ipv4 unicast
 !
 neighbor-group IBGP
  remote-as 65000
  local address 172.16.4.1
  address-family ipv4 unicast
  !
 !
 neighbor 172.16.1.1
  remote-as 65000
  use neighbor-group IBGP
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!

```


### Task 1:  Use Ansible with YDK


Since you are going to spend time to figure out the YDK code to create the above configuration, you might as well do it as a module of Ansible!

This is already set up for you.

Browse into the `code-samples/ansible/playbooks/config_bgp/` directory. the structure is shown below.

There are two options available;

1. **The Easy route**:  Use the pre-existing `config_xr_bgp_netconf.yml` playbook with the Ansible netconf_config module to utilize the native IOS-XR BGP yang model and configure the two routers.
This playbook utilizes `rtr1-bgp.xml` and `rtr2-bgp.xml` files shown below.  



```
tesuto@dev1:~/code-samples/ansible/playbooks/config_bgp$ 
tesuto@dev1:~/code-samples/ansible/playbooks/config_bgp$ tree -I yang .
.
├── config_oc_bgp_ydk.yml
├── config_xr_bgp_netconf.yml
├── library
│   └── config_bgp_oc_ydk.py
└── xml
    ├── rtr1-bgp.xml
    └── rtr4-bgp.xml

2 directories, 5 files
tesuto@dev1:~/code-samples/ansible/playbooks/config_bgp$ 

```

2. **The Fun route**:  You've seen how the YDK script above is written. And also have seen the openconfig model to utilize to create the object to configure BGP on the two routers. There is a library file for ansible that is created already for you. Look at `~/code-samples/ansible/playbooks/config_bgp/library/config_bgp_oc_ydk.py`:

```
tesuto@dev1:~/code-samples/ansible/playbooks/config_bgp$ cat library/config_bgp_oc_ydk.py 
#!/usr/bin/env python3
#
# Copyright 2019 Cisco Systems, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ANSIBLE_METADATA = {
    'metadata_version': '1.0',
    'supported_by': 'community',
    'status': ['preview']
}

DOCUMENTATION = """
---
module: configure_bgp_oc_ydk
requirements:
    - ydk 0.8.1 (python)
    - ydk-models-cisco-ios-xr 6.5.1 (python)
"""

EXAMPLES = """
name: configure BGP using YDK with GNMI
      config_bgp_oc_ydk:
        yang_repo_location: '/home/userx/yang'
        host: "10.1.1.20"
        grpc_port: "57777"
        username: "rtruser"
        password: "rtrcreds"
        crud_op: "add"
        bgp_params: "{{ bgp_parameters }}"

where bgp_params-->

bgp_parameters: {
                       'vrf': 'default',
                       'as': "65000",
                       'router_id': "172.16.1.1",
                       'peer-group-name': "IBGP",
                       'peer-as': "65000",
                       'peer-group-local-address': "172.16.1.1",
                       'neighbor': "172.16.4.1"
                      }


"""

RETURN = """
  True or False based on the operation result from ydk
"""



from ansible.module_utils.basic import AnsibleModule

from ydk.gnmi.providers import gNMIServiceProvider
from ydk.gnmi.services import gNMIService
from ydk.path import Repository
from ydk.models.openconfig import openconfig_network_instance as oc_ni
from ydk.services import CRUDService
from ydk.models.openconfig import openconfig_policy_types as oc_policy_types
from ydk.models.openconfig import openconfig_bgp_types as oc_bgp_types
import sys

def config_bgp_ipv4(yang_repo="",
                    address="",
                    grpc_port="",
                    username="",
                    password="",
                    crud_op="add",
                    bgp={}):

     repository = Repository(yang_repo)
     provider = gNMIServiceProvider(repo=repository, 
                                    address=address, 
                                    port=int(grpc_port), 
                                    username=username, 
                                    password=password)

     gnmi_service = gNMIService()
     crud = CRUDService()

     ni = oc_ni.NetworkInstances.NetworkInstance()

     if "vrf" in list(bgp.keys()):
       ni.name = bgp["vrf"]
     else:
         print("Vrf for network Instance not specified")
         sys.exit(1)

     protocol = ni.protocols.Protocol()

     protocol.identifier =  oc_policy_types.BGP()
     protocol.name = "default"
     protocol.config.identifier = oc_policy_types.BGP()
     protocol.config.name = "default"

     if "as" in list(bgp.keys()):
         protocol.bgp.global_.config.as_ = int(bgp["as"])
     else:
         print("AS for BGP instance not specified")
         sys.exit(1)

     # Fill out your BGP object properly using the Openconfig Yang Model
     # You will need to bring up the IBGP neighbor between rtr1 and rtr4

     ni.protocols.protocol.append(protocol)


     if crud_op == "add":
         response = crud.create(provider, ni)
     elif crud_op == "delete":
         response = crud.delete(provider, ni)
     elif crud_op is "update":
         response = crud.update(provider, ni)
     else:
         print("Invalid operation requested, allowed values =  add, update, delete")
         return False
     return response


def main():
    """Ansible module to configure BGP using the Openconfig Network Instances model"""
    module = AnsibleModule(
        argument_spec=dict(
            yang_repo_location=dict(type='str', required=True),
            host=dict(type='str', required=True),
            grpc_port=dict(type='str', required=True),
            username=dict(type='str', required=True),
            password=dict(type='str', required=True, no_log=True),
            crud_op=dict(type='str', required=True),
            bgp_params=dict(type='dict',required=True)
        ),
        supports_check_mode=True
    )

    if module.check_mode:
        module.exit_json(changed=False)

    try:
        retvals = config_bgp_ipv4(module.params['yang_repo_location'],
                                  module.params['host'],
                                  module.params['grpc_port'],
                                  module.params['username'],
                                  module.params['password'],
                                  module.params['crud_op'],
                                  module.params['bgp_params'])
    except Exception as exc:
        module.fail_json(msg='Failed to configure BGP ({})'.format(exc))

    if (retvals):
        module.exit_json(changed=True)
    else: 
        module.exit_json(changed=False)

if __name__ == '__main__':
    """Execute main program."""
    main()
# End of module
tesuto@dev1:~/code-samples/ansible/playbooks/config_bgp$ 



```


You can execute this ansible-playbook like so (verbose output):

```
tesuto@dev1:~/code-samples/ansible$ pwd
/home/tesuto/code-samples/ansible
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ ansible-playbook -i ansible_hosts playbooks/config_bgp/config_
config_oc_bgp_ydk.yml      config_xr_bgp_netconf.yml  
tesuto@dev1:~/code-samples/ansible$ ansible-playbook -i ansible_hosts playbooks/config_bgp/config_oc_bgp_ydk.yml -vvv
ansible-playbook 2.6.0
  config file = None
  configured module search path = [u'/home/tesuto/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/local/lib/python2.7/dist-packages/ansible
  executable location = /usr/local/bin/ansible-playbook
  python version = 2.7.15rc1 (default, Nov 12 2018, 14:31:15) [GCC 7.3.0]
No config file found; using defaults
Parsed /home/tesuto/code-samples/ansible/ansible_hosts inventory source with ini plugin

PLAYBOOK: config_oc_bgp_ydk.yml ***********************************************************************************************************************
1 plays in playbooks/config_bgp/config_oc_bgp_ydk.yml

PLAY [Configure iBGP on routers using OC NetworkInstance Model with YDK] ******************************************************************************
META: ran handlers

TASK [configure BGP using YDK with GNMI] **************************************************************************************************************
task path: /home/tesuto/code-samples/ansible/playbooks/config_bgp/config_oc_bgp_ydk.yml:19
<100.96.0.14> ESTABLISH LOCAL CONNECTION FOR USER: tesuto
<100.96.0.14> EXEC /bin/sh -c 'echo ~tesuto && sleep 0'
<100.96.0.14> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590 `" && echo ansible-tmp-1550404266.7-92535267491590="` echo /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590 `" ) && sleep 0'
<100.96.0.26> ESTABLISH LOCAL CONNECTION FOR USER: tesuto
<100.96.0.26> EXEC /bin/sh -c 'echo ~tesuto && sleep 0'
<100.96.0.26> EXEC /bin/sh -c '( umask 77 && mkdir -p "` echo /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133 `" && echo ansible-tmp-1550404266.72-154672175170133="` echo /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133 `" ) && sleep 0'
Using module file /home/tesuto/code-samples/ansible/playbooks/config_bgp/library/config_bgp_oc_ydk.py
<100.96.0.14> PUT /home/tesuto/.ansible/tmp/ansible-local-8551Zl40bD/tmpCgqwow TO /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590/config_bgp_oc_ydk.py
Using module file /home/tesuto/code-samples/ansible/playbooks/config_bgp/library/config_bgp_oc_ydk.py
<100.96.0.14> EXEC /bin/sh -c 'chmod u+x /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590/ /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590/config_bgp_oc_ydk.py && sleep 0'
<100.96.0.26> PUT /home/tesuto/.ansible/tmp/ansible-local-8551Zl40bD/tmptvOvzF TO /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133/config_bgp_oc_ydk.py
<100.96.0.26> EXEC /bin/sh -c 'chmod u+x /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133/ /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133/config_bgp_oc_ydk.py && sleep 0'
<100.96.0.14> EXEC /bin/sh -c '/usr/bin/python /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590/config_bgp_oc_ydk.py && sleep 0'
<100.96.0.26> EXEC /bin/sh -c '/usr/bin/python /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133/config_bgp_oc_ydk.py && sleep 0'
<100.96.0.14> EXEC /bin/sh -c 'rm -f -r /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.7-92535267491590/ > /dev/null 2>&1 && sleep 0'
changed: [rtr1] => {
    "changed": true, 
    "invocation": {
        "module_args": {
            "bgp_params": {
                "as": 65000, 
                "neighbor": "172.16.4.1", 
                "peer-as": 65000, 
                "peer-group-local-address": "172.16.1.1", 
                "peer-group-name": "IBGP", 
                "router_id": "172.16.1.1", 
                "vrf": "default"
            }, 
            "crud_op": "add", 
            "grpc_port": "57777", 
            "host": "100.96.0.14", 
            "password": "VALUE_SPECIFIED_IN_NO_LOG_PARAMETER", 
            "username": "rtrdev", 
            "yang_repo_location": "./yang"
        }
    }
}
<100.96.0.26> EXEC /bin/sh -c 'rm -f -r /home/tesuto/.ansible/tmp/ansible-tmp-1550404266.72-154672175170133/ > /dev/null 2>&1 && sleep 0'
changed: [rtr4] => {
    "changed": true, 
    "invocation": {
        "module_args": {
            "bgp_params": {
                "as": 65000, 
                "neighbor": "172.16.1.1", 
                "peer-as": 65000, 
                "peer-group-local-address": "172.16.4.1", 
                "peer-group-name": "IBGP", 
                "router_id": "172.16.4.1", 
                "vrf": "default"
            }, 
            "crud_op": "add", 
            "grpc_port": "57777", 
            "host": "100.96.0.26", 
            "password": "VALUE_SPECIFIED_IN_NO_LOG_PARAMETER", 
            "username": "rtrdev", 
            "yang_repo_location": "./yang"
        }
    }
}
META: ran handlers
META: ran handlers

PLAY RECAP ********************************************************************************************************************************************
rtr1                       : ok=1    changed=1    unreachable=0    failed=0   
rtr4                       : ok=1    changed=1    unreachable=0    failed=0   

tesuto@dev1:~/code-samples/ansible$ 


```

This is using the exact same code as the earlier YDK script in the `config_bgp_ipv4()` function above. So deconstruct the fields from `openconfig-bgp.yang model` you need to fill out, for YDK to generate the required RPC call. Then you can complete the code in the above library file.

No need to touch any part of the code other than  `config_bgp_ipv4()` (look for the comment in the code) and you should be then able to run the ansible-playbook to configure BGP for you.





### Telemetry with gNMI and YDK


It is also possible to use YDK with gNMI to create a simple telemetry collector. The code for this can be found in the `code-samples/telemetry` directory:


```python
tesuto@dev1:~$ cd ~/code-samples/telemetry/
tesuto@dev1:~/code-samples/telemetry$ pwd
/home/tesuto/code-samples/telemetry
tesuto@dev1:~/code-samples/telemetry$ 
tesuto@dev1:~/code-samples/telemetry$ cat telemetry.py 
#!/usr/bin/env python
import pdb
import json

#import logging
#logger = logging.getLogger("ydk")
#logger.setLevel(logging.INFO)
#handler = logging.StreamHandler()
#formatter = logging.Formatter(("%(asctime)s - %(name)s - %(levelname)s - %(message)s"))
#handler.setFormatter(formatter)
#logger.addHandler(handler)
import time

from ydk.models.openconfig import openconfig_interfaces
from ydk.models.openconfig import openconfig_network_instance
from ydk.path import Repository
from ydk.gnmi.providers import gNMIServiceProvider
from ydk.gnmi.services import gNMIService
from ydk.gnmi.services import gNMISubscription
from ydk.filters import YFilter
import pdb, sys
import json
import grpc
from gnmi_pb2 import SubscribeResponse

from google.protobuf import text_format
from google.protobuf.json_format import MessageToJson
from kafka import KafkaConsumer, KafkaProducer

producer = KafkaProducer(bootstrap_servers='100.96.0.22:9092')

# Callback function to handle telemetry data
def push_to_kafka(var):
    response = SubscribeResponse()
    text_format.Parse(var, response)
    jsonResponse = MessageToJson(response)
    producer.send('gnmi-telemetry', json.dumps(jsonResponse).encode('utf-8'))


# Function to subscribe to telemetry data
def subscribe(func):
    gnmi = gNMIService()

    try:
        try:
            #Running on dev1 environment of tesuto cloud setup for Nanog75
            repository = Repository('/home/tesuto/yang/')
        except:
            #Running in a docker container started off image akshshar/nanog75-telemetry
            repository = Repository('/root/yang/')
            raise
    except Exception as e:
        print("Failed to import yang models, check repository path,  aborting....")
        sys.exit(1)

    provider = gNMIServiceProvider(repo=repository, address='100.96.0.14', port=57777, username='rtrdev', password='nanog75sf')

    # The below will create a telemetry subscription path 'openconfig-interfaces:interfaces/interface'
    interfaces = openconfig_interfaces.Interfaces()
    interface = openconfig_interfaces.Interfaces.Interface()
    interfaces.interface.append(interface)

    network_instances = openconfig_network_instance.NetworkInstances()


    subscription = gNMISubscription()
    subscription.entity = interfaces #network_instances

    subscription.subscription_mode = "SAMPLE";
    subscription.sample_interval = 10* 1000000000;
    subscription.suppress_redundant = False;
    subscription.heartbeat_interval = 100 * 1000000000;

    # Subscribe for updates in STREAM mode.
    var = gnmi.subscribe(provider, subscription, 10, "STREAM", "PROTO", func)


if __name__ == "__main__":
    subscribe(push_to_kafka)

tesuto@dev1:~/code-samples/telemetry$ 


```

It can be seen from the above code YDK can be used with gNMI to set up a streaming subscriber that creates a callback to a function `push_to_kafka` in the code above. In other words, this code automatically pushes data received into the Kafka bus running on `dev1` at `100.96.0.20:9092`.

As the code comments above suggest, the `telemetry.py` subscribes to the openconfig yang model `openconfig-interface.yang` by creating an object using the YDK generated bindings:

```
# The below will create a telemetry subscription path 'openconfig-interfaces:interfaces/interface'
    interfaces = openconfig_interfaces.Interfaces()
    interface = openconfig_interfaces.Interfaces.Interface()
    interfaces.interface.append(interface)
```


Running this script directly will push data to kafka automatically. The Day2 team will be provided with a consumer for kafka that can read the same pushed data.


### Task 2: Spinning up Telemetry collector with Ansible


The Telemetry collector as shown above is simply a python script. We need to create a service out of this that can be spun up by Ansible.

This is simple. We have already created a docker image for you that contains all the dependencies for the script to run on dockerhub:  

><https://cloud.docker.com/u/akshshar/repository/docker/akshshar/nanog75-telemetry>

with the tag:  `akshshar/nanog75-telemetry`





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




## Your Task


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


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

Now connect to router `rtr1` over SSH to see the effect of this YDK script:

Username: rtrdev
Password: nanog75sf
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

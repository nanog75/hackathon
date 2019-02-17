---
published: true
date: '2019-02-17 05:26 -0800'
title: Traffic Engineering Controller using gRIBI
author: Nanog75 Hackathon Team
excerpt: >-
  Use gRIBI to engineer LSP paths and respond to Telemetry and/or application
  events to trigger LSP path changes
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---


{% include toc %}
{% include base_path %}



## Understanding Day2 Operations

Day2 Operations tend to vary across deployments since they are quite intrinsically tied to the nature of the network Operator's business model and the operator's core focus.  

For an infrastructure provider, the network itself is the most important entity that needs to be protected from failures and is also the source of telemetry/monitoring information for alarms and/or remediation decisions, usually through provisioning a new policy or configuration snippet.
  
For an application provider, much like most of the large scale Web service-providers, the network is primarily a plumbing mechanism and the health of the applications running in the datacenters is core focus. In such cases, we usually see telemetry/monitoring information being extracted out of the applications directly and remediation actions based on this information typically involve real-time traffic engineering or route manipulation.   

   
   
The second scenario is what often leads to the need for low-level highly performant APIs that can help an operator create ephemeral state (non-static, not saved in config) in the router's control plane or directly the data plane in response to real-time monitoring data.   


Openconfig's gRIBI API is one such api that aims to provide a standard inteface to the RIB and MPLS database of a router to enable manipulation of routes and Incoming Label Mapping (ILM) entries with high performance expectations.    

The API model (proto files) can be found here : < https://github.com/openconfig/gribi>



## Before we Begin


`rtr2` is exclusively for use by the Day0 group in your team. You will be able to incorporate the changes meant for rtr2 and test them only after the Day0 group successfully performs ZTP on rtr2. {: .notice--warning} 

Make sure you have access to a Pod based on the pod number assigned to you.
The instructions to connect to your pod can be found here: 

>[Connect to your pod]({{ base_path }}/assets/NANOG75_Hackathon_Lab_Info.pdf)


The topology for the hack is shown below: 

![]({{ base_path }}/images/topology_nanog75.png)  


{% capture "connect_text" %}
### Connecting to the nodes in the topology:  

You will need the tesuto private key to ssh to the instances in your topology.  

This key can be downloaded from here:    
><https://storage.googleapis.com/tesuto-public/nanog75.key>  

Download and save the key in your local machine. This document assumes you saved it to ~/nanog75.key 

```
curl -o ~/nanog75.key https://storage.googleapis.com/tesuto-public/nanog75.key 
```

Change the permission of the key file before using it:

```
chmod 400 ~/nanog75.key
```


Assuming the pod number assigned to you is `x`, the FQDN to access each of the nodes in the topology is: 

> 1. **JumpHost**:     hackathon.pod`x`.cloud.tesuto.com
> 2. **ztp**:          ztp.hackathon.pod`x`.cloud.tesuto.com
> 3. **dev1**:         dev1.hackathon.pod`x`.cloud.tesuto.com
> 4. **dev2**:         dev2.hackathon.pod`x`.cloud.tesuto.com

Using the key downloaded, ssh to the instances like so (for example for the dev2 node):

```
ssh -i ~/nanog75.key tesuto@dev2.hackathon.podx.cloud.tesuto.com

```
{% endcapture %}


<div class="notice--info">
  {{ connect_text | markdownify }}
 </div>





## Task 1: Create a gRIBI controller app



### Topology paths

The figure below shows the possible LSP paths in the current topology:

![lsp_paths.png]({{site.baseurl}}/images/lsp_paths.png)


The gRIBI controller must therefore have predefined policies on the label PUSH, SWAP and POP operations to be performed per path.
Once the Telemetry information is received via the kafka bus, any network event (like shut of an interface or interface counters or BGP session state etc.) could be utilized as a trigger to change the currently programmed LSP path.



### Looking at the Base code


### Connecting to dev2 box 


```
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@dev2.hackathon.pod0.cloud.tesuto.com
Warning: the ECDSA host key for 'dev2.hackathon.pod0.cloud.tesuto.com' differs from the key for the IP address '35.230.50.124'
Offending key for IP in /Users/akshshar/.ssh/known_hosts:2
Matching host key in /Users/akshshar/.ssh/known_hosts:9
Are you sure you want to continue connecting (yes/no)? yes
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 17 14:10:02 UTC 2019

  System load:  0.07              Users logged in:        0
  Usage of /:   9.7% of 21.35GB   IP address for ens3:    100.96.0.24
  Memory usage: 3%                IP address for ens4:    10.8.1.20
  Swap usage:   0%                IP address for docker0: 172.17.0.1
  Processes:    110

 * 'snap info' now shows the freshness of each channel.
   Try 'snap info microk8s' for all the latest goodness.


  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

7 packages can be updated.
0 updates are security updates.


Last login: Sun Feb 17 14:09:48 2019 from 128.107.241.176
tesuto@dev2:~$ 
tesuto@dev2:~$ 
```


### JSON abstraction for gRIBI


To save some time in the creation of gRIBI app, we have already coded up a gRPC client that talks to the gRIBI API exposed by IOS-XR.

Further, we have abstracted out the individual pieces of the API into a JSON representation so that creation of a new LSP path is simply equivalent to creating and using a new json file.

To view the base code, let's clone the <https://github.com/nanog75/code-samples> repository:


### Clone the code-samples repo

First let's install the `tree` package to help us analyse the available code in the `code-samples` repo.


```
tesuto@dev2:~$ sudo apt-get install -y tree
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  tree
0 upgraded, 1 newly installed, 0 to remove and 6 not upgraded.
Need to get 0 B/40.7 kB of archives.
After this operation, 105 kB of additional disk space will be used.
Selecting previously unselected package tree.
(Reading database ... 75456 files and directories currently installed.)
Preparing to unpack .../tree_1.7.0-5_amd64.deb ...
Unpacking tree (1.7.0-5) ...
Setting up tree (1.7.0-5) ...
Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
tesuto@dev2:~$ 
tesuto@dev2:~$ 
```


Now clone the repo located at:

><https://github.com/nanog75/code-samples.git>


```
tesuto@dev2:~$ git clone https://github.com/nanog75/code-samples.git
Cloning into 'code-samples'...
remote: Enumerating objects: 1594, done.
remote: Counting objects: 100% (1594/1594), done.
remote: Compressing objects: 100% (621/621), done.
remote: Total 1594 (delta 996), reused 1543 (delta 960), pack-reused 0
Receiving objects: 100% (1594/1594), 12.74 MiB | 7.12 MiB/s, done.
Resolving deltas: 100% (996/996), done.
Checking out files: 100% (1360/1360), done.
tesuto@dev2:~$ 
``` 
  
Drop into the `code-samples/gribi` directory and dump the files using tree: 

```
tesuto@dev2:~/code-samples/gribi$ tree .
.
├── protos
│   ├── enums.proto
│   ├── gribi.proto
│   ├── gribi_aft.proto
│   ├── yext.proto
│   └── ywrapper.proto
└── src
    ├── Makefile
    ├── __init__.py
    ├── genpy
    │   ├── __init__.py
    │   ├── enums_pb2.py
    │   ├── gribi_aft_pb2.py
    │   ├── gribi_pb2.py
    │   ├── yext_pb2.py
    │   └── ywrapper_pb2.py
    ├── gribi_api
    │   ├── __init__.py
    │   ├── exceptions.py
    │   ├── gribi_api.py
    │   └── serializers.py
    ├── gribi_client
    │   ├── __init__.py
    │   ├── gribi_client.py
    │   ├── gribi_template.json
    │   ├── path3
    │   │   ├── r1.gribi.json
    │   │   ├── r3.gribi.json
    │   │   └── r4.gribi.json
    │   ├── path3_add_lsp.sh
    │   └── path3_delete_lsp.sh
    └── util
        ├── __init__.py
        └── util.py

7 directories, 27 files
tesuto@dev2:~/code-samples/gribi$ 
```

The `protos` directory contains the protofiles (models) for gribi. The generated python bindings for these models is located in `src/genpy`. 
The base library created for the client is under `gribi_api/`.
The gribi_client that accepts a json based input is in the `gribi_client` directory.


### View pre-created LSP json


Let's the view the json files that are pre-created for path2 as shown in the figure above (green from rtr1 to rtr3 to rtr4). 

Drop into the directory `gribi_client/path3` directory:

```
tesuto@dev2:~$ cd code-samples/gribi/src/gribi_client/path3/
tesuto@dev2:~/code-samples/gribi/src/gribi_client/path3$ ls
r1.gribi.json  r3.gribi.json  r4.gribi.json
tesuto@dev2:~/code-samples/gribi/src/gribi_client/path3$ 

```


Let's look at `r1.gribi.json`:


```
tesuto@dev2:~/code-samples/gribi/src/gribi_client/path3$ cat r1.gribi.json 
{
    "_copyright_": "All rights reserved. Copyright (c) 2018 by Cisco Systems, Inc.",
    "global_init": {
        "major": 0,
        "minor": 0,
        "sub_ver": 0
    },
    "oc_aft_encap_type" : [
        "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_UNSET",
        "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_GRE",
        "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4",
        "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV6",
        "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_MPLS"
    ],
    "gribi_nhs" : [
        {
            "id" : 1000,
            "inst_name" : "default",
            "key_index" : 1,
            "entry" : {
                "encap_header_type" : "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_MPLS",
                "int" : "GigabitEthernet0/0/0/2",
                "subint" : 0,
                "ip_address": "10.3.1.20",
                "mac_address" : "",
                "origin_protocol": "OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC",
                "pushed_mpls_label_stack" : [
                    17010
                ]
            }
        },
        {
            "id" : 1001,
            "inst_name" : "default",
            "key_index" : 2,
            "entry" : {
                "encap_header_type" : "OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4",
                "int" : "GigabitEthernet0/0/0/0",
                "subint" : 0,
                "ip_address": "10.1.1.10",
                "mac_address" : "",
                "origin_protocol": "OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC",
                "pushed_mpls_label_stack" : [
                ]
            }
        }
    ],
    "gribi_nh_groups" : [
        {
            "id" : 1100,
            "inst_name" : "default",
            "key_id" : 1,
            "entry" : {
                "backup_nh_group" : 0,
                "color" : 0,
                "nh_keys" : [
                    {
                        "key_index" : 1,
                        "nh_weight" : 100
                    }
                ]
            }
        },
        {   
            "id" : 1101,
            "inst_name" : "default",
            "key_id" : 2,
            "entry" : {
                "backup_nh_group" : 0,
                "color" : 0,
                "nh_keys" : [
                    {   
                        "key_index" : 2,
                        "nh_weight" : 100
                    }  
                ]
            }
        }
    ],
    "gribi_routes" : [
        {
            "id" : 1200,
            "inst_name" : "default",
            "entry" : {
                "prefix" : "10.8.1.0/24",
                "type" : "v4",
                "encap_header_type" :"OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4",
                "next_hop_group" : 1,
                "octets_forwarded" : 0,
                "packets_forwarded" : 0
            }
        }
    ],
    "gribi_mpls" : [
        {
            "id" : 1301,
            "inst_name" : "default",
            "label" : 17010,
            "entry" : {
                "nh_group" : 2,
                "octets_forwarded" : 1,
                "packets_forwarded" : 1,
                "popped_mpls_label_stack" : [
                    17010
                ]
            }
        }
    ]
}


```

The way to read this is as follows:

1. First consider the `gribi_routes` entry. This always corresponds to the label imposition entry that also pushes a route to our destination (rtr4-dev2 subnet) into the rib of rtr1.   
So the `gribi_routes` entry with `id:1200` will create prefix "10.8.1.0/24" in the rtr1 rib with `next_hop_group: 1`.  
If you browse up, `next_hop_group: 1` corresponds to the next_hop_group with `key_id : 1` in `gribi_nh_groups` structure.   
Further this `next_hop_group: 1` then points to a single next hop (`nh_keys`) with `key_index: 1`.  
Lastly, this `key_index: 1` corresponds to next hop with `key_id: 1` in the `gribi_nhs` data structure.  
In other words, a route to `10.8.1.0/24` via next_hop 10.3.1.20 and Gig0/0/0/2 will be pushed into the RIB of router rtr1. Further this route will push a single label = 17010 on the packets using this route as evidenced by the `pushed_mpls_label_stack` entry in the `gribi_nhs` structure for `key_index: 1`



2. Now consider the `gribi_mpls` entries and follow the same pattern. In this case an entry will be created in Label Switch Database (seen through `show mpls forwarding`) on XR to pop an incoming 17010 label and forward the packet for an IPv4 lookup.



In this way, try to write down the label entries and route entries that will created by r2.gribi.json and r3.gribi.json.

Effectively, these input json files will help create an LSP path from rtr1 to rtr4 via rtr3 such that packets entering rtr1 from dev1, destined towards 10.8.1.0/24 via rtr1 will follow this path and packets entering rtr4 from dev2, destined towards 10.1.1.0/24 will follow the reverse path.




### Test the gribi client

To test this client, start a ping on dev2 towards 10.1.1.10 (ens4 interface of dev1 connected to rtr1 Gig0/0/0/0). Initially the pings will not go through:


```
tesuto@dev2:~$ 
tesuto@dev2:~$ ping 10.1.1.10
PING 10.1.1.10 (10.1.1.10) 56(84) bytes of data.






```



Now, dump the contents of the `path3_add_lsp.sh` script provided as part of the code samples: 


```
tesuto@dev2:~$ cd code-samples/gribi/src/gribi_client/
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ 
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ 
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ cat path3_add_lsp.sh 
#!/bin/bash

python3 gribi_client.py -j path3/r1.gribi.json -ip 100.96.0.14 -port 57777 -p 
python3 gribi_client.py -j path3/r3.gribi.json -ip 100.96.0.18 -port 57777 -p
python3 gribi_client.py -j path3/r4.gribi.json -ip 100.96.0.26 -port 57777 -p
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ 
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ 

```


Finally, let's run this script:

```
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ 
tesuto@dev2:~/code-samples/gribi/src/gribi_client$ ./path3_add_lsp.sh 
Next Hop
operation {
  id: 1000
  network_instance: "default"
  op: ADD
  next_hop {
    index: 1
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_MPLS
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.3.1.20"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/2"
        }
        subinterface {
        }
      }
      mac_address {
      }
      pushed_mpls_label_stack {
        pushed_mpls_label_stack_uint64: 17010
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop
operation {
  id: 1001
  network_instance: "default"
  op: ADD
  next_hop {
    index: 2
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.1.1.10"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/0"
        }
        subinterface {
        }
      }
      mac_address {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 1100
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 1
    next_hop_group {
      next_hop {
        index: 1
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 1101
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 2
    next_hop_group {
      next_hop {
        index: 2
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Route
operation {
  id: 1200
  network_instance: "default"
  op: ADD
  ipv4 {
    prefix: "10.8.1.0/24"
    ipv4_entry {
      packets_forwarded {
      }
      octets_forwarded {
      }
      decapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4
      next_hop_group {
        value: 1
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Mpls
operation {
  id: 1301
  network_instance: "default"
  op: ADD
  mpls {
    label_entry {
      popped_mpls_label_stack {
        popped_mpls_label_stack_uint64: 17010
      }
      next_hop_group {
        value: 2
      }
      packets_forwarded {
        value: 1
      }
      octets_forwarded {
        value: 1
      }
    }
    label_uint64: 17010
  }
}

Got a response!
result {
  id: 1301
  status: OK
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop
operation {
  id: 2000
  network_instance: "default"
  op: ADD
  next_hop {
    index: 1
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_MPLS
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.6.1.20"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/2"
        }
        subinterface {
        }
      }
      mac_address {
      }
      pushed_mpls_label_stack {
        pushed_mpls_label_stack_uint64: 16030
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop
operation {
  id: 2001
  network_instance: "default"
  op: ADD
  next_hop {
    index: 2
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_MPLS
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.3.1.10"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/0"
        }
        subinterface {
        }
      }
      mac_address {
      }
      pushed_mpls_label_stack {
        pushed_mpls_label_stack_uint64: 17010
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 2100
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 1
    next_hop_group {
      next_hop {
        index: 1
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 2101
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 2
    next_hop_group {
      next_hop {
        index: 2
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Mpls
operation {
  id: 2300
  network_instance: "default"
  op: ADD
  mpls {
    label_entry {
      popped_mpls_label_stack {
        popped_mpls_label_stack_uint64: 17010
      }
      next_hop_group {
        value: 1
      }
      packets_forwarded {
        value: 1
      }
      octets_forwarded {
        value: 1
      }
    }
    label_uint64: 17010
  }
}

Got a response!
result {
  id: 2300
  status: OK
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Mpls
operation {
  id: 2302
  network_instance: "default"
  op: ADD
  mpls {
    label_entry {
      popped_mpls_label_stack {
        popped_mpls_label_stack_uint64: 16030
      }
      next_hop_group {
        value: 2
      }
      packets_forwarded {
        value: 1
      }
      octets_forwarded {
        value: 1
      }
    }
    label_uint64: 16030
  }
}

Got a response!
result {
  id: 2302
  status: OK
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop
operation {
  id: 3000
  network_instance: "default"
  op: ADD
  next_hop {
    index: 1
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.6.1.10"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/2"
        }
        subinterface {
        }
      }
      mac_address {
      }
      pushed_mpls_label_stack {
        pushed_mpls_label_stack_uint64: 16030
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop
operation {
  id: 3001
  network_instance: "default"
  op: ADD
  next_hop {
    index: 2
    next_hop {
      encapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4
      origin_protocol: OPENCONFIGPOLICYTYPESINSTALLPROTOCOLTYPE_STATIC
      ip_address {
        value: "10.8.1.20"
      }
      interface_ref {
        interface {
          value: "GigabitEthernet0/0/0/0"
        }
        subinterface {
        }
      }
      mac_address {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 3100
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 1
    next_hop_group {
      next_hop {
        index: 1
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Next Hop Group
operation {
  id: 3101
  network_instance: "default"
  op: ADD
  next_hop_group {
    id: 2
    next_hop_group {
      next_hop {
        index: 2
        next_hop {
        }
      }
      color {
      }
      backup_next_hop_group {
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Route
operation {
  id: 3200
  network_instance: "default"
  op: ADD
  ipv4 {
    prefix: "10.1.1.0/24"
    ipv4_entry {
      packets_forwarded {
      }
      octets_forwarded {
      }
      decapsulate_header: OPENCONFIGAFTENCAPSULATIONHEADERTYPE_IPV4
      next_hop_group {
        value: 1
      }
    }
  }
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True
Mpls
operation {
  id: 3300
  network_instance: "default"
  op: ADD
  mpls {
    label_entry {
      popped_mpls_label_stack {
        popped_mpls_label_stack_uint64: 16030
      }
      next_hop_group {
        value: 2
      }
      packets_forwarded {
        value: 1
      }
      octets_forwarded {
        value: 1
      }
    }
    label_uint64: 16030
  }
}

Got a response!
result {
  id: 3300
  status: OK
}

<_Rendezvous of RPC that terminated with (StatusCode.OK, )>
True

```






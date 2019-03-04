---
published: true
date: '2019-02-17 01:54 -0800'
title: 'Step 1:  Understand Day 1 Operations'
author: Nanog75 Hackathon Team
excerpt: Understand how to perform Day 1 operations and the end goal.
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---

{% include toc %}
{% include base_path %}



## Day 1 Operations

Day1 operations would comprise any service or feature activation or application deployment that is orchestrated on the routers after the Day0 operations have applied a base configuration on the routers.  

## Before we Begin

**Note**:  
`rtr2` is exclusively for use by the Day0 group in your team. You will be able to incorporate the changes meant for rtr2 and test them only after the Day0 group successfully performs ZTP on rtr2. 
{: .notice--warning} 

Make sure you have access to a Pod based on the pod number assigned to you.
The instructions to connect to your pod can be found here: 

>[Connect to your pod]({{ base_path }}/connect-to-pods/)



## Basic Components

In the current topology for Day 1 (rtr2 is not available for Day1 and Day2 operations until Day0 group successfully bootstraps it),  the components we intend to use are:

1. Ansible to orchestrate all the configurations and applications on the network and on all routers.

2. gNMI as a way of communicating with the router for configuration using Yang models over gRPC as well as to subscribe to telemetry data from the routers.

3. YDK to enable the use of Yang models through python objects. Learn more about YDK here: <http://ydk.io> and its integration with gNMI here : <https://github.com/CiscoDevNet/ydk-py/tree/master/gnmi/samples>


4.  Open/R to act as an IGP on the IOS-XR routers and distribute loopback addresses so that BGP sessions can come up.





## Looking at Base code


## Connect to the dev1 box


For the Day1 group, all of the operations will be performed on the dev1 box.

Based on the pod number (assuming `x`), connect to your dev1 box:


```
AKSHSHAR-M-33WP:~ akshshar$ 
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@dev1.hackathon.podx.cloud.tesuto.com
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 17 10:14:22 UTC 2019

  System load:  0.0                Users logged in:                0
  Usage of /:   48.8% of 21.35GB   IP address for ens3:            100.96.0.22
  Memory usage: 19%                IP address for ens4:            10.1.1.10
  Swap usage:   0%                 IP address for docker0:         172.17.0.1
  Processes:    116                IP address for br-eabb3a76ca77: 172.21.0.1

 * 'snap info' now shows the freshness of each channel.
   Try 'snap info microk8s' for all the latest goodness.


  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

7 packages can be updated.
0 updates are security updates.


Last login: Sun Feb 17 10:00:46 2019 from 128.107.241.176

tesuto@dev1:~$ 
tesuto@dev1:~$ 


```



### Clone the code-samples repo

First let's install the `tree` package to help us analyse the available code in the `code-samples` repo.


```
tesuto@dev1:~$ sudo apt-get install -y tree
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
tesuto@dev1:~$ 
tesuto@dev1:~$ 
```


Now clone the repo located at:

><https://github.com/nanog75/code-samples.git>


```
tesuto@dev1:~$ git clone https://github.com/nanog75/code-samples.git
Cloning into 'code-samples'...
remote: Enumerating objects: 1594, done.
remote: Counting objects: 100% (1594/1594), done.
remote: Compressing objects: 100% (621/621), done.
remote: Total 1594 (delta 996), reused 1543 (delta 960), pack-reused 0
Receiving objects: 100% (1594/1594), 12.74 MiB | 7.12 MiB/s, done.
Resolving deltas: 100% (996/996), done.
Checking out files: 100% (1360/1360), done.
tesuto@dev1:~$ 
``` 
  
Drop into the code-samples directory:  

```
tesuto@dev1:~$ 
tesuto@dev1:~$ cd code-samples/
```


Excluding the pre-packaged yang models (using the `-I` option with tree, we see the available files as shown below:  

```
tesuto@dev1:~/code-samples$ 
tesuto@dev1:~/code-samples$ 
tesuto@dev1:~/code-samples$ tree -I yang
.
├── README.md
├── ansible
│   ├── ansible_hosts
│   └── playbooks
│       ├── config_bgp
│       │   ├── config_oc_bgp_ydk.yml
│       │   ├── config_xr_bgp_netconf.yml
│       │   ├── library
│       │   │   └── config_bgp_oc_ydk.py
│       │   └── xml
│       │       ├── rtr1-bgp.xml
│       │       └── rtr4-bgp.xml
│       ├── openr_bringup
│       │   ├── automate_docker_setup.py
│       │   ├── docker_bringup.yml
│       │   └── openr
│       │       ├── hosts_rtr1
│       │       ├── hosts_rtr2
│       │       ├── hosts_rtr3
│       │       ├── hosts_rtr4
│       │       ├── increment_ipv4_prefix1.py
│       │       ├── increment_ipv4_prefix2.py
│       │       ├── launch_openr_rtr1.sh
│       │       ├── launch_openr_rtr2.sh
│       │       ├── launch_openr_rtr3.sh
│       │       ├── launch_openr_rtr4.sh
│       │       ├── run_openr_rtr1.sh
│       │       ├── run_openr_rtr2.sh
│       │       ├── run_openr_rtr3.sh
│       │       └── run_openr_rtr4.sh
│       └── reachability_check
│           ├── ip_dest_reachable_ydk.yml
│           ├── library
│           │   └── ip_destination_reachable.py
│           └── ping_variables.yml
├── gribi
│   ├── protos
│   │   ├── enums.proto
│   │   ├── gribi.proto
│   │   ├── gribi_aft.proto
│   │   ├── yext.proto
│   │   └── ywrapper.proto
│   └── src
│       ├── Makefile
│       ├── __init__.py
│       ├── genpy
│       │   ├── __init__.py
│       │   ├── enums_pb2.py
│       │   ├── gribi_aft_pb2.py
│       │   ├── gribi_pb2.py
│       │   ├── yext_pb2.py
│       │   └── ywrapper_pb2.py
│       ├── gribi_api
│       │   ├── __init__.py
│       │   ├── exceptions.py
│       │   ├── gribi_api.py
│       │   └── serializers.py
│       ├── gribi_client
│       │   ├── __init__.py
│       │   ├── gribi_client.py
│       │   ├── gribi_template.json
│       │   ├── path3
│       │   │   ├── r1.gribi.json
│       │   │   ├── r3.gribi.json
│       │   │   └── r4.gribi.json
│       │   ├── path3_add_lsp.sh
│       │   └── path3_delete_lsp.sh
│       └── util
│           ├── __init__.py
│           └── util.py
├── telemetry
│   ├── gnmi_pb2.py
│   ├── kafka_consumer.py
│   └── telemetry.py
├── ydk
│   └── config_oc_bgp_ydk.py
└── ztp
    ├── dhcp
    │   └── dhcpd.conf
    └── web_server
        ├── configs
        │   ├── rtr1.config
        │   ├── rtr2.config
        │   ├── rtr3.config
        │   └── rtr4.config
        ├── packages
        │   └── python-pip-7.1.0-r0.0.core2_64.rpm
        ├── scripts
        │   ├── restart_docker_xr.sh
        │   ├── retry_manual_ztp.sh
        │   └── ztp_ncclient.py
        └── xml
            └── rtr2
                ├── grpc_config.xml
                ├── hostname.xml
                └── mpls_static.xml

27 directories, 69 files
tesuto@dev1:~/code-samples$ 

```


### High Level Tasks

For the Day1 group, the relevant top level directories are :


1. `ansible`:  This directory contains all the available playbooks for you to understand, modify and add to. These playbooks will eventually need to be combined into a single playbook.
You will eventually need to have a playbook for each of the following operations:
    *  Configure BGP on rtr1 and rtr4 using YDK
    *  Bring up Open/R docker instances as IGP on all the routers(except rtr2, not until later)
    *  Test reachability on each router for the Open/R distributed loopbacks by leveraging YDK to execute pings on the IOS-XR routers through netconf actions.
    *  Set up the telemetry collector to push data to the local kafka bus.

2.  `ydk`:  This directory contains a sample YDK code that will help you understand how YDK works along with gNMI


3.  `telemetry`: This directory contains a telemetry collector using YDK with gNMI that you will utilize to set up monitoring data for the Day2 group. 




Proceed to [Step 2]({{ base_path }}) to start understanding the provided code and building the ansible playbooks together.

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

```


### JSON abstraction for gRIBI


To save some time in the creation of gRIBI app, we have already coded up a gRPC client that talks to the gRIBI API exposed by IOS-XR.

Further, we have abstracted out the individual pieces of the API into a JSON representation so that creation of a new LSP path is simply equivalent to creating a new json file.

To view 








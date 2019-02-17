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

Using the key downloaded, ssh to the instances like so (for example for the ZTP node):

```
ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com

```
{% endcapture %}


<div class="notice--info">
  {{ connect_text | markdownify }}
 </div>




## Connect to the ZTP box


For the Day0 group, all of the operations will be performed on the ZTP box.

Based on the pod number (assuming `x`), connect to your ZTP box first:


```
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com
Warning: the ECDSA host key for 'ztp.hackathon.podx.cloud.tesuto.com' differs from the key for the IP address '35.197.94.94'
Offending key for IP in /Users/akshshar/.ssh/known_hosts:1
Matching host key in /Users/akshshar/.ssh/known_hosts:5
Are you sure you want to continue connecting (yes/no)? yes
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 17 07:51:27 UTC 2019

  System load:  0.0               Processes:           108
  Usage of /:   9.6% of 21.35GB   Users logged in:     0
  Memory usage: 2%                IP address for ens3: 100.96.0.20
  Swap usage:   0%

 * 'snap info' now shows the freshness of each channel.
   Try 'snap info microk8s' for all the latest goodness.


  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

8 packages can be updated.
0 updates are security updates.


Last login: Sun Feb 17 01:02:43 2019 from 128.107.241.176
tesuto@ztp:~$ 
tesuto@ztp:~$ 


```


## Basic Components

In the current topology, the components we intend to use are:

1. Ansible to orchestrate all the configurations and applications on the network and on all routers.

2. Using gNMI as a way of communicating with the router for configuration using Yang models over gRPC as well as to subscribe to telemetry data from the routers.

3. YDK to enable the use of Yang models through python objects. Learn more about YDK here: <http://ydk.io> and its integration with gNMI here : <https://github.com/CiscoDevNet/ydk-py/tree/master/gnmi/samples>


4.  Open/R to act as an IGP on the IOS-XR routers and distribute loopback addresses so that BGP sessions can come up.





## Looking at Base code


### SSH to dev1


For the Day1 group, the development box to be used is `dev1`







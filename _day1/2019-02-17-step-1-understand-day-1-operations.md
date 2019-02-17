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

Using the key downloaded, ssh to the instances like so (for example for the ZTP node):

```
ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com

```
{% endcapture %}


<div class="notice--info">
  {{ connect_text | markdownify }}
 </div>




## Connect to the dev1 box


For the Day1 group, all of the operations will be performed on the dev1 box.

Based on the pod number (assuming `x`), connect to your dev1 box:


```
AKSHSHAR-M-33WP:~ akshshar$ 
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@dev1.hackathon.pod0.cloud.tesuto.com
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


## Basic Components

In the current topology for Day 1 (rtr2 is not available for Day1 and Day2 operations),  the components we intend to use are:

1. Ansible to orchestrate all the configurations and applications on the network and on all routers.

2. Using gNMI as a way of communicating with the router for configuration using Yang models over gRPC as well as to subscribe to telemetry data from the routers.

3. YDK to enable the use of Yang models through python objects. Learn more about YDK here: <http://ydk.io> and its integration with gNMI here : <https://github.com/CiscoDevNet/ydk-py/tree/master/gnmi/samples>


4.  Open/R to act as an IGP on the IOS-XR routers and distribute loopback addresses so that BGP sessions can come up.





## Looking at Base code


### SSH to dev1


For the Day1 group, the development box to be used is `dev1`







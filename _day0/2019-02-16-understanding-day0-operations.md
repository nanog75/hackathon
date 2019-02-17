---
published: true
date: '2019-02-16 23:11 -0800'
title: Understanding Day0 Operations
author: Nanog75 Hackathon Team
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
excerpt: >-
  Understand the purpose of Day0 operations in the hackathon and the various
  components and workflows involved.
---


{% include toc %}
{% include base_path %}

## Before we Begin

Make sure you have access to a Pod based on the pod number assigned to you.
The instructions to connect to your pod can be found here: 

>[Connect to your pod]({{ base_path }}/assets/NANOG75_Hackathon_Lab_Info.pdf)


The topology for the hack is shown below: 

![]({{ base_path }}/images/topology_nanog75.png)  


{% capture "connect_text" %}
### Connecting to the nodes in the topology:
You will need the tesuto private key to ssh to the instances in your topology.
This key can be downloaded from here:  <https://storage.googleapis.com/tesuto-public/nanog75.key>  
Save the key in your local machine. This document assumes you saved it to ~/nanog75.key 

```
curl -o ~/nanog75.key https://storage.googleapis.com/tesuto-public/nanog75.key 
```

Change the permission of the key file before using it:

```
chmod 400 ~/nanog75.key
```


Assuming the pod number assigned to you is `x`, the FQDN to access each of the nodes in the topology is: 

> * **JumpHost** : hackathon.podx.cloud.tesuto.com
> * **ztp**: ztp.hackathon.podx.cloud.tesuto.com
> * **dev1**:  dev1.hackathon.podx.cloud.tesuto.com
> * **dev2**:  dev2.hackathon.podx.cloud.tesuto.com

Using the key downloaded, ssh to the instances like so (for example for the ZTP node):

```
ssh -i ~/nanog75.key ztp.hackathon.podx.cloud.tesuto.com

```
{% endcapture %}


<div class="notice--info">
  {{ connect_text | markdownify }}
 </div>






---
layout: single
title: "Reserving and Connecting to Devnet Pods"
permalink: /connect-to-pods/
author_profile: true 
---

{% include base_path %}
{% include toc %}

## Topology 

The topology for the hack is shown below: 

![]({{ base_path }}/images/topology_nanog75.png)  



## Connecting to the nodes in the topology:  

### Setting up the key

You will need the tesuto private key to ssh to the instances in your topology.  

>The key location should have been sent to you via email.    
 

Download and save the key in your local machine. This document assumes you saved it to `~/nanog75.key`. 


Change the permission of the key file before using it:

```
chmod 400 ~/nanog75.key
```

### FQDN for each node
Assuming the pod number assigned to you is `x`, the FQDN to access each of the nodes in the topology is: 

| Node | FQDN|
|:-----:|:----:|
|**JumpHost**:   |  hackathon.pod`x`.cloud.tesuto.com |
|**ztp**:        |  ztp.hackathon.pod`x`.cloud.tesuto.com |
|**dev1**:       |  dev1.hackathon.pod`x`.cloud.tesuto.com |
|**dev2**:       |  dev2.hackathon.pod`x`.cloud.tesuto.com |


### SSH to each node

Using the key downloaded, ssh to the instances like so (for example for the ZTP node):

```
ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com

```



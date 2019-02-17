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

## Basic Components

In the current topology, the components we intend to use are:

1) Ansible to orchestrate all the configurations and applications on the network and on all routers.

2) YDK to enable the use of Yang models through python objects. Learn more about YDK here: <http://ydk.io> and its integration with gNMI here : <https://github.com/CiscoDevNet/ydk-py/tree/master/gnmi/samples>

3) Using YDK with GNMI





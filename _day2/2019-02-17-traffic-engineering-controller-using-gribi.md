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






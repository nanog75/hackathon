---
published: true
date: '2019-02-17 04:27 -0800'
title: 'Step 3: Spinning up Open/R and combining Ansible Playbooks'
author: Nanog75 Hackathon Team
excerpt: >-
  Spin up Open/R to bring up the BGP sessions and combine playbooks into a
  single playbook to automate Day1 operations using a single step.
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---


{% include toc %}
{% include base_path %} 



If you've accomplished the tasks in Step2, then you're one step closer to finishing off Day1 operations.

At the end of Step2, you should have:


1.  IBGP configuration applied to rtr1 and rtr4

2.  A Telemetry collector running inside a docker container collecting data over gNMI from a router of your choice and pushing the data to a kafka bus.



**Note**: Even if you were not able to complete the steps in Step 2, you can still proceed with the steps outlined below and go back to solving the problem statements in Step 2.




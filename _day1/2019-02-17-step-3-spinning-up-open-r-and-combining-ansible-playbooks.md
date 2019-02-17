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




## Spinning up Open/R as an IGP on the routers


Open/R (<https://github.com/facebook/openr>) is a routing protocol stack that was open-sourced by Facebook in late 2017.

It was designed to be easily integratable with lower level APIs provided by Vendor stacks and it was therefore straightforward to integrate with IOS-XR's Service-Layer APIs to allow Open/R to co-exist with standard IOS-XR protocols while running as an application on IOS-XR.

The integration code can be found here:

><https://github.com/akshshar/openr-xr>


Open/R runs inside a docker container on IOS-XR as shown below:


![openr-xr-deploy.png]({{site.baseurl}}/images/openr-xr-deploy.png)



## Spinning up the docker container using Ansible


We've already automated this process for you. If you jump into the `code-samples/ansible/playbooks/openr_bringup` directory and dump the set of files available:


```
tesuto@dev1:~$ 
tesuto@dev1:~$ cd ~/code-samples/ansible/playbooks/openr_bringup
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ 
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ tree .
.
├── automate_docker_setup.py
├── docker_bringup.yml
└── openr
    ├── hosts_rtr1
    ├── hosts_rtr2
    ├── hosts_rtr3
    ├── hosts_rtr4
    ├── increment_ipv4_prefix1.py
    ├── increment_ipv4_prefix2.py
    ├── launch_openr_rtr1.sh
    ├── launch_openr_rtr2.sh
    ├── launch_openr_rtr3.sh
    ├── launch_openr_rtr4.sh
    ├── run_openr_rtr1.sh
    ├── run_openr_rtr2.sh
    ├── run_openr_rtr3.sh
    └── run_openr_rtr4.sh

1 directory, 16 files
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ 
```

You'll see that there is an ansible playbook (docker_bringup.yml) along with a few scripts.
The `automate_docker_setup.py` script is a simple python script that restarts the docker daemon on the routers and pulls the `akshshar/openr-xr` docker image onto each router.
The remaining files under the `openr` directory are associated with the basic launch and configuration settings of open/R for each router.




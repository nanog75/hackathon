---
published: true
date: '2019-02-17 01:01 -0800'
title: 'Step 2:  Performing ZTP'
author: Nanog75 Hackathon Team
excerpt: >-
  Learn how to perform ZTP manually on the router and properly through the
  repair process
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---

{% include toc %}
{% include base_path %}

## Ways to perform ZTP

We have already seen that the DHCP server is set up to respond with the location of the script `ztp_ncclient.py` to router `rtr2` when it sends the appropriate ZTP request. So, let's perform 
ZTP for rtr2.

There are two ways to perform ZTP on any router in the current topology:

### Manually from console while the router is up


To initiate ZTP on router rtr2 while it is still up, first connect to the console of router `rtr2`.   

To do this, first ssh to the Base Pod VM (assuming podx):

```
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@hackathon.podx.cloud.tesuto.com
Last login: Sun Feb 17 03:04:40 2019 from 172.17.0.1
== Tesuto Jumpbox ==
Type '?' or 'help' to get the list of commands
Running Hosts:

host: rtr1
  ip: 100.96.0.14
host: rtr2
  ip: 100.96.0.16
host: rtr3
  ip: 100.96.0.18
host: ztp
  ip: 100.96.0.20
host: dev1
  ip: 100.96.0.22
host: dev2
  ip: 100.96.0.24
host: rtr4
  ip: 100.96.0.26
tesuto@jumpbox:~$ 
tesuto@jumpbox:~$ 

```

As the above output indicates, on the Base pod VM you press `?` to see the list of commands available to you.

To console into router rtr2, simply type:  `console rtr2` on the base Pod VM:

>For your convenience, we have set up a backdoor user on the router with credentials:  
>
>**Username**: rtrdev
>**Password**: nanog75sf  
>
>On an actual router, this user is not present, this is only been made available for this hackathon to make things simpler.
{: .notice--info}


```

tesuto@jumpbox:~$ console rtr2
Trying 10.138.0.20...
Connected to rtr2-console.
Escape character is '^]'.
rtr
User Access Verification

Username: rtrdev
Password: 

RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#
```


### Initiating a wipe and reset (Repair)

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

As the above output indicates, on the Base pod VM you can press `?` to see the list of commands available to you.

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

Now to initiate ZTP manually, we basically need to execute the CLI `ztp initiate noprompt` in the router CLI shell.
However, since this setup reserves a single IP address from the DHCP server for each router, executing ZTP manually from an already provisioned box (rtr2) will fail since the ISC-DHCP server will not reissue an IP to the router that already has the IP present.

To make it simple, we have a created a small script to remove the ip from the Managment port and execute `ztp initiate noprompt` for us. 

So, in order to perform manual ZTP on any of the routers in the topology, simply run the following commands in router console:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:ios#
RP/0/RP0/CPU0:ios#<mark>bash</mark>
Sun Feb 17 09:18:46.493 UTC
[host:~]$ 
[host:~]$ 
[host:~]$ <mark>wget http://100.96.0.20/scripts/retry_manual_ztp.sh</mark>
--2019-02-17 09:19:04--  http://100.96.0.20/scripts/retry_manual_ztp.sh
Connecting to 100.96.0.20:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 881 [text/x-sh]
Saving to: 'retry_manual_ztp.sh.1'

100%[======================================>] 881         --.-K/s   in 0s      

2019-02-17 09:19:04 (79.2 MB/s) - 'retry_manual_ztp.sh.1' saved [881/881]

[host:~]$ 
[host:~]$ <mark>chmod +x retry_manual_ztp.sh</mark>
[host:~]$ 
[host:~]$ <mark>./retry_manual_ztp.sh</mark>
+ read -r -d '' mgmt_clean_config
+ xrapply_string '!
interface MgmtEth0/RP0/CPU0/0
 no ipv4 address
 shutdown
!
!
end'
+ xrnns_apply_string '!
interface MgmtEth0/RP0/CPU0/0
 no ipv4 address
 shutdown
!
!
end'
++ mktemp
+ local filename=/tmp/tmp.3PLe1NGYaN
+ printf '!
interface MgmtEth0/RP0/CPU0/0
 no ipv4 address
 shutdown
!
!
end\n'
+ xrnns_apply /tmp/tmp.3PLe1NGYaN
+ local filename=/tmp/tmp.3PLe1NGYaN
+ ip netns exec xrnns /pkg/bin/ztp_exec.sh xrnns_apply_ noisy ZTP /tmp/tmp.3PLe1NGYaN
RP/0/RP0/CPU0:Feb 17 09:19:13.425 UTC: config[67046]: %MGBL-CONFIG_HIST_UPDATE-3-SYSDB_GET : Error 'sysdb' detected the 'warning' condition 'A verifier or EDM callback function returned: 'not found'' getting host address from  sysdb 
+ local ret=0
+ safe_rm_file /tmp/tmp.3PLe1NGYaN
+ [[ /tmp/tmp.3PLe1NGYaN = '' ]]
+ /bin/rm -f /tmp/tmp.3PLe1NGYaN
+ return 0
+ xrcmd 'ztp initiate noprompt'
+ xrnns_cmd 'ztp initiate noprompt'
+ ip netns exec xrnns /pkg/bin/ztp_exec.sh xrnns_cmd_ 'ztp initiate noprompt'
ZTP will now run in the background.
Please use "show logging" or look at /disk0:/ztp/ztp.log to check progress.
Killed
[host:~]$

</code>
</pre>
</div>


### Initiating a wipe and reset (Repair)

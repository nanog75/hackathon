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

## Console into your router

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




## Testing your ZTP script

You don't need to actually perform ZTP to test your ZTP script. To do this, ssh into your ZTP box:  
```
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@ztp.hackathon.pod0.cloud.tesuto.com
Warning: the ECDSA host key for 'ztp.hackathon.pod0.cloud.tesuto.com' differs from the key for the IP address '35.197.94.94'
Offending key for IP in /Users/akshshar/.ssh/known_hosts:1
Matching host key in /Users/akshshar/.ssh/known_hosts:5
Are you sure you want to continue connecting (yes/no)? yes
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 17 09:38:59 UTC 2019

  System load:  0.0               Processes:           110
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


Last login: Sun Feb 17 07:51:28 2019 from 128.107.241.176
tesuto@ztp:~$ 
tesuto@ztp:~$ 
tesuto@ztp:~$ 
tesuto@ztp:~$ 

```

Simply console into the router, drop into bash, down



## Testing the ZTP process

We have already seen that the DHCP server is set up to respond with the location of the script `ztp_ncclient.py` to router `rtr2` when it sends the appropriate ZTP request. So, let's perform 
ZTP for rtr2.

There are two ways to perform ZTP on any router in the current topology:

### Manually from console while the router is up


To initiate ZTP on router rtr2 while it is still up, first connect to the console of router `rtr2`.   



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

**Note**:  You only need to do this to test your ZTP script once it is ready. Repair takes a long time (10+ minutes). Use the manual ZTP trigger process shown above to initiate the ZTP process while coding the script.
{: .notice--warning} 


The other option is to perform a repair on the node from the Tesuto UI.

For this purpose login to the Tesuto UI at :   

><https://cloud.tesuto.com>  

with the following credentials:

**Username**: pod**x**@tesuto.com # Replace x with your pod number
**Password**: nanog75sf
{: .notice--info}



![tesuto_repair_1.png]({{site.baseurl}}/images/tesuto_repair_1.png)


Now click on `hackathon` and it will open up the devices page for you. Hover to the button on right of the name `rtr2`, click and select `repair`.


![tesuto_repair2.png]({{site.baseurl}}/images/tesuto_repair2.png)


You will get a prompt, select `yes` and this will wipe out the rtr2 disk and boot from scratch.
This process is equivalent to provisioning a fresh router without config in your network.

![tesuto_repair3.png]({{site.baseurl}}/images/tesuto_repair3.png)


**Note**:

{: .notice--warning}

### Logs During Execution.

When ZTP is executing, to view the ongoing logs, console into the router, drop into `bash` and do a `tail -f` on `/var/log/ztp.log`





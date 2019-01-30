---
published: true
date: '2018-06-12 08:47 -0400'
title: 'Step 1: IOS-XR ZTP Bash and Python hooks'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - devnet
  - CLUS2018
  - programmability
excerpt: Using ZTP hooks in bash and python to automate IOS-XR cli
---


{% include toc %}

## To Know More

As part of the Zero-touch provisioning infrastructure, IOS-XR provides the option to automate CLI operations in either bash or python in the IOS-XR shell environment. This enables a large variety of integrations - including native scripts and offbox integrations with tools like Ansible

Complete CLI support: 
*  Automate CLI operations such as "show commands", "config merge", "config replace"
*  Available in bash and python: Choose the scripting language you prefer and integrate with ztp, cronjobs, onbox-apps and more


>Check out:
>
>1) [IOS-XR Bash ZTP hooks](https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp_helpersh)
>2) [IOS-XR Python ZTP library](https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/#ztp_helpersh)


>Connect to your Pod first! Make sure your Anyconnect VPN connection to the Pod assigned to you is active. 
>
> If you haven't connected yet, check out the instructions to do so here: 
><https://iosxr-devnet-ciscolive.github.io/cleur2019-workshop/assets/CLEUR19-IOS-XR-Programmability-Workshop.pdf>
>
>
> Once you're connected, use the following instructions to connect to the individual nodes.
> The instructions in the workshop will simply refer to the Name of the box to connect without
> repeating the connection details and credentials. So refer back to this list when you need it.
>  
>
> The 3 nodes in the topology are: 
> 
><p style="font-size: 16px;"><b>Development Linux System (DevBox)</b></p> 
>      IP Address: 10.10.20.170
>      Username/Password: [admin/admin]
>      SSH Port: 2211
> 
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R1: (Router r1)</b></p> 
>
>     IP Address: 10.10.20.170  
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170  
>     XR SSH Port: 2221    
>     NETCONF Port: 8321   
>     gRPC Port: 57021  
>     XR-Bash SSH Port: 2222    
>
>
><p style="font-size: 16px;"><b>IOS-XRv9000 R2:  (Router r2)</b></p> 
>
>     IP Address: 10.10.20.170   
>     Username/Password: [admin/admin]   
>     Management IP: 10.10.20.170   
>     XR SSH Port: 2231    
>     NETCONF Port: 8331   
>     gRPC Port: 57031    
>     XR-Bash SSH Port: 2232
{: .notice--info}


The Topology in use is shown below:
![topology_devnet.png]({{site.baseurl}}/images/topology_devnet.png)




## SSH into the devbox 

Drop into the devbox using the credentials above and clone the following git repository:
<https://github.com/akshshar/iosxr-devnet-cleur2019>


```

AKSHSHAR-M-33WP:~ akshshar$ 
AKSHSHAR-M-33WP:~ akshshar$ ssh -p 2211 admin@10.10.20.170
admin@10.10.20.170's password: 
Last login: Tue Jan 29 18:33:43 2019 from 192.168.122.1
admin@devbox:~$ 
admin@devbox:~$ 
admin@devbox:~$ 
admin@devbox:~$ git clone https://github.com/akshshar/iosxr-devnet-cleur2019.git
Cloning into 'iosxr-devnet-cleur2019'...
remote: Enumerating objects: 33, done.
remote: Counting objects: 100% (33/33), done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 33 (delta 8), reused 33 (delta 8), pack-reused 0
Unpacking objects: 100% (33/33), done.
Checking connectivity... done.
admin@devbox:~$ 
admin@devbox:~$ 

```

You should see the following files:

```
admin@devbox:~$ cd iosxr-devnet-cleur2019/
admin@devbox:iosxr-devnet-cleur2019$ 
admin@devbox:iosxr-devnet-cleur2019$ tree .
.
├── ansible
│   ├── ansible_hosts
│   ├── configure_bgp_oc_netconf.yml
│   ├── docker_bringup.retry
│   ├── docker_bringup.yml
│   ├── execute_python_ztp.yml
│   ├── openr
│   │   ├── hosts_r1
│   │   ├── hosts_r2
│   │   ├── increment_ipv4_prefix1.py
│   │   ├── increment_ipv4_prefix2.py
│   │   ├── launch_openr_r1.sh
│   │   ├── launch_openr_r2.sh
│   │   ├── run_openr_r1.sh
│   │   └── run_openr_r2.sh
│   ├── set_ipv6_route.sh
│   └── xml
│       ├── r1-bgp.xml
│       └── r2-bgp.xml
├── README.md
└── ztp_hooks
    ├── automate_cli_bash.sh
    └── automate_cli_python.py

4 directories, 19 files
admin@devbox:iosxr-devnet-cleur2019$ 
```


cd into the ztp_hooks directory and you should a couple of files we will deal with in this section

```
admin@devbox:iosxr-devnet-cleur2019$ 
admin@devbox:iosxr-devnet-cleur2019$ cd ztp_hooks/
admin@devbox:ztp_hooks$ ls
automate_cli_bash.sh  automate_cli_python.py
admin@devbox:ztp_hooks$ 

```

## Bash ZTP script
The bash script will utilize the ZTP bash hooks in IOS-XR to configure `Loopback0` and `grpc` on each router.

>To learn about all the ZTP Bash hooks available in IOS-XR use the following learning lab on DevNet:
><https://learninglabs.cisco.com/tracks/iosxr-programmability/iosxr-cli-automation/01-iosxr-01-cli-automation-bash/step/1>
{: .notice--warning}



### Transfer the bash script to Router r1 over SSH

password for `Router r1` is `admin`
{: .notice--info}  


```



```

### Execute the copied bash script over ssh 

```
admin@devbox:ztp_hooks$ ssh -p 2221  admin@10.10.20.170 run /misc/scratch/automate_cli_bash.sh


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password: 


Wed Jan 30 02:43:20.540 UTC
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface Loopback0
 ipv4 address 50.1.1.1 255.255.255.255
!
grpc
 port 57777
 no-tls
 service-layer
 !
!
end

admin@devbox:ztp_hooks$ 

```







password for rtr1 is `vagrant`
{: .notice--info}

```
vagrant@devbox:ztp_cli_automation$ 
vagrant@devbox:ztp_cli_automation$ ssh vagrant@20.1.1.10 run /misc/scratch/automate_cli_bash.sh
Password: 


Tue Jun 12 04:25:37.278 UTC
Building configuration...
!! IOS XR Configuration version = 6.2.25
netconf-yang agent
 ssh
!
end

vagrant@devbox:ztp_cli_automation$ 
```


Excellent, we just configured netconf-yang agent in rtr1 to run over ssh as a subsystem and open up port 830 for subsequent netconf connections.
{: .notice--success}



## Python ZTP script

This script is only for learning purposes and doesn't directly affect the rest of the lab. View the python script  and see how we leverage the return values provided by the `ztp_helper.py` library methods to easily automate using IOS-XR CLI.


### Transfer the python script to rtr1 over the connected link

password for rtr1 is `vagrant`
{: .notice--info}  

```
vagrant@devbox:ztp_cli_automation$ scp automate_cli_python.py vagrant@20.1.1.10:/misc/scratch/
Password: 
automate_cli_python.py                          100%  444     0.4KB/s   00:00    
Connection to 20.1.1.10 closed by remote host.
vagrant@devbox:ztp_cli_automation$ 
```


### Execute the copied python script over ssh 

password for rtr1 is `vagrant`
{: .notice--info}   

```
vagrant@devbox:ztp_cli_automation$ ssh vagrant@20.1.1.10 run /misc/scratch/automate_cli_python.py
Password: 


Tue Jun 12 04:30:16.883 UTC

###### Debugs enabled ######


###### Using Child class method, creating a new user ######

2018-06-12 04:30:18,300 - DebugZTPLogger - DEBUG - Config File content to be applied  !
                     username vagrant2 
                     group root-lr
                     group cisco-support
                     secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1 
                     !
                     end
2018-06-12 04:30:24,558 - DebugZTPLogger - DEBUG - Received exec command request: "show configuration commit changes last 1"
2018-06-12 04:30:24,558 - DebugZTPLogger - DEBUG - Response to any expected prompt ""
Building configuration...
2018-06-12 04:30:26,632 - DebugZTPLogger - DEBUG - Exec command output is ['!! IOS XR Configuration version = 6.2.25', 'username vagrant2', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']
2018-06-12 04:30:26,632 - DebugZTPLogger - DEBUG - Config apply through file successful, last change = ['!! IOS XR Configuration version = 6.2.25', 'username vagrant2', 'group root-lr', 'group cisco-support', 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1', '!', 'end']

###### New user successfully created, return value: ######

['!! IOS XR Configuration version = 6.2.25',
 'username vagrant2',
 'group root-lr',
 'group cisco-support',
 'secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1',
 '!',
 'end']

###### return value in json: ######

[
    "!! IOS XR Configuration version = 6.2.25", 
    "username vagrant2", 
    "group root-lr", 
    "group cisco-support", 
    "secret 5 $1$FzMk$Y5G3Cv0H./q0fG.LGyIJS1", 
    "!", 
    "end"
]

###### Debugs Disabled  ######


###### Applying an incorrect config  ######


###### Failed to apply configuration, error is:######

['!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to',
 '!! one or more of the following reasons:',
 '!!  - the entered commands do not exist,',
 '!!  - the entered commands have errors in their syntax,',
 '!!  - the software packages containing the commands are not active,',
 '!!  - the current user is not a member of a task-group that has',
 '!!    permissions to use the commands.',
 'hostnaime rtr1']

###### error in json: ######

[
    "!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to", 
    "!! one or more of the following reasons:", 
    "!!  - the entered commands do not exist,", 
    "!!  - the entered commands have errors in their syntax,", 
    "!!  - the software packages containing the commands are not active,", 
    "!!  - the current user is not a member of a task-group that has", 
    "!!    permissions to use the commands.", 
    "hostnaime rtr1"
]

vagrant@devbox:ztp_cli_automation$ 
```

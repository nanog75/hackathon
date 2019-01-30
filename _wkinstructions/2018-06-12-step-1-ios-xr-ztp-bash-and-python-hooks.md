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


>The vagrant environment consisting of two IOS-XRv9k instances must already be running. Verify >this by opening up a terminal and running the following commands:
>
>  ```
>  cd ~/topology
>  vagrant status
>
>  ```
> You should then see something like this:
>  ```
>  cisco@pod2:~/topology$ vagrant status
>  Current machine states:
>
>  rtr1                      running (virtualbox)
>  rtr2                      running (virtualbox)
>  devbox                    running (virtualbox)
>
>   This environment represents multiple VMs. The VMs are all listed
>   above with their current state. For more information about a specific
>   VM, run `vagrant status NAME`.
>   cisco@pod2:~/topology$ 
>  ```

{: .notice--info}


The topology we will be dealing with looks something like this:

![topology.png]({{site.baseurl}}/images/topology.png)


Ask the proctor for your credentials to access the pod. Click on the "VNC" option under Devnet_Pods/Podx where Podx is the pod assigned to you.
{: .notice--warning}


## Open up Sublime to view the code pieces we will be dealing with
![sublime.png]({{site.baseurl}}/images/sublime.png)

Keep this open to view the code at any point. Drop back into the terminal to continue.


## SSH into the devbox 

Drop into the devbox and cd into the `/vagrant/code` directory


You should see the following files:

```
cisco@pod2:~/topology$ vagrant ssh devbox

Welcome to Cisco YDK-Py sandbox (Ubuntu 16.04)

68 packages can be updated.
10 updates are security updates.


Last login: Sat Jun  9 16:09:52 2018 from 10.0.2.2

vagrant@devbox:~$ 
vagrant@devbox:~$ cd /vagrant/code
vagrant@devbox:code$ 
vagrant@devbox:code$ 
vagrant@devbox:code$ ls
advanced                          ncclient                ydk
bigmuddy-network-telemetry-proto  service-layer-objmodel  ztp_cli_automation
vagrant@devbox:code$ 
```

cd into the ztp_cli_automation directory and you should a couple of files we will deal with in this section


## Bash ZTP script
The bash script will utilize the ZTP bash hooks in IOS-XR to configure netconf-yang agent on rtr1


### Transfer the bash script to rtr1 over the connected link

password for rtr1 is `vagrant`
{: .notice--info}  

```
vagrant@devbox:ztp_cli_automation$ scp automate_cli_bash.sh vagrant@20.1.1.10:/misc/scratch/
Password: 
automate_cli_bash.sh                          100%  444     0.4KB/s   00:00    
Connection to 20.1.1.10 closed by remote host.
vagrant@devbox:ztp_cli_automation$ 
```




### Execute the copied bash script over ssh


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

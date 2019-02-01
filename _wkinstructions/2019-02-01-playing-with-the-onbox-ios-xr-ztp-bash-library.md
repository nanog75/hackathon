---
published: true
date: '2019-02-01 03:24 +0100'
title: Playing with the onbox IOS-XR ZTP bash and Python Library
author: Akshat Sharma
excerpt: >-
  Play around with the onbox ZTP hooks for bash to familiarize yourself with the
  available utilities.
tags:
  - iosxr
  - cisco
  - ztp
  - lab
---

{% include toc %}

# Playing with the onbox ZTP bash hooks

Time to play around with the ZTP Bash hooks! Let's try out a few use cases.
We'll choose router r1 as our test platform.

## Connect to router r1

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Username</b>: admin<br/><b>Password</b>: admin<br/><b>SSH port</b>: 2221<br/><b>IP</b>: 10.10.20.170
</p>  

```
Laptop-terminal:$ ssh -p 2221 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password:


RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```

## Show commands

The `xrcmd` utility is used for this purpose. Let's dump the running configuration.
To use this utility drop into the IOS-XR bash shell using the `bash` CLI and first source the `/pkg/bin/ztp_helper.sh` library.  



#### Drop into Bash

```

RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#bash
Sun Aug 19 21:43:45.615 UTC
[r1:~]$
[r1:~]$
```

#### Source the ztp_helper.sh library

```
[r1:~]$ source /pkg/bin/ztp_helper.sh
[r1:~]$
```

#### Execute the show command

```
[r1:~]$ xrcmd "show running-config"
Building configuration...
!! IOS XR Configuration version = 6.4.1
!! Last configuration change at Sun Aug 19 21:44:14 2018 by ZTP
!
hostname r1
banner motd ;
--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------
;
service timestamps log datetime msec
service timestamps debug datetime msec
username admin
 group root-lr
 group cisco-support
 secret 5 $1$A4C9$oaNorr6BXDruE4gDd086L.
!
line console
 timestamp disable
 exec-timeout 0 0
!
vty-pool default 0 4 line-template VTY-TEMPLATE
call-home
 service active
 contact smart-licensing
 profile CiscoTAC-1
  active
  destination transport-method http
 !
!
interface MgmtEth0/RP0/CPU0/0
 description *** MANAGEMENT INTERFACE ***
 ipv4 address dhcp
!
interface GigabitEthernet0/0/0/0
 shutdown
!
interface GigabitEthernet0/0/0/1
 shutdown
!
interface GigabitEthernet0/0/0/2
 shutdown
!
interface GigabitEthernet0/0/0/3
 shutdown
!
interface GigabitEthernet0/0/0/4
 shutdown
!
router static
 address-family ipv4 unicast
  0.0.0.0/0 192.168.122.1
 !
!
netconf-yang agent
 ssh
!
ssh server v2
ssh server netconf vrf default
end

[r1:~]$


```



### Checking an invalid show command

The return value for xrcmd will unfortunately always be 0, so it is incumbent on us to parse the output and check for the string `% Invalid input detected at '^' marker.` which is dependable. The python library that the next learning lab in this module will showcase, already does this and returns a proper error code.


```
[r1:~]$
[r1:~]$ xrcmd "Hello World"
showtech_helper error: Parsing of command "Hello World" failed
Hello World
 ^
% Invalid input detected at '^' marker.
[r1:~]$

```

### Running an exec command

Exec commands are essentially run in the exec mode and they can affect the state of the system unlike show commands that are read-only actions.
For example, let's say we want to clear the logs on the system. We can utilize `xrcmd` to do so. But in addition, exec commands usually offer **prompts** that must be handled as part of the xrcmd call. We do so by taking advantage of some bash capabilities as shown below:  

#### Dump the initial set of logs

Your logs at this stage could look completely different but dump the last 10 lines of the current logs to verify that the clearing process works:

```
[r1:~]$ xrcmd "show logging" |  tail -10
RP/0/RP0/CPU0:Aug 20 01:39:10.662 UTC: devc-vty[298]: The specified TTY (4, 8) is not registered in the DB
RP/0/RP0/CPU0:Aug 20 01:39:10.704 UTC: devc-vty[298]: The specified TTY (4, 8) is not registered in the DB
RP/0/RP0/CPU0:Aug 20 01:39:23.505 UTC: nvgen[65561]: %MGBL-CONFIG_HIST_UPDATE-3-SYSDB_GET : Error 'sysdb' detected the 'warning' condition 'A verifier or EDM callback function returned: 'not found'' getting host address from  sysdb
RP/0/RP0/CPU0:Aug 20 01:39:23.660 UTC: devc-vty[298]: The specified TTY (4, 8) is not registered in the DB
RP/0/RP0/CPU0:Aug 20 01:39:23.676 UTC: devc-vty[298]: The specified TTY (4, 8) is not registered in the DB
RP/0/RP0/CPU0:Aug 20 01:39:23.703 UTC: devc-vty[298]: The specified TTY (4, 8) is not registered in the DB
RP/0/RP0/CPU0:Aug 20 01:49:21.506 UTC: SSHD_[68896]: %SECURITY-SSHD-6-INFO_USER_LOGOUT : User 'admin' from '192.168.122.1' logged out on 'vty0'
RP/0/RP0/CPU0:Aug 20 01:57:56.782 UTC: SSHD_[66208]: %SECURITY-SSHD-6-INFO_SUCCESS : Successfully authenticated user 'admin' from '192.168.122.1' on 'vty0'(cipher 'aes128-ctr', mac 'hmac-sha2-256')
RP/0/RP0/CPU0:Aug 20 02:08:17.238 UTC: SSHD_[66208]: %SECURITY-SSHD-6-INFO_USER_LOGOUT : User 'admin' from '192.168.122.1' logged out on 'vty0'
RP/0/RP0/CPU0:Aug 20 02:13:26.371 UTC: SSHD_[66446]: %SECURITY-SSHD-6-INFO_SUCCESS : Successfully authenticated user 'admin' from '192.168.122.1' on 'vty0'(cipher 'aes128-ctr', mac 'hmac-sha2-256')
[r1:~]$

```

#### Clear the logs using xrcmd

`clear logging` as an exec command prompts the user like so:

```
RP/0/RP0/CPU0:r1#clear logging
Mon Aug 20 01:58:14.572 UTC
Clear logging buffer [confirm] [y/n] :
```

So to automate against this, simply echo out "y" in response to the xrcmd output. To do that, we can use:

```
[r1:~]$
[r1:~]$
[r1:~]$ echo -ne "y\n" | xrcmd "clear logging"
Clear logging buffer [confirm] [y/n] :[r1:~]$
[r1:~]$

```

Great! Now checking the output of "show logging" again:

```
[r1:~]$ xrcmd "show logging"
Syslog logging: enabled (0 messages dropped, 0 flushes, 0 overruns)
    Console logging: level warnings, 39 messages logged
    Monitor logging: level debugging, 0 messages logged
    Trap logging: level informational, 0 messages logged
    Buffer logging: level debugging, 0 messages logged

Log Buffer (2097152 bytes):

[r1:~]$
[r1:~]$

```

### Running admin commands

In IOS-XR the admin mode is a privileged mode that allows you to run certain administrative exec commands like reloading or shutting down the box, running privileged show commands like "admin show environment power" etc.

To run these commands we again utilize `xrcmd` and the power of `echo` to funnel in the required admin show/exec commands to the admin mode.
Since we're on a virtual router, we can't really get environment dumps, so let's issue a simple "show platform" on the admin cli.



<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Running admin commands require a root-lr user to be configured on the router and an environment variable AAA_USER is used along with ZTP bash hooks to enable privilege associated with the root-lr user to gain access to the admin mode. Since the router has `admin` configured as the root-lr user, we set `AAA_USER=admin`
</p>  

```
[r1:~]$
[r1:~]$ export AAA_USER=admin && echo -ne "show platform\n" | xrcmd "admin"

ztp-user connected from 127.0.0.1 using console on r1
sysadmin-vm:0_RP0# show platform
Mon Aug  20 02:29:28.260 UTC
Location  Card Type               HW State      SW State      Config State  
----------------------------------------------------------------------------
0/0       R-IOSXRV9000-LC-C       OPERATIONAL   N/A           NSHUT         
0/RP0     R-IOSXRV9000-RP-C       OPERATIONAL   OPERATIONAL   NSHUT         

sysadmin-vm:0_RP0# [r1:~]$

```

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #eff9ef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Perfect! Any admin command you would potentially want to perform (even configuration in the admin shell) can be performed using the above method - reloads, reload to ipxe, change the state of the LEDs on the box, etc. Just use a combination of `\n` to separate out individual lines meant for the admin CLI. We can now proceed with the configuration manipulation hooks.</p>



## Configuration Merge

Earlier, we listed 4 different utilities that allow you to play around with configuration merge on IOS-XR:    

```
xrapply
xrapply_with_reason
xrapply_string
xrapply_string_with_reason
```

We will just try `xrapply` and `xrapply_string_with_reason` as a quick showcase. It is an exercise for the reader to try out the other utilities outside the context of this lab. We've already seen the existing configuration, so let's use the config merge utilities one by one to bring up GigabitEthernet Interfaces on r1.


<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Important</b>: We expect the user to perform the same exact steps on r2 before we head to the next section! Remember interfaces on both the routers should be up before we try to bring up protocols on the box using a bash script we will develop in the next section</p>


### Using xrapply to bring up GigabitEthernet0/0/0/0

`xrapply` uses a configuration file as an argument.
So let's create a file in the IOS-XR shell with the following content:

#### Create the local configuration file

```
[r1:~]$ cat > /root/gig0up.conf << EOF
> !
> interface GigabitEthernet0/0/0/0
>   ipv4 address 10.1.1.10/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
```
You could use `vi` for this purpose as well. In the end the file looks something like:

```
[r1:~]$ cat /root/gig0up.conf
!
interface GigabitEthernet0/0/0/0
  ipv4 address 10.1.1.10/24
  no shutdown
!
end
[r1:~]$
```

Now `xrapply` will add this configuration to the pre-existing configuration and will bring up `GigabitEthernet0/0/0/0` with the ip address `10.1.1.10/24`.

#### Source /pkg/bin/ztp_helper.sh

<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">You might not need to source it if you're working in the same shell as the xrcmd command earlier.</p>

```
[r1:~]$
[r1:~]$ source /pkg/bin/ztp_helper.sh
[r1:~]$
```

#### Do a configuration merge using xrapply


```
[r1:~]$ xrapply /root/gig0up.conf
[r1:~]$

```

#### Check the return value

```
[r1:~]$ echo $?
0
[r1:~]$
```
exit code 0 indicates that the configuration application was successful!

#### Verify the configuration was properly applied

We'll use the `xrcmd` utility to verify (exactly as you would in your own script):

```
[r1:~]$
[r1:~]$ xrcmd "show configuration commit changes last 1"
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface GigabitEthernet0/0/0/0
 ipv4 address 10.1.1.10 255.255.255.0
!
end

[r1:~]$

```
Also, verify that the interface is up as expected:

```

[r1:~]$ xrcmd "show ip int br"

Interface                      IP-Address      Status          Protocol Vrf-Name
MgmtEth0/RP0/CPU0/0            192.168.122.21  Up              Up       default
GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default
GigabitEthernet0/0/0/1         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/2         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/3         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/4         unassigned      Shutdown        Down     default
[r1:~]$
[r1:~]$
```



### Using xrapply with an invalid Configuration file

Just for kicks, let's see what happens if we create an invalid configuration file and try to use xrapply:

#### Create an invalid configuration file


```
[r1:~]$
[r1:~]$ cat > /root/invalid_config_file << EOF
> !
> interface invalid-interface-name
>   ipv4 address 1.1.1.2/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
```

#### Do a config merge using xrapply

```
[r1:~]$
[r1:~]$ xrapply /root/invalid_config_file
!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to
!! one or more of the following reasons:
!!  - the entered commands do not exist,
!!  - the entered commands have errors in their syntax,
!!  - the software packages containing the commands are not active,
!!  - the current user is not a member of a task-group that has
!!    permissions to use the commands.

interface invalid-interface-name
  ipv4 address 1.1.1.2/24
  no shutdown

[r1:~]$
```
Great! it throws up an error!


#### Check the return code

```
[r1:~]$
[r1:~]$ echo $?
1
[r1:~]$
```
This is very useful: by throwing up a distinct non-zero exit code upon failure to apply the configuration, it allows us to automate more deterministically.  


### Using xrapply_string_with_reason to configure GigabitEthernet0/0/0/1

The steps are illustrated below, think of it as a combination of the requirements for `xrapply_string` and `xrapply_with_reason`.

```
[r1:~]$
[r1:~]$ read -r -d '' gigup1_config << EOF
> !
> interface GigabitEthernet0/0/0/1
>   ipv4 address 11.1.1.10/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
[r1:~]$ echo "$gigup1_config"
!
interface GigabitEthernet0/0/0/3
  ipv4 address 11.1.1.10/24
  no shutdown
!
end
[r1:~]$
[r1:~]$ xrapply_string_with_reason "Testing xrapply_string_with_reason" "$gigup1_config"
[r1:~]$
[r1:~]$
[r1:~]$ xrcmd "show configuration commit list 1 detail"

   1) CommitId: 1000000024                 Label: NONE
      UserId:   ZTP                        Line:  ZTP
      Client:   CLI                        Time:  Mon Aug 20 00:47:24 2018
      Comment:  Testing xrapply_string_with_reason
[r1:~]$
[r1:~]$
[r1:~]$ xrcmd "show configuration commit changes last 1 "
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface GigabitEthernet0/0/0/1
 ipv4 address 11.1.1.10 255.255.255.0
 no shutdown
!
end

[r1:~]$
[r1:~]$ xrcmd "show ipv4 interface brief"

Interface                      IP-Address      Status          Protocol Vrf-Name
MgmtEth0/RP0/CPU0/0            192.168.122.21  Up              Up       default
GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default
GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default
GigabitEthernet0/0/0/2         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/3         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/4         unassigned      Shutdown        Down     default
[r1:~]$
[r1:~]$
```


## Configuration Replace



<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Very Very Important</b>: Configuration Replace can be potentially dangerous. If you make a mistake with the type of configuration you want to enforce, you can potentially lose connectivity to the router. So make sure you take precautions to ensure the final config is what you want at the end of the process</p>


We will not be attempting an xrreplace as part of this lab. But The basic workflow for using `xrreplace` is the same as `xrapply`. Provide a file containing the config to `xrreplace` and upon execution it will replace the entire configuration on the router with the configuration specified in the file.





# Playing with the Onbox ZTP Python hooks


Time to play around with the ZTP Python hooks! Let's try out a few use cases.
We'll choose router r1 as our test platform.
Follow instructions in the "Before you Begin" section to understand the SSH ports you have access to.

## Connect to router r1

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">**Username**: admin<br/>**Password**: admin<br/>**SSH port**: 2221<br/>**IP**: 10.10.20.170
</p>  

```
Laptop-terminal:$ ssh -p 2221 admin@10.10.20.170


--------------------------------------------------------------------------
  Router 1 (Cisco IOS XR Sandbox)
--------------------------------------------------------------------------


Password:


RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#

```

## Show commands

The `xrcmd()` utility is used for this purpose. Let's dump the running configuration.
To use this utility drop into the IOS-XR bash shell using the `bash` CLI and open up the python interpreter:  

```python
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#bash  
Sat Sep  8 18:53:22.730 UTC
[r1:~]$
[r1:~]$ python
Python 2.7.3 (default, Dec 12 2017, 08:22:03)
[GCC 4.9.1] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>

```

Now let's import the `ZtpHelpers` class from the  `/pkg/bin/ztp_helper.py` module. Multiple ways to do this, but we simply append the `sys.path` to include `/pkg/bin`.
Further, to use the utilities provided by the `ZtpHelpers` class we need to create an object of this class:

```python
>>>
>>> import sys
>>> sys.path.append("/pkg/bin")
>>> from ztp_helper import ZtpHelpers
>>>
>>> ztp_obj=ZtpHelpers()
>>>

```


#### Execute the show command

The `xrcmd` method in the `ZtpHelpers` class allows us to run show and exec commands in XR CLI. It takes a `cmd` parameter as input which is a dictionary of the structure:  
`{ 'exec_cmd': '', 'prompt_response': '' }`

`prompt_response` is an optional field meant for exec commands that require the script to answer prompts offered by the IOS-XR shell in response to `exec_cmd`.

Let's try and fetch the `running-config` from the router. We're still in the python interpreter on the router:

```python
>>> cmd={"exec_cmd" : "show running-config"}
>>> ztp_obj.xrcmd(cmd)
Building configuration...
{'status': 'success', 'output': ['!! IOS XR Configuration version = 6.4.1', '!! Last configuration change at Sat Sep  8 19:07:36 2018 by ZTP', '!', 'hostname r1', 'banner motd ;', '--------------------------------------------------------------------------', 'Router 1 (Cisco IOS XR Sandbox)', '--------------------------------------------------------------------------', ';', 'logging console debugging', 'service timestamps log datetime msec', 'service timestamps debug datetime msec', 'username admin', 'group root-lr', 'group cisco-support', 'secret 5 $1$A4C9$oaNorr6BXDruE4gDd086L.', '!', 'line console', 'timestamp disable', 'exec-timeout 0 0', '!', 'vty-pool default 0 4 line-template VTY-TEMPLATE', 'call-home', 'service active', 'contact smart-licensing', 'profile CiscoTAC-1', 'active', 'destination transport-method http', '!', '!', 'interface MgmtEth0/RP0/CPU0/0', 'description *** MANAGEMENT INTERFACE ***', 'ipv4 address dhcp', '!', 'interface GigabitEthernet0/0/0/0', 'ipv6 address 1010:1010::10/64', 'ipv6 enable', '!', 'interface GigabitEthernet0/0/0/1', 'ipv6 address 2020:2020::10/64', 'ipv6 enable', '!', 'interface GigabitEthernet0/0/0/2', 'shutdown', '!', 'interface GigabitEthernet0/0/0/3', 'shutdown', '!', 'interface GigabitEthernet0/0/0/4', 'shutdown', '!', 'router static', 'address-family ipv4 unicast', '0.0.0.0/0 10.0.2.2', '1.2.3.5/32 10.0.2.2', '!', '!', 'router bgp 65400', 'bgp router-id 11.1.1.10', 'address-family ipv4 unicast', 'network 11.1.1.0/24', '!', 'neighbor 11.1.1.20', 'remote-as 65450', 'address-family ipv4 unicast', 'next-hop-self', '!', '!', '!', 'grpc', 'port 57777', 'no-tls', 'service-layer', '!', '!', 'telemetry model-driven', 'destination-group DGroup1', 'address-family ipv4 192.168.122.11 port 5432', 'encoding self-describing-gpb', 'protocol tcp', '!', '!', 'sensor-group SGroup1', 'sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters', '!', 'sensor-group IPV6Neighbor', 'sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address', '!', 'subscription IPV6', 'sensor-group-id IPV6Neighbor sample-interval 15000', '!', 'subscription Sub1', 'sensor-group-id SGroup1 sample-interval 30000', 'destination-id DGroup1', '!', '!', 'netconf-yang agent', 'ssh', '!', 'ssh server v2', 'ssh server netconf vrf default', 'end']}
>>>
```

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #eff9ef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Great! The running-config was returned as a list of lines. Let's pretty print this configuration.
</p>  

```python
>>> from pprint import pprint
>>> pprint(ztp_obj.xrcmd(cmd))
Building configuration...
{'output': ['!! IOS XR Configuration version = 6.4.1',
            '!! Last configuration change at Sat Sep  8 19:07:36 2018 by ZTP',
            '!',
            'hostname r1',
            'banner motd ;',
            '--------------------------------------------------------------------------',
            'Router 1 (Cisco IOS XR Sandbox)',
            '--------------------------------------------------------------------------',
            ';',
            'logging console debugging',
            'service timestamps log datetime msec',
            'service timestamps debug datetime msec',
            'username admin',
            'group root-lr',
            'group cisco-support',
            'secret 5 $1$A4C9$oaNorr6BXDruE4gDd086L.',
            '!',
            'line console',
            'timestamp disable',
            'exec-timeout 0 0',
            '!',
            'vty-pool default 0 4 line-template VTY-TEMPLATE',
            'call-home',
            'service active',
            'contact smart-licensing',
            'profile CiscoTAC-1',
            'active',
            'destination transport-method http',
            '!',
            '!',
            'interface MgmtEth0/RP0/CPU0/0',
            'description *** MANAGEMENT INTERFACE ***',
            'ipv4 address dhcp',
            '!',
            'interface GigabitEthernet0/0/0/0',
            'ipv6 address 1010:1010::10/64',
            'ipv6 enable',
            '!',
            'interface GigabitEthernet0/0/0/1',
            'ipv6 address 2020:2020::10/64',
            'ipv6 enable',
            '!',
            'interface GigabitEthernet0/0/0/2',
            'shutdown',
            '!',
            'interface GigabitEthernet0/0/0/3',
            'shutdown',
            '!',
            'interface GigabitEthernet0/0/0/4',
            'shutdown',
            '!',
            'router static',
            'address-family ipv4 unicast',
            '0.0.0.0/0 10.0.2.2',
            '1.2.3.5/32 10.0.2.2',
            '!',
            '!',
            'router bgp 65400',
            'bgp router-id 11.1.1.10',
            'address-family ipv4 unicast',
            'network 11.1.1.0/24',
            '!',
            'neighbor 11.1.1.20',
            'remote-as 65450',
            'address-family ipv4 unicast',
            'next-hop-self',
            '!',
            '!',
            '!',
            'grpc',
            'port 57777',
            'no-tls',
            'service-layer',
            '!',
            '!',
            'telemetry model-driven',
            'destination-group DGroup1',
            'address-family ipv4 192.168.122.11 port 5432',
            'encoding self-describing-gpb',
            'protocol tcp',
            '!',
            '!',
            'sensor-group SGroup1',
            'sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters',
            '!',
            'sensor-group IPV6Neighbor',
            'sensor-path Cisco-IOS-XR-ipv6-nd-oper:ipv6-node-discovery/nodes/node/neighbor-interfaces/neighbor-interface/host-addresses/host-address',
            '!',
            'subscription IPV6',
            'sensor-group-id IPV6Neighbor sample-interval 15000',
            '!',
            'subscription Sub1',
            'sensor-group-id SGroup1 sample-interval 30000',
            'destination-id DGroup1',
            '!',
            '!',
            'netconf-yang agent',
            'ssh',
            '!',
            'ssh server v2',
            'ssh server netconf vrf default',
            'end'],
 'status': 'success'}
>>>
```

You can clearly see that the response  is a dictionary that contains two fields: `output` and `status`. Since the `exec_cmd` was successfull executed, `status`=`success` and `output`=`output of the exec/show command`.

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">
**The return values using the ztp python library utilities are very useful in deterministically writing CLI automation in python.**</p>


### Checking an invalid show command

Unlike the ZTP bash library, the ZTP python library does not put the onus of determining an invalid show command on the end user. Instead it returns `status`=`error` and `output`=`% Invalid input detected at '^' marker.`.

```python
>>> cmd={"exec_cmd" : "Hello World"}
>>> pprint(ztp_obj.xrcmd(cmd))
showtech_helper error: Parsing of command "Hello World" failed
{'output': ['Hello World', '^', "% Invalid input detected at '^' marker."],
 'status': 'error'}
>>>

```

### Running an exec command

Exec commands are essentially run in the exec mode and they can affect the state of the system unlike show commands that are read-only actions.
For example, let's say we want to clear the logs on the system. We can utilize `xrcmd` to do so. But in addition, exec commands usually offer **prompts** that must be handled as part of the xrcmd call. We do so by taking advantage of the `prompt_response` field in the dictionary passed into the `xrcmd()` method:

#### Dump the initial set of logs

Your logs at this stage could look completely different but dump the last 10 lines of the current logs to verify that the clearing process works (we do this using the python library as well):

```python
>>> cmd={"exec_cmd" : "show logging"}
>>> show_logging=ztp_obj.xrcmd(cmd)
>>>
>>> pprint(show_logging["status"])
'success'
>>>
>>> pprint(show_logging["output"][-10:])
["RP/0/RP0/CPU0:Sep  8 19:07:34.182 UTC: config[68961]: %MGBL-CONFIG-6-DB_COMMIT : Configuration committed by user 'ZTP'. Use 'show configuration commit changes 1000000175' to view the changes.",
 "RP/0/RP0/CPU0:Sep  8 19:07:35.559 UTC: exec[69085]: %SECURITY-LOGIN-6-AUTHEN_SUCCESS : Successfully authenticated user 'ztp-user' from '<unknown>' on 'vty9'",
 'RP/0/RP0/CPU0:Sep  8 19:07:35.562 UTC: devc-vty[298]: The specified TTY (4, 9) is not registered in the DB',
 'RP/0/RP0/CPU0:Sep  8 19:07:35.687 UTC: devc-vty[298]: The specified TTY (4, 9) is not registered in the DB',
 "RP/0/RP0/CPU0:Sep  8 19:07:35.692 UTC: exec[69085]: %SECURITY-LOGIN-6-CLOSE : User 'ztp-user' logged out",
 'RP/0/RP0/CPU0:Sep  8 19:07:37.094 UTC: syslog_dev[119]: locald_DLRSC[327] PID-16853: passwd: password expiry information changed.',
 'RP/0/RP0/CPU0:Sep  8 19:07:37.187 UTC: syslog_dev[119]: locald_DLRSC[327] PID-16877: Removing user ztp-user from group root-lr',
 "RP/0/RP0/CPU0:Sep  8 19:07:39.156 UTC: config[69121]: %MGBL-CONFIG-6-DB_COMMIT : Configuration committed by user 'ZTP'. Use 'show configuration commit changes 1000000176' to view the changes.",
 "RP/0/RP0/CPU0:Sep  8 19:08:03.417 UTC: SSHD_[68000]: %SECURITY-SSHD-6-INFO_USER_LOGOUT : User 'admin' from '192.168.122.1' logged out on 'vty0'",
 "RP/0/RP0/CPU0:Sep  8 19:08:15.515 UTC: SSHD_[69282]: %SECURITY-SSHD-6-INFO_SUCCESS : Successfully authenticated user 'admin' from '192.168.122.1' on 'vty0'(cipher 'aes128-ctr', mac 'hmac-sha2-256')"]
>>>

```


#### Clear the logs using xrcmd

`clear logging` as an exec command prompts the user like so:

```
RP/0/RP0/CPU0:r1#clear logging
Mon Aug 20 01:58:14.572 UTC
Clear logging buffer [confirm] [y/n] :
```

So to automate against this, simply provide "y\n" as a `prompt_response` field in the `xrcmd()` input dictionary:


```python
>>> cmd={"exec_cmd" : "clear logging", "prompt_response" : "y\n"}
>>> ztp_obj.xrcmd(cmd)
{'status': 'success', 'output': ['Clear logging buffer [confirm] [y/n] :']}
>>>
```

Great! Now checking the output of "show logging" again:

```python
>>> cmd={"exec_cmd" : "show logging"}
>>> show_logging=ztp_obj.xrcmd(cmd)
>>>
>>> pprint(show_logging["status"])
'success'
>>> pprint(show_logging["output"][-10:])
['Syslog logging: enabled (0 messages dropped, 0 flushes, 0 overruns)',
 'Console logging: level debugging, 9948 messages logged',
 'Monitor logging: level debugging, 9610 messages logged',
 'Trap logging: level informational, 0 messages logged',
 'Buffer logging: level debugging, 0 messages logged',
 'Log Buffer (2097152 bytes):']
>>>
```

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #eff9ef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">The log buffer has been cleared!</p>

### Running admin commands

In IOS-XR the admin mode is a privileged mode that allows you to run certain administrative exec commands like reloading or shutting down the box, running privileged show commands like "admin show environment power" etc.

To accomplish this we rope in the ZTP Bash library (get acquainted with ZTP Bash library through the first lab in this CLI Automation module) to execute the admin command using Python's Subprocess module.

Let us first define an `admincmd()`  utility of our own.

Drop into the `bash` shell of router `r1` again and create a `run_admin_cmd.py` file that contains the following code (you can use `vi` to accomplish this on the box itself):


```python
RP/0/RP0/CPU0:r1#
RP/0/RP0/CPU0:r1#bash
Sun Sep  9 00:12:24.569 UTC
[r1:~]$
[r1:~]$ cat run_admin_cmd.py
#!/usr/bin/env python

import subprocess

def admincmd(root_lr_user=None, cmd=None):

    if cmd is None:
        return {"status" : "error", "output" : "No command specified"}

    if root_lr_user is None:
        return {"status" : "error", "output" : "root-lr user not specified"}

    status = "success"

    # Set up the AAA_USER environment variable to set up
    # task map for admin cmd execution
    export_aaa_user="export AAA_USER="+root_lr_user

    # Source ztp_helper.sh ZTP Bash library to make sure that
    # the xrcmd bash command is available

    source_ztp_bash="source /pkg/bin/ztp_helper.sh"

    # Combine the two shell commands
    cmd_env_setup=export_aaa_user+ "&&" +source_ztp_bash


    # Set up use of xrcmd and echo to funnel in admin commands
    # to the admin shell of IOS-XR
    run_admin_cmd="echo -ne \""+cmd+"\\n \" | xrcmd \"admin\""

    #Set up the final bash command to run
    cmd = cmd_env_setup+ "&&" +run_admin_cmd


    # Utilize SubProcess to run the shell command
    process = subprocess.Popen(cmd, stdout=subprocess.PIPE, shell=True)
    out, err = process.communicate()


    # Filter out invalid admin exec/show commands and construct list out of output

    if process.returncode:
        status = "error"
        output = "Failed to get command output"
    else:
        output_list = []
        output = ""

        for line in out.splitlines():
            fixed_line= line.replace("\n", " ").strip()
            output_list.append(fixed_line)
            if "syntax error: expecting" in fixed_line:
                status = "error"
            output = filter(None, output_list)    # Removing empty items

    # Return the result with status and output
    return {"status" : status, "output" : output}
[r1:~]$
```

Here, we have created our own python function to run an admincmd in IOS-XR. Walk through the code comments to understand the flow of the code in greater detail.

Now open up a python interpreter in the same directory as the `run_admin_cmd.py` file and import the `admincmd()` function we just created:

```python
[r1:~]$
[r1:~]$ python
Python 2.7.3 (default, Dec 12 2017, 08:22:03)
[GCC 4.9.1] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>
>>> from run_admin_cmd import admincmd
```
We are now ready to run some admin commands through the python interpreter shell.
We'll try to run a simple show command in the admin mode:

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Running admin commands require a root-lr user to be configured on the router and an environment variable AAA_USER is used along with ZTP bash hooks to enable privilege associated with the root-lr user to gain access to the admin mode. Since the router has `admin` configured as the root-lr user, we set `root_lr_user`=`admin`
</p>  

```python
[r1:~]$ python
Python 2.7.3 (default, Dec 12 2017, 08:22:03)
[GCC 4.9.1] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>
>>> from run_admin_cmd import admincmd
>>>
>>> result = admincmd(root_lr_user="admin", cmd="show platform")
>>> from pprint import pprint
>>> pprint(result)
{'output': ['ztp-user connected from 127.0.0.1 using console on r1',
            '\x1b[?7hsysadmin-vm:0_RP0#show platform',
            'Sun Sep  9  00:17:29.516 UTC',
            'Location  Card Type               HW State      SW State      Config State',
            '----------------------------------------------------------------------------',
            '0/0       R-IOSXRV9000-LC-C       OPERATIONAL   N/A           NSHUT',
            '0/RP0     R-IOSXRV9000-RP-C       OPERATIONAL   OPERATIONAL   NSHUT',
            'sysadmin-vm:0_RP0#'],
 'status': 'success'}
>>>

```


Perfect! we can now proceed with the configuration manipulation hooks.  



## Configuration Merge

There are 2 different utilities that allow you to play around with **configuration merge** on IOS-XR:    

```
xrapply()
xrapply_string()
```

Let's try each one of these out. We've already seen the existing configuration, so let's use the config merge utilities one by one to bring up four GigabitEthernet Interfaces on r1.


<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">**Important**: We expect the user to perform the same exact steps on r2 before we head to the next section! Remember interfaces on both the routers should be up before we try to bring up protocols on the box using a python script we will develop in the next section</p>


### Using xrapply() to bring up GigabitEthernet0/0/0/0

`xrapply` uses a configuration file as an argument.
So let's create a file in the IOS-XR shell with the following content:

#### Create the local configuration file

```python
[r1:~]$ python
Python 2.7.3 (default, Dec 12 2017, 08:22:03)
[GCC 4.9.1] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> file_path="/root/gig0up.conf"
>>>
>>> file_content="""
... !
... interface GigabitEthernet0/0/0/0
...   ipv4 address 10.1.1.10/24
...   no shutdown
... !
... end
... """
>>>
>>>with open(file_path, 'w') as fd:
...     fd.write(file_content)
...
>>>   
```

You could use `vi` for this purpose as well. In the end the file looks something like:

```
[r1:~]$ cat /root/gig0up.conf
!
interface GigabitEthernet0/0/0/0
  ipv4 address 10.1.1.10/24
  no shutdown
!
end
[r1:~]$
```

Now `xrapply()` will add this configuration to the pre-existing configuration and will bring up `GigabitEthernet0/0/0/0` with the ip address `100.1.1.10/24`.

#### Import  ZtpHelpers from /pkg/bin/ztp_helper.py

In the python interpreter:

```python
>>> sys.path.append("/pkg/bin")
>>> from ztp_helper import ZtpHelpers
>>> ztp_obj=ZtpHelpers()

```

#### Do a configuration merge using xrapply


```python
>>> response=ztp_obj.xrapply(filename=file_path)
Building configuration...
>>> response
{'status': 'success', 'output': ['!! IOS XR Configuration version = 6.4.1', 'interface GigabitEthernet0/0/0/0', 'ipv4 address 100.1.1.10 255.255.255.0', 'no shutdown', '!', 'end']}
>>>
```


#### Check the return value

```python
>>> from pprint import pprint
>>> pprint(response)
{'output': ['!! IOS XR Configuration version = 6.4.1',
            'interface GigabitEthernet0/0/0/0',
            'ipv4 address 100.1.1.10 255.255.255.0',
            'no shutdown',
            '!',
            'end'],
 'status': 'success'}
>>>

```

The configuration merge was successful (`status` in the response) and the `output` field contains the contents of the  last configuration merge.



### Using xrapply with an invalid Configuration file

Just for kicks, let's see what happens if we create an invalid configuration file and try to use xrapply:

#### Create an invalid configuration file

without leaving the python interpreter:

```python
>>>
>>> invalid_config_filepath="/root/invalid_config_file"
>>>
>>> file_content="""
... !
... interface invalid-interface-name
...   ipv4 address 1.1.1.2/24
...   no shutdown
... !
... end
... """
>>>
>>> with open(invalid_config_filepath, 'w') as fd:
...     fd.write(file_content)
...
>>>
```


#### Do a config merge using xrapply

```python
>>>
>>> response = ztp_obj.xrapply(filename=invalid_config_filepath)
>>>
>>> from pprint import pprint
>>>
>>> pprint(response)
{'output': ['!! SYNTAX/AUTHORIZATION ERRORS: This configuration failed due to',
            '!! one or more of the following reasons:',
            '!!  - the entered commands do not exist,',
            '!!  - the entered commands have errors in their syntax,',
            '!!  - the software packages containing the commands are not active,',
            '!!  - the current user is not a member of a task-group that has',
            '!!    permissions to use the commands.',
            'interface invalid-interface-name',
            'ipv4 address 1.1.1.2/24',
            'no shutdown'],
 'status': 'error'}
>>>
```

Great! it throws up an error as expected. With the help of the return value, we can write more deteministic code to address configuration errors in our scripts.





### Using xrapply_string() with the `reason` parameter to configure GigabitEthernet0/0/0/1

The steps are illustrated below. In this case the `reason` parameter is passed to the python utility as an option. 
The python interpreter we opened earlier continues to be utilized.

```
>>>
>>> gig1up_config="""
... !
... interface GigabitEthernet0/0/0/1
...   ipv4 address 11.1.1.10/24
...   no shutdown
... !
... end
... """
>>>
>>> print(gig1up_config)

!
interface GigabitEthernet0/0/0/3
  ipv4 address 11.1.1.10/24
  no shutdown
!
end

>>>
>>> import sys
>>> sys.path.append("/pkg/bin")
>>> from ztp_helper import ZtpHelpers
>>> ztp_obj=ZtpHelpers()
>>>
>>>
>>> reason_string = "Testing python xrapply_string() with the reason parameter"
>>>
>>> response = ztp_obj.xrapply_string(cmd=gig3up_config, reason=reason_string)                    
Building configuration...
>>>
>>>
>>> from pprint import pprint
>>>
>>> pprint(response)
{'output': ['!! IOS XR Configuration version = 6.4.1',
            'interface GigabitEthernet0/0/0/3',
            'ipv4 address 11.1.1.10 255.255.255.0',
            'no shutdown',
            '!',
            'end'],
 'status': 'success'}
>>>
>>>
>>> response = ztp_obj.xrcmd({"exec_cmd" : "show configuration commit list 1 detail"})
>>>
>>> pprint(response)
{'output': ['1) CommitId: 1000000184                 Label: NONE',
            'UserId:   ZTP                        Line:  ZTP',
            'Client:   CLI                        Time:  Sun Sep  9 02:16:46 2018',
            'Comment:  Testing python xrapply_string() with the reason parameter'],
 'status': 'success'}
>>>
>>>
```


## Configuration Replace


<p style="margin: 2em 0!important;padding: 0.85em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 0.85em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Very Very Important</b>: Configuration Replace can be potentially dangerous. If you make a mistake with the type of configuration you want to enforce, you can potentially lose connectivity to the router. So make sure you take precautions to ensure the final config is what you want at the end of the process</p>


We will not be attempting an xrreplace as part of this lab. But The basic workflow for using `xrreplace` is the same as `xrapply`. Provide a file containing the config to `xrreplace` and upon execution it will replace the entire configuration on the router with the configuration specified in the file.
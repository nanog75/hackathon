---
published: true
date: '2019-02-01 03:24 +0100'
title: Playing with the onbox IOS-XR ZTP bash Library
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

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Username</b>: admin<br/><b>Password</b>: admin<br/><b>SSH port</b>: 2221<br/><b>IP</b>: 10.10.20.170
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



<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">Running admin commands require a root-lr user to be configured on the router and an environment variable AAA_USER is used along with ZTP bash hooks to enable privilege associated with the root-lr user to gain access to the admin mode. Since the router has `admin` configured as the root-lr user, we set `AAA_USER=admin`
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

Perfect! Any admin command you would potentially want to perform (even configuration in the admin shell) can be performed using the above method - reloads, reload to ipxe, change the state of the LEDs on the box, etc. Just use a combination of `\n` to separate out individual lines meant for the admin CLI. We can now proceed with the configuration manipulation hooks.  
{: .notice-success}


## Configuration Merge

Earlier, we listed 4 different utilities that allow you to play around with configuration merge on IOS-XR:    

```
xrapply
xrapply_with_reason
xrapply_string
xrapply_string_with_reason
```

Let's try each one of these out. We've already seen the existing configuration, so let's use the config merge utilities one by one to bring up four GigabitEthernet Interfaces on r1.


<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Important</b>: We expect the user to perform the same exact steps on r2 before we head to the next section! Remember interfaces on both the routers should be up before we try to bring up protocols on the box using a bash script we will develop in the next section</p>


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

<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #e6f2f7;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);">You might not need to source it if you're working in the same shell as the xrcmd command earlier.</p>

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

### Using xrapply_with_reason to bring up GigabitEthernet0/0/0/1

`xrapply_with_reason` works in exactly the same way as `xrapply`, except it takes a **reason** (i.e. a string of your choice to identify the config merge in IOS-XR's internal database - SYSDB) as the first argument.

The steps remain the same as `xrapply`:

#### Create the configuration file for GigabitEthernet0/0/0/1

```
[r1:~]$
[r1:~]$ cat > /root/gig1up.conf << EOF
> !
> interface GigabitEthernet0/0/0/1
>   ipv4 address 11.1.1.10/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
[r1:~]$
[r1:~]$ cat /root/gig1up.conf
!
interface GigabitEthernet0/0/0/1
  ipv4 address 11.1.1.10/24
  no shutdown
!
end
[r1:~]$
```

#### Provide a "reason" when configuring using xrapply_with_reason

```
[r1:~]$
[r1:~]$ source /pkg/bin/ztp_helper.sh
[r1:~]$
[r1:~]$
[r1:~]$ xrapply_with_reason "Testing out xrapply_with_reason" /root/gig1up.conf
[r1:~]$
```

#### Check that the "reason" was committed to SYSDB
The last commit automatically has the id `1`. So fetch the last commit's details:

```
[r1:~]$ xrcmd "show configuration commit list 1 detail"

   1) CommitId: 1000000022                 Label: NONE
      UserId:   ZTP                        Line:  ZTP
      Client:   CLI                        Time:  Mon Aug 20 00:24:30 2018
      Comment:  Testing out xrapply_with_reason
[r1:~]$
```
Perfect!, the comment shows up as expected. Very useful for accounting purposes.


#### Verify the last commit was accurate

```
[r1:~]$ xrcmd "show configuration commit changes last 1"
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



### Using xrapply_string to configure GigabitEthernet0/0/0/2

`xrapply_string` also does a configuration merge but accepts a single string instead of a file. Useful when you're working with only variables in your scripts.

We can either create a multiline string as shown below:

```
[r1:~]$
[r1:~]$ read -r -d '' gig2up_config << EOF
> !
> interface GigabitEthernet0/0/0/2
>   ipv4 address 12.1.1.10/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
[r1:~]$ echo "$gig2up_config"
!
interface GigabitEthernet0/0/0/2
  ipv4 address 12.1.1.10/24
  no shutdown
!
end
[r1:~]$
```
Or a single line string with the appropriate carriage return and escape characters:

```
[r1:~]$ echo "\!\ninterface GigabitEthernet0/0/0/2\nipv4 address 12.1.1.10/24\nno shutdown\n\!\nend" > gig2up_config
[r1:~]$
[r1:~]$ echo "$gig2up_config"
!
interface GigabitEthernet0/0/0/2
  ipv4 address 12.1.1.10/24
  no shutdown
!
end
[r1:~]$
```
I think it's fair to say that multiline strings are more readable.

Once you have the variable available, pass it to `xrapply_string` and verify (as before) that the configuration goes through fine.

```
[r1:~]$
[r1:~]$ xrapply_string "$gig2up_config"
[r1:~]$
[r1:~]$
[r1:~]$ echo $?
0
[r1:~]$
[r1:~]$ xrcmd "show configuration commit changes last 1"
Building configuration...
!! IOS XR Configuration version = 6.4.1
interface GigabitEthernet0/0/0/2
 ipv4 address 12.1.1.10 255.255.255.0
 no shutdown
!
end

[r1:~]$ xrcmd "show ipv4 interface brief"

Interface                      IP-Address      Status          Protocol Vrf-Name
MgmtEth0/RP0/CPU0/0            192.168.122.21  Up              Up       default
GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default
GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default
GigabitEthernet0/0/0/2         12.1.1.10       Up              Up       default
GigabitEthernet0/0/0/3         unassigned      Shutdown        Down     default
GigabitEthernet0/0/0/4         unassigned      Shutdown        Down     default
[r1:~]$


```

### Using xrapply_string_with_reason to configure GigabitEthernet0/0/0/3

The steps are illustrated below, think of it as a combination of the requirements for `xrapply_string` and `xrapply_with_reason`.

```
[r1:~]$
[r1:~]$ read -r -d '' gigup3_config << EOF
> !
> interface GigabitEthernet0/0/0/3
>   ipv4 address 13.1.1.10/24
>   no shutdown
> !
> end
> EOF
[r1:~]$
[r1:~]$ echo "$gigup3_config"
!
interface GigabitEthernet0/0/0/3
  ipv4 address 13.1.1.10/24
  no shutdown
!
end
[r1:~]$
[r1:~]$ xrapply_string_with_reason "Testing xrapply_string_with_reason" "$gigup3_config"
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
interface GigabitEthernet0/0/0/3
 ipv4 address 13.1.1.10 255.255.255.0
 no shutdown
!
end

[r1:~]$
[r1:~]$ xrcmd "show ipv4 interface brief"

Interface                      IP-Address      Status          Protocol Vrf-Name
MgmtEth0/RP0/CPU0/0            192.168.122.21  Up              Up       default
GigabitEthernet0/0/0/0         10.1.1.10       Up              Up       default
GigabitEthernet0/0/0/1         11.1.1.10       Up              Up       default
GigabitEthernet0/0/0/2         12.1.1.10       Up              Up       default
GigabitEthernet0/0/0/3         13.1.1.10       Up              Up       default
GigabitEthernet0/0/0/4         unassigned      Shutdown        Down     default
[r1:~]$
[r1:~]$
```


## Configuration Replace


<p style="margin: 2em 0!important;padding: 1em;font-family: CiscoSans,Arial,Helvetica,sans-serif;font-size: 1em !important;text-indent: initial;background-color: #fdefef;border-radius: 5px;box-shadow: 0 1px 1px rgba(0,127,171,0.25);"><b>Very Very Important</b>: Configuration Replace can be potentially dangerous. If you make a mistake with the type of configuration you want to enforce, you can potentially lose connectivity to the router. So make sure you take precautions to ensure the final config is what you want at the end of the process</p>


We will not be attempting an xrreplace as part of this lab. But The basic workflow for using `xrreplace` is the same as `xrapply`. Provide a file containing the config to `xrreplace` and upon execution it will replace the entire configuration on the router with the configuration specified in the file. 


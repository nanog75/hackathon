---
published: true
date: '2019-02-01 03:18 -0400'
title: 'Step 1: Understanding ZTP Libraries on IOS-XR'
author: Akshat Sharma
excerpt: >-
  Understanding the basic ZTP infrastructure of IOS-XR and libraries available
  for CLI automation
tags:
  - iosxr
  - cisco
  - ztp
  - cleur2019
  - lab
---

{% include toc %} . 

# Introduction: Understanding ZTP and the ZTP bash and Python helper libraries

Zero Touch Provisioning(ZTP) is a device provisioning mechanism that allows network devices running IOS-XR to be powered-on and provisioned in a completely automated fashion. The high-level workflow for ZTP is as follows:

1. The network-device with an IOS-XR image installed is powered on.
2. Upon boot-up, the ZTP process runs if the device does not have a prior configuration.
3. The ZTP process triggers dhclient on the Management port (and with the upcoming IOS-XR 6.5.1 release, even on the production/data ports) to send out a DHCP request identifying itself using DHCP options:
    * DHCP(v4/v6) client-id=Serial Number,
    * DHCPv4 option 124: Vendor, Platform, Serial-Number
    * DHCPv6 option 16: Vendor, Platform, Serial-Number
4. The DHCP server identifies the device and responds with either an IOS-XR configuration file or a ZTP script as the filename option.
5. If the device receives a configuration file, it would simply apply the configuration and terminate the ZTP process.
6. If the device receives a script or binary executable, it will simply execute the script/binary in the default bash shell in a network namespace corresponding to the global/default VRF. This script can be used to configure the device and/or install IOS-XR packages, set up linux applications etc.

This workflow is depicted in the figure below:

![ztp_workflow.png]({{site.baseurl}}/images/ztp_workflow.png)


>The concepts behind IOS-XR ZTP and further details on its operationalization in your network are expanded upon in the great set of blogs and tutorials on <https://xrdocs.io>.   
In particular:
>  * [**Working with IOS-XR ZTP**](https://xrdocs.io/software-management/tutorials/2016-08-26-working-with-ztp/)
>  * [**IOS-XR ZTP: Learning through Packet Captures**](https://xrdocs.io/software-management/blogs/2017-09-21-ios-xr-ztp-learning-through-packet-captures/)




## The ZTP Helper Libraries

It is clear from the above workflow that the DHCP server can respond to the device with a script/binary as one of the options.
This script/binary is executed in the IOS-XR Bash shell and may be used to interact with IOS-XR CLI to configure, verify the configured state and even run exec commands based on the workflow that the operator chooses.

So it goes without saying that the IOS-XR Bash shell must offer utilties/APIs/hooks that can allow a downloaded script/binary to interact-with/automate the IOS-XR CLI.

These utilities are provided by the `ZTP helper libraries for Bash and Python` that are available for scripts running on-box in the IOS-XR Linux shell.

### ZTP Helper bash Libary

The ZTP helper bash library is simply a bash script that creates useful wrappers out of pre-existing IOS-XR CLI interaction binaries in the IOS-XR shell.

On the router, this helper script is located at `/pkg/bin/ztp_helper.sh`.
To use this library, any Bash script (or even python scripts utilizing bash calls) must import the `ztp_helper.sh` library.  Upon import, the following bash hooks become available:

|Bash Utility|Argument|Function|
|:-------------:|:-------------|:------------|
|`xrcmd`|`<exec or show command>`|<br/>Exec commands and show commands in XR CLI<br/><br/>|
|`xrapply`| `<local filename>`|<br/> **Configuration Merge.**<br/>Apply additional configuration using a file<br/><br/>|
|`xrapply_with_reason`| **Arg1**: `<reason>` <br/>**Arg2**: `<local filename>`<br/><img width=180/> |<br/> **Configuration Merge**<br/> Apply additional configuration using a file along with a **reason** <br/> <br/> P.S. **reason** shows up as comment in `show configuration commit list detail`<br/><br/>|
|`xrapply_string`|`<config string>`|<br/>**Configuration Merge.**<br/> Apply additional configuration using a single line string (carriage returns are affected using `\n`)<br/><br/>|
|`xrapply_string_with_reason`|**Arg1**: `<reason>`<br/>**Arg2**:`<config string>`|<br/>**Configuration Merge.**<br/>Apply additional configuration using a single line string (carriage returns are affected using `\n`)<br/> <br/> P.S. **reason** shows up as comment in `show configuration commit list detail`<br/><br/>|
|`xrreplace`|``<local filename>``|<br/>**Configuration Replace**.<br/>Replace existing configuration with the configuration contained in the filename specified as argument.<br/><br/>|




### ZTP Helper Python Libary

The ZTP helper Python library is simply a python script that creates useful wrappers out of pre-existing IOS-XR CLI interaction binaries in the IOS-XR shell.

On the router, this helper script is located at `/pkg/bin/ztp_helper.sh`.
To use this library, any Python script  must import the `ztp_helper.py` library.  Upon import, the following python hooks become available:

The ZTP python library defines a single Python class called `ZtpHelpers`. This class contains all the utility methods that are described below.


#### ZtpHelpers class Methods:   


<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Object Creation:  <pre><code>__init__()</code></pre></b></p></div></div>

**Purpose**: This method is invoked when the `ZtpHelpers` object is created.

All of the following parameters are optional. Python's default `syslog` capability is utilized for the setting below. When nothing is specified during object creation, then all logs are sent to a log rotated file `/tmp/ztp_python.log` (max size of 1MB)

<b><u>Input Parameters</u></b>

* `syslog_server`: IP address of reachable Syslog Server  
     *  <u>Parameter type</u>: `string`  


* `syslog_port`: Port for the reachable syslog server  
     *  <u>Parameter type</u>: `int`  


* `syslog_file`: Alternative or add-on file to store syslog
     *  <u>Parameter type</u>: `string`



<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Debug Logging:  <pre><code>toggle_debug()</code></pre></b></p></div></div>

**Purpose**: Used to Enable/disable verbose debug logging

<b><u>Input Parameters</u></b>

* `enable`: Enable/Disable flag
    * Parameter Type: `int`  


<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Show/Exec CLI commands:    <pre><code>xrcmd()</code></pre></b></p></div></div>

**Purpose**: Issue an IOS-XR show command or exec command and obtain the output.

<b><u>Input Parameters</u>

* `cmd`: Dictionary representing the XR exec cmd and response to potential prompts  
  <u>Parameter Type</u>: `dict`  
  These values are encoded in the dict as follows<br/>`{ 'exec_cmd': '', 'prompt_response': '' }`.   

  In the dictionary, `prompt_response` is an optional field meant for exec commands that require the script to answer prompts offered by the IOS-XR shell in response to `exec_cmd`.  

<b><u>Return Value</u></b>  

<u>Return Type</u>: `dict`  

Returns a dictionary with status and output in the format:  
`{ 'status': 'error/success', 'output': '' }`  

Here `status`=`error` if an invalid exec/show command is specified as input to XR CLI and `output` is the actual `show command output` or `exec command response` in case of success.  



<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Configuration Merge using a File: <pre><code>xrapply()</code></pre></b></p></div></div>

**Purpose**: Apply Configuration to XR using a local file on the router. This method does a **configuration merge**.  

<b><u>Input Parameters</u></b>  
* `filename`: Filepath for a local file containing valid IOS-XR CLI configuration  
  * <u>Parameter Type</u>: `string`  


* `reason`: Reason for the configuration commit. Will show up in the output of: `show configuration commit list <> detail`. **This parameter is optional.**  
  * <u>Parameter Type</u>: `string`  

<b><u>Return Value</u></b>  

<u>Return Type</u>: `dict`  

Dictionary specifying the effect of the config change:  
`{ 'status' : 'error/success', 'output': 'exec command based on status'}`    

* Here `status`=`error` if the configuration merge was unsuccessful and the corresponding `output` is the response of the show command = `show configuration failed`.    

* Similarly, `status`=`success` if the configuration merge is successful and the corresponding `output` is the response of `show configuration commit changes last 1`  


<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Configuration Merge using a String: <pre><code>xrapply_string()</code></pre></b></p></div></div>

**Purpose**: Apply Configuration to XR using a string. This method does a **configuration merge**.  

<b><u>Input Parameters</u></b>  
* `cmd`: Single line or Multi-Line string that contains valid IOS-XR CLI configuration.  
 * <u>Parameter Type</u>: `string`    


* `reason`: Reason for the configuration commit. Will show up in the output of: `show configuration commit list <> detail`.**This parameter is optional.**  
  * <u>Parameter Type</u>: `string`  

<b><u>Return Value</u></b  

<u>Return Type</u>: `dict`  

Dictionary specifying the effect of the config change:  
`{ 'status' : 'error/success', 'output': 'exec command based on status'}`  

* Here `status`=`error` if the configuration merge was unsuccessful and the corresponding `output` is the response of `show configuration failed`.    

* Similarly, `status`=`success` if the configuration merge is successful and the corresponding `output` is the response of `show configuration commit changes last 1`  


<div class="notice--primary" style="background-color: #bac5de; font-size: 1.1em !important; margin: 2em 0 !important; padding: 1.5em 1em 1em 1em;text-indent: initial; border-radius: 5px; box-shadow: 0 1px 1px rgba(88,88,91,0.25) "><div class="text-center"><p><b>Configuration Replace using a file: <pre><code>xrreplace()</code></pre></b></p></div></div>

**Purpose**: Completely Replace existing Router configuration with the configuration specified in a file.

<b><u>Input Parameters</u></b>

* `filename`: Filepath for a local file containing valid IOS-XR CLI configuration  
  * <u>Parameter Type</u>: `string`

<b><u>Return Value</u></b><br/>  

<u>Return Type</u>: `dict`  

Dictionary specifying the effect of the config change:  

`{ 'status' : 'error/success', 'output': 'exec command based on status'}`  

* Here `status`=`error` if the configuration merge was unsuccessful and the corresponding `output` is the response of `show configuration failed`.  

* Similarly, `status`=`success` if the configuration merge is successful and the corresponding `output` is the response of `show configuration commit changes last 1`.

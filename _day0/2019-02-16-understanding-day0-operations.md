---
published: true
date: '2019-02-16 23:11 -0800'
title: Understanding Day0 Operations
author: Nanog75 Hackathon Team
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
excerpt: >-
  Understand the purpose of Day0 operations in the hackathon and the various
  components and workflows involved.
---


{% include toc %}
{% include base_path %}

## Before we Begin

Make sure you have access to a Pod based on the pod number assigned to you.
The instructions to connect to your pod can be found here: 

>[Connect to your pod]({{ base_path }}/assets/NANOG75_Hackathon_Lab_Info.pdf)


The topology for the hack is shown below: 

![]({{ base_path }}/images/topology_nanog75.png)  


{% capture "connect_text" %}
### Connecting to the nodes in the topology:
You will need the tesuto private key to ssh to the instances in your topology.  

This key can be downloaded from here:    
><https://storage.googleapis.com/tesuto-public/nanog75.key>  

Download and save the key in your local machine. This document assumes you saved it to ~/nanog75.key 

```
curl -o ~/nanog75.key https://storage.googleapis.com/tesuto-public/nanog75.key 
```

Change the permission of the key file before using it:

```
chmod 400 ~/nanog75.key
```


Assuming the pod number assigned to you is `x`, the FQDN to access each of the nodes in the topology is: 

> 1. **JumpHost**:     hackathon.pod`x`.cloud.tesuto.com
> 2. **ztp**:          ztp.hackathon.pod`x`.cloud.tesuto.com
> 3. **dev1**:         dev1.hackathon.pod`x`.cloud.tesuto.com
> 4. **dev2**:         dev2.hackathon.pod`x`.cloud.tesuto.com

Using the key downloaded, ssh to the instances like so (for example for the ZTP node):

```
ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com

```
{% endcapture %}


<div class="notice--info">
  {{ connect_text | markdownify }}
 </div>




## Connect to the ZTP box


For the Day0 group, all of the operations will be performed on the ZTP box.

Based on the pod number (assuming `x`), connect to your ZTP box first:


```
AKSHSHAR-M-33WP:~ akshshar$ ssh -i ~/nanog75.key tesuto@ztp.hackathon.podx.cloud.tesuto.com
Warning: the ECDSA host key for 'ztp.hackathon.podx.cloud.tesuto.com' differs from the key for the IP address '35.197.94.94'
Offending key for IP in /Users/akshshar/.ssh/known_hosts:1
Matching host key in /Users/akshshar/.ssh/known_hosts:5
Are you sure you want to continue connecting (yes/no)? yes
Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Feb 17 07:51:27 UTC 2019

  System load:  0.0               Processes:           108
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


Last login: Sun Feb 17 01:02:43 2019 from 128.107.241.176
tesuto@ztp:~$ 
tesuto@ztp:~$ 


```


## The Goal

The goal of the Day0 group is to make all the routers perform ZTP to attain a base configuration required for subsequent operations.


**Important**: ZTP is an impactful operation and can affect other users in the topology. Therefore, to start off, only `rtr2` has been earmarked for exclusive use by the Day0 group.
**DO NOT** perform ZTP on the other routers in your topology until you have a working ZTP script.
{: .notice--danger}

## The Basics

### What is ZTP?

ZTP stands for Zero touch provisioning and involves bringing up a router from scratch (without any configuration) by providing a ZTP script or a CLI configuration file to the router as part of the  DHCP offer in response to the router's request.
This process is shown below:

![ztp_workflow.png]({{site.baseurl}}/images/ztp_workflow.png)


1. When the router receives a CLI configuration file, it will replace its existing configuration with the configuration it downloads as part of the ZTP process.  
2. When the router receives a script (bash or python), it will execute the downloaded script which is responsible for applying a config to the router or perform any linux operation on the system.


## Understanding the Setup

Let us focus only on router `rtr2` for now. We intend to bootstrap it to a required configuration.


### DHCP setup

To respond to the router with the required file (script or config), the DHCP server must be suitably setup to identify the requesting router based on its Serial Number before responding with a bootfile-name (Option 67) specifying the file location.

The DHCP server expects the serial number to be provided by the IOS-XR router in Option 61 (client-identifier) as well as in option 124 (which contains the Serial Number, platform family and vendor code - 9 for Cisco). These are options for DHCPv4.

**You are NOT required to set up the DHCP server**. This is already done for you.
You will only modify some settings in the existing file as we go along.
Each router in the topology has a unique, persistent Serial-Number as well as a unique IP assigned to that Serial Number.

As the file below shows, the following IP addresses are assigned based on the Serial Numbers:

```
rtr1:  
    IPv4 Address:   100.96.0.14
    Serial Number:  FGE00080000
    
rtr2:
    IPv4 Address:   100.96.0.16
    Serial Number:  FGE000e0000
    
rtr3:
    IPv4 Address:   100.96.0.18
    Serial Number:  FGE00140000
    
rtr4:
    IPv4 Address:   100.96.0.26
    Serial Number:  FGE002c0000
```


To understand how, on the ZTP node, dump the `/etc/dhcp/dhcpd.conf` file:

```

tesuto@ztp:~$ 
tesuto@ztp:~$ cat /etc/dhcp/dhcpd.conf 
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#
# Attention: If /etc/ltsp/dhcpd.conf exists, that will be used as
# configuration file instead of this file.
#

# option definitions common to all supported networks...
option domain-name "cisco.local";

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

log-facility local7;

# DHCP Pools
#################################
# localpool
#################################


option space cisco-vendor-id-vendor-class code width 1 length width 1;
option vendor-class.cisco-vendor-id-vendor-class code 9 = {string};

######### Network 100.96.0.0/12 ################
shared-network 100-96-0-0 {


####### Pools ##############

  subnet 100.96.0.0 netmask 255.240.0.0 {
      option subnet-mask 255.240.0.0;
      option broadcast-address 100.111.255.255;
      option routers 1;
      option domain-name-servers 8.8.8.8;
  }


log (info, substring(option dhcp-client-identifier,0,11));
######## Matching Classes ##########


        class "xrv9k-rtr1" {
            match if (substring(option dhcp-client-identifier,0,11) = "FGE00080000");
        }

        pool {
            allow members of "xrv9k-rtr1";
            range 100.96.0.14 100.96.0.14;
            next-server 100.96.0.20;

            log (info, option vendor-class.cisco-vendor-id-vendor-class);
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,3,11));
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,19,99));

            if exists user-class and option user-class = "exr-config" {
              if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE00080000")
              {
                if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="R-IOSXRV9000-CC")
                {
                    #option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py";
                    option bootfile-name "http://100.96.0.20/configs/rtr1.config";
                }
              }
            }

            ddns-hostname "xrv9k-rtr1";
            option routers 100.96.0.1;
        }


        class "xrv9k-rtr2" {
            match if (substring(option dhcp-client-identifier,0,11) = "FGE000e0000");
        }

        pool {
            allow members of "xrv9k-rtr2";
            range 100.96.0.16 100.96.0.16;
            next-server 100.96.0.20;

            log (info, option vendor-class.cisco-vendor-id-vendor-class);
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,3,11));
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,19,99));

            if exists user-class and option user-class = "exr-config" {
              if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE000e0000")
              {
                if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="R-IOSXRV9000-CC")
                {
                    option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py";
                    #option bootfile-name "http://100.96.0.20/configs/rtr2.config";
                }
              }
            }

            ddns-hostname "xrv9k-rtr2";
            option routers 100.96.0.1;
        }



        class "xrv9k-rtr3" {
            match if (substring(option dhcp-client-identifier,0,11) = "FGE00140000");
        }

        pool {
            allow members of "xrv9k-rtr3";
            range 100.96.0.18 100.96.0.18;
            next-server 100.96.0.20;

            log (info, option vendor-class.cisco-vendor-id-vendor-class);
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,3,11));
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,19,99));

            if exists user-class and option user-class = "exr-config" {
              if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE00140000")
              {
                if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="R-IOSXRV9000-CC")
                {
                    #option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py";
                    option bootfile-name "http://100.96.0.20/configs/rtr3.config";
                }
              }
            }

            ddns-hostname "xrv9k-rtr3";
            option routers 100.96.0.1;
        }


        class "xrv9k-rtr4" {
          match if (substring(option dhcp-client-identifier,0,11) = "FGE002c0000");
        }

        pool {
            allow members of "xrv9k-rtr4";
            range 100.96.0.26 100.96.0.26;
            next-server 100.96.0.20;

            log (info, option vendor-class.cisco-vendor-id-vendor-class);
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,3,11));
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,19,99));

            if exists user-class and option user-class = "exr-config" {
              if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)="FGE002c0000")
              {
                if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="R-IOSXRV9000-CC")
                {
                    #option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py";
                    option bootfile-name "http://100.96.0.20/configs/rtr4.config";
                }
              }
            }

            ddns-hostname "xrv9k-rtr4";
            option routers 100.96.0.1;
        }

}
tesuto@ztp:~$ 


```






### Focus on rtr2

The DHCP settings relevant to rtr2 are highlighted below:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
        class "xrv9k-rtr2" {
            match if (substring(option dhcp-client-identifier,0,11) = <mark>"FGE000e0000");</mark>
        }

        pool {
            allow members of "xrv9k-rtr2";
            <mark>range 100.96.0.16 100.96.0.16;</mark>
            next-server 100.96.0.20;

            log (info, option vendor-class.cisco-vendor-id-vendor-class);
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,3,11));
            log (info, substring(option vendor-class.cisco-vendor-id-vendor-class,19,99));

            if exists user-class and option user-class = <mark>"exr-config" {</mark>
              if (substring(option vendor-class.cisco-vendor-id-vendor-class,3,11)=<mark>"FGE000e0000"</mark>)
              {
                if (substring(option vendor-class.cisco-vendor-id-vendor-class,19,99)="R-IOSXRV9000-CC")
                {
                    <mark>option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py";</mark>
                    <mark>#option bootfile-name "http://100.96.0.20/configs/rtr2.config";</mark>
                }
              }
            }

            ddns-hostname "xrv9k-rtr2";
            option routers 100.96.0.1;
        }


</code>
</pre>
</div>



Notice that based on the Serial Number = `FGE000e0000`, rtr2 is assigned the IP address `100.96.0.16`.  

Further, when the user-class (option 77) in the router's DHCP request is `exr-config`, then bootfile-name is currently set to be:

```
option bootfile-name "http://100.96.0.20/scripts/ztp_ncclient.py"
```


This means that the script `ztp_ncclient.py` located on the web server at `http://100.96.0.20/scripts` will be specified to the router in option 67 causing the router `rtr2` to download and execute it.



### Understanding the current ZTP script


The current ZTP script for rtr2 can be found on the webserver running locally on the ZTP node at:

```
/var/www/html/scripts/ztp_ncclient.py.

```

Let's dump this script. The script has appropriate comments at the different points to help a reader understand its content. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
tesuto@ztp:~$ 
tesuto@ztp:~$ cat /var/www/html/scripts/ztp_ncclient.py 
#!/usr/bin/env python

import sys, os, warnings
import time
sys.path.append("/pkg/bin")
from ztp_helper import ZtpHelpers
warnings.simplefilter("ignore", DeprecationWarning)
from ncclient import manager
import logging, json, subprocess
from lxml import etree

from ctypes import cdll

libc = cdll.LoadLibrary('libc.so.6')
_setns = libc.setns
CLONE_NEWNET = 0x40000000

rootLogger = logging.getLogger('ncclient.transport.session')
rootLogger.setLevel(logging.DEBUG)
handler = logging.StreamHandler()
rootLogger.addHandler(handler)

WEB_SERVER_URL = "http://100.96.0.20/"
PIP_RPM_URL = WEB_SERVER_URL + "packages/python-pip-7.1.0-r0.0.core2_64.rpm"
SERVER_XML_URL = WEB_SERVER_URL+"xml/"
SYSLOG_SERVER = "100.96.0.20"
SYSLOG_PORT = "514"
HTTPS_PROXY = ""

# List of XML files to be downloaded and applied as configuration 
# through ncclient onbox. 
# Hint: Add to this list on the server and download and use with ncclient
# appropriately for each router to achieve your final configuration 
XML_FILE_LIST =  ['grpc_config.xml',
                  'hostname.xml',
                  'mpls_static.xml']

# Use this Serial Number map to figure out the router specific URL
# to use for downloads based on the local router's Serial Number
SERIAL_NO_MAP = {'FGE00080000': 'rtr1',
                 'FGE000e0000': 'rtr2',
                 'FGE00140000': 'rtr3',
                 'FGE002c0000': 'rtr4'}

def install_and_import(package):
    import importlib
    try:
        importlib.import_module(package)
    except ImportError:
        try:
          from pip import main as pipmain
        except:
          from pip._internal import main as pipmain
        pipmain(['install', '--proxy='+HTTPS_PROXY, package])
    finally:
        globals()[package] = importlib.import_module(package)

class ZtpFunctions(ZtpHelpers):
    def ncclient_connect(self, host, duration=120):
        """User defined method in Child Class
           Method to connect to the netconf agent in IOS-XR
           using ncclient and return the ncclient manager handle.
           Since the netconf agent takes time to initialize,
           this method retries on failure for max specified duration
           until the connection succeeds.
           :param host: The local loopback ip to connect to (setup
                        using netconf_init()
           :type cmd: str 
           :param duration: maximum amount of time till which the method
                            retries to achieve a successful connection
           :type duration: str

           :return: Returns ncclient.manager() object on success
                    and None on failure
           :rtype: object 
        """
        connected = False
        t_start = time.time()
        t_end = t_start + duration 
        while time.time() < t_end:
            try:
                ncclient_manager = manager.connect(host=host, 
                                           port=830, 
                                           username="ztp-user", 
                                           device_params={'name':'iosxr'}, 
                                           hostkey_verify=False,
                                           look_for_keys=False,
                                           key_filename="/root/.ssh/id_rsa",
                                           allow_agent=False)
                connected = True 
                break
            except Exception as e:
                self.syslogger.info("Failed to connect to netconf agent")
                self.syslogger.info(e)
                self.syslogger.info("Trying to connect again, time elapsed: "+str(time.time()-t_start))
                time.sleep(5)

        if connected:
            self.syslogger.info("ncclient session established!, time elapsed: "+str(time.time()-t_start))
            return ncclient_manager
        else:
            return None 


    def run_bash(self, cmd=None, vrf="global-vrf", pid=1):
        """User defined method in Child Class
           Wrapper method for basic subprocess.Popen to execute 
           bash commands on IOS-XR in specified vrf (or global-vrf 
           by default).
           :param cmd: bash command to be executed in XR linux shell. 
           :type cmd: str 
           
           :return: Return a dictionary with status and output
                    { 'status': '0 or non-zero', 
                      'output': 'output from bash cmd' }
           :rtype: dict
        """

        with open(self.get_netns_path(nsname=vrf,nspid=pid)) as fd:
            self.setns(fd, CLONE_NEWNET)

            if self.debug:
                self.logger.debug("bash cmd being run: "+cmd)
            if cmd is not None:
                process = subprocess.Popen(cmd, 
                                           stdout=subprocess.PIPE,
                                           stderr=subprocess.PIPE,
                                           shell=True)
                out, err = process.communicate()
                if self.debug:
                    self.logger.debug("output: "+out)
                    self.logger.debug("error: "+err)
            else:
                self.syslogger.info("No bash command provided")
                return {"status" : 1, "output" : "",
                        "error" : "No bash command provided"}

            status = process.returncode

            return {"status" : status, "output" : out, "error" : err}


    def get_serial_number(self):
        """User defined method in Child Class
           Method to fetch the serial number of the router
           that this script is running on. Can be useful in
           invoking router/device specific URLs to take specific
           action based on the router.
           :return: Returns Serial number of the device (also used
                    by ZTP DHCP requests) on success
                    Returns empty string on failure
           :rtype: str 
        """
        cmd = "dmidecode -s system-serial-number | grep -v -e \"^#\""        
        response = self.run_bash(cmd)
        if not response["status"]:
            self.syslogger.info("Successfully fetched Serial Number:")
            if self.debug:
                self.logger.debug(response["output"])
            return response["output"].strip()
        else:
            self.syslogger.info("Failed to fetch Serial Number:")
            if self.debug:
                self.logger.debug(response["output"])
                self.logger.debug(response["error"])
            return ""


if __name__ == '__main__':

    # ZtpFunctions is a child class that inherits capabilities from
    # the ZtpHelpers library native to the OS.
    # A couple of methods(see above) are added to ZtpFunctions on top 
    # of the base functionality.
    # The ZtpHelpers class and consequently the ZtpFunctions class
    # accepts syslog parameters for the __init__() method. 
    #
    # Using the class method ztp_script.syslogger.info(<string>)
    # will automatically send a formatted syslog to the specified
    # syslog server and file (all optional)

    ztp_script = ZtpFunctions(syslog_server=SYSLOG_SERVER,
                              syslog_port=SYSLOG_PORT,
                              syslog_file="/root/ztp_python.log")


    # Let's set up some packages we'll use later in the script

    # Before we fetch the python packages, install pip and 
    # upgrade it.
   
    cmd = "wget " + PIP_RPM_URL + " -O /tmp/pip.rpm"
    if not ztp_script.run_bash(cmd)["status"]:
        ztp_script.syslogger.info("Successfully downloaded python-pip rpm")
 
        cmd = "rpm -ivh --force /tmp/pip.rpm"
        if not ztp_script.run_bash(cmd)["status"]:
            ztp_script.syslogger.info("Successfully installed python-pip rpm")

            cmd = "pip --proxy="+HTTPS_PROXY+" install --upgrade pip"
            if not ztp_script.run_bash(cmd)["status"]:
                ztp_script.syslogger.info("Successfully upgraded pip")
            else:
                ztp_script.syslogger.info("Failed to upgrade pip")
                sys.exit(1)
        else:
            ztp_script.syslogger.info("Failed to install pip")
            sys.exit(1)
    else:
        ztp_script.syslogger.info("Failed to download python-pip rpm")
        sys.exit(1) 
  


    
    # Install packages through pip and import them. Requires internet access
    # through the gateway in your ZTP LAN
    for package in ['xmltodict']:
        try:
          install_and_import(package)
          ztp_script.syslogger.info("Successfully installed and imported "+package)
        except:
          ztp_script.syslogger.info("Failed to install/import "+package)
          sys.exit(1)


    # Choose any ip you need for the host value
    # This will be set up as a local loopback(/32) that the onbox ncclient will connect 
    # to through the ncclient_init() method. 
    # It will be cleaned up as part of ncclient_cleanup() method
    host = "172.16.20.1"

   <mark> # Initialize the basic configuration for ncclient to work
    # Sets up local loopback as host to connect to and enable netconf on port 830.
    # Also imports the local key (/root/.ssh/id_rsa.pub) for password free operation
    result = ztp_script.ncclient_init(host_ip=host)
    if result["status"] == "error":
        ztp_script.syslogger.info("Failed to initialize configuration for ncclient, aborting....")

        # sys.exit() used judiciously is quite important. ZTP will retry if your script returns a
        # non zero exit code. In production, this ensures the box continuously looks to download and
        # execute a working script by retrying on failure.
        sys.exit(1)


    # Connects to IOS-XR netconf agent and returns a manager handle for ncclient
    # This is a method of the child class defined above that retries the connection
    # during initial boot for a specified maximum duration (default = 120seconds)
    nc_mgr = ztp_script.ncclient_connect(host=host)
   
    if nc_mgr is None:
        ztp_script.syslogger.info("ncclient Manager not initialized, aborting...")
        sys.exit(1)    
    else:
        # From here on, the operation is just normal ncclient usage
        response = nc_mgr.get_config(source="running")
        print(response)
        response_dict=xmltodict.parse(str(response))
        print json.dumps(response_dict, indent=4)

        nc_mgr.close_session()


    # Cleaning up the internal loopback configuration created as part of the
    # ncclient_init() configuration is good practice. This is optional however.
    result = ztp_script.ncclient_cleanup()
    if result["status"] == "error":
        ztp_script.syslogger.info("Failed to cleanup ncclient dependencies, aborting...")
        sys.exit(1)</mark>
tesuto@ztp:~$ 
</code>
</pre>
</div>



Notice the highlighted portion above. It shows 3 important steps:


1. Initialize the environment for ncclient:
   Using `ztp_script.ncclient_init(host_ip=host)`


2. Connect to the IOS-XR netconf server/agent with appropriate retries:

   ````# Connects to IOS-XR netconf agent and returns a manager handle for ncclient
    # This is a method of the child class defined above that retries the connection
    # during initial boot for a specified maximum duration (default = 120seconds)
    nc_mgr = ztp_script.ncclient_connect(host=host)```


3. Once the ncclient manager is available (nc_mgr), peform all the typical operations as you would with ncclient (<https://github.com/ncclient/ncclient>)

    ```
    # From here on, the operation is just normal ncclient usage
        response = nc_mgr.get_config(source="running")
        print(response)
        response_dict=xmltodict.parse(str(response))
        print json.dumps(response_dict, indent=4)
    ```




You job is to focus only on the above portion of code to add ncclient operations to `edit_config` using XML files that the script will download during its operation.
{: .notice--success}



To understand how to perform ZTP on the router rtr2, follow along to the [Step2]({{ base_path }})




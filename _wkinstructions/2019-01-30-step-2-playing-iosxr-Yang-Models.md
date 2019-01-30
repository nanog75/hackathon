---
published: true
date: '2018-06-12 09:51 -0400'
title: 'Step 2: Playing with IOS-XR YANG Models'
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - devnet
  - cleur2019
excerpt: >-
  Learn to use Ansible , YDK and Telemetry to solve several provisioning and
  monitoring tasks. We run some clients from written from scratch to help the
  reader grasp these new techniques.
---


{% include toc %}

## To Know More

There are several tools in the industry that allow you to play around with YANG models on IOS-XR and other Network OS stacks.

In this workshop we will look at a few of them:

* The Ansible `netconf_config` module to configure a BGP session on the routers. <https://docs.ansible.com/ansible/2.4/netconf_config_module.html>  

* Yang Development Kit (YDK) to configure Telemetry and interfaces on the routers: <http://ydk.io>
* Your own python Telemetry client which we will use to extract python data coming from YANG paths set up by YDK:
<https://learninglabs.cisco.com/tracks/iosxr-programmability/iosxr-streaming-telemetry/03-iosxr-02-telemetry-python/step/1>. Also, a lot more on Telemetry here: <https://xrdocs.io/telemetry/>

     
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

     
     
## Keep running the Telemetry client

Before we begin Step 3 of this workshop where we combine IOS-XR Telemetry, Ansible, Open/R and Service-Layer APIs in XR, make sure you have run through Steps 1 and 2 already so that you have the Telemetry client already listening to BGP Session updates from router r1.  

To recap, at the end of Step 2, you must have one terminal window connected to your devbox where you have a client receiving data from router r1's BGP Session (Currently in idle state). If not, please start it again as shown below. 



```
admin@devbox:python$ 
admin@devbox:python$ export SERVER_IP=10.10.20.170
admin@devbox:python$ export SERVER_PORT=57021
admin@devbox:python$ 
admin@devbox:python$ ./telemetry_client_json.py 
Using GRPC Server IP(10.10.20.170) Port(57021)
{
   "encoding_path": "Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/sessions/session", 
   "subscription_id_str": "1", 
   "collection_start_time": 1548827148102, 
   "msg_timestamp": 1548827148108, 
   "collection_end_time": 1548827148108, 
   "node_id_str": "r1", 
   "data_json": [
      {
         "keys": {
            "neighbor-address": "60.1.1.1", 
            "instance-name": "default"
         }, 
         "timestamp": 1548827148107, 
         "content": {
            "messages-queued-in": 0, 
            "is-local-address-configured": false, 
            "local-as": 65000, 
            "nsr-state": "bgp-nbr-nsr-st-none", 
            "description": "", 
            "connection-remote-address": {
               "ipv4-address": "60.1.1.1", 
               "afi": "ipv4"
            }, 
            "messages-queued-out": 0, 
            "connection-state": "bgp-st-idle", 
            "speaker-id": 0, 
            "vrf-name": "default", 
            "remote-as": 65000, 
            "postit-pending": false, 
            "nsr-enabled": true, 
            "connection-local-address": {
               "ipv4-address": "0.0.0.0", 
               "afi": "ipv4"
            }
         }
      }
   ], 
   "collection_id": 16
}


```


## Open/R Deployment

With the Telemetry client running, we can now progress to the next set of steps where we deploy Open/R as an application on both the routers.  

The basic deployment of Open/R we intend to achieve is represented below:

![openr-b2b.png]({{site.baseurl}}/images/openr-b2b.png)



### View the Open/R config files 

We will use Ansible to deploy Open/R as docker instances to the routers r1 and r2. In addition to the docker instances, Open/R requires some configuration files to be present (much like an ISIS or BGP config). We will utilize Ansible to push these files to the routers as well.


To view the relevant config files, **open a new shell** on the devbox with the telemetry client running in the earlier shell. 

Drop into the `ansible/openr` folder in the original `iosxr-devnet-cleur2019` git repository you cloned earlier: 


```
admin@devbox:~$ cd iosxr-devnet-cleur2019/
admin@devbox:iosxr-devnet-cleur2019$ ls
ansible  README.md  ydk  ztp_hooks
admin@devbox:iosxr-devnet-cleur2019$ 
admin@devbox:iosxr-devnet-cleur2019$ 
admin@devbox:iosxr-devnet-cleur2019$ cd ansible/openr/
admin@devbox:openr$ 
admin@devbox:openr$ ls
hosts_r1  increment_ipv4_prefix1.py  launch_openr_r1.sh  run_openr_r1.sh
hosts_r2  increment_ipv4_prefix2.py  launch_openr_r2.sh  run_openr_r2.sh
admin@devbox:openr$ 


```










---
published: true
date: '2018-06-12 10:13 -0400'
title: 'Step 3:  Combining Telemetry with Ansible and Open/R '
author: Akshat Sharma
tags:
  - iosxr
  - cisco
  - CLUS2018
  - devnet
  - programmability
excerpt: >-
  Combining IOS-XR Telemetry and Service layer APIs to solve a remediation use
  case
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

1. The `hosts_rtr*` files are used to set up `/etc/hosts` inside the docker container that we launch using Ansible. This setting is important for Open/R to work.

2. The `launch_openr_*.sh` scripts contain the actual docker run command which will launch the container and mount relevant config files into it.

3. The `run_openr_*.sh` script contains the configuration knobs for Open/R to know what routes to advertise to other Open/R instances on the network, the gRPC port for Service-Layer API to use while running locally on the router, the interfaces of XR that it should send its hellos out on, etc.

4. The `increment_ipv4_prefix*.py` script is an optional script we use to push a large set of routes into the configuration file for Open/R to advertise. 







### Ansible Playbook to deploy Open/R


The Ansible playbook we intend to use can be found in the `ansible` directory  of the `iosxr-devnet-cleur2019` git repository:  


```
admin@devbox:~$ 
admin@devbox:~$ cd ~/iosxr-devnet-cleur2019/
admin@devbox:iosxr-devnet-cleur2019$ cd ansible/
admin@devbox:ansible$ 
admin@devbox:ansible$ cat docker_bringup.yml 
---
- hosts: routers_shell
  gather_facts: no
  sudo: yes

  vars:
    connect_vars:
       host: "{{ ansible_host }}"
       username: "{{ ansible_user }}"
       password: "{{ ansible_ssh_pass }}"
    up: "sudo -i /misc/app_host/launch_openr_{{ inventory_hostname }}.sh"
    down: "sudo -i docker rm -f openr"

  tasks:

  - name: Copy run_openr script to rtr
    copy:
      src: "{{ run_openr_script }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 
  - name: Copy hosts_r file to rtr
    copy:
      src: "{{ hosts_r }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy launch_openr script to rtr
    copy:
      src: "{{ launch_openr_script }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy increment_ipv4 script to rtr
    copy:
      src: "{{ increment_ipv4_prefix }}"
      dest: "/misc/app_host/"
      owner: root 
      group: root 
      mode: a+x 

  - name: Copy cron file to rtr (CSCvh76067)
    copy:
      src: "{{ cron_file }}"
      dest: "/misc/app_host/ipv6_fe80_route_append.sh"
      owner: root
      group: root
      mode: a+x

  - name: Set up Cronjob (CSCvh76067)
    cron:
      name: set up ipv6 fe80::/64 routes 
      weekday: "*"
      minute: "*"
      hour: "*"
      user: root
      job: "/misc/app_host/ipv6_fe80_route_append.sh"
      cron_file: ansible_ipv6_fe80_route_append


  - name: Check docker container is running
    shell: sudo -i docker inspect --format={{ '{{.State.Running}}' }}  openr
    args:
      executable: /bin/bash
    register: status
    ignore_errors: yes
  - debug: var=output.stdout_lines

 
  - name: Clean up docker container if running 
    shell: "{{ down }}"
    args:
      executable: /bin/bash
    register: output
    when: status.stdout == "true"
  - debug: var=output.stdout_lines


  - name: Bring up the docker container 
    shell: "{{ up }}"
    args:
      executable: /bin/bash
    register: output
    ignore_errors: yes
  - debug: var=output.stdout_lines  
admin@devbox:ansible$ 
```

As can be seen above, the first 4 tasks of the playbook copy over the relevant config files and scripts for Open/R into `/misc/app_host` directory on the routers. This particular directory is mounted into the docker containers when we launch the container to make the config files available to Open/R inside the docker container. 

The 5th task is only required for this particular version of the IOS-XR software (6.4.1) we are using in the lab due to the bug: `CSCvh76067` which prevents `fe80::/64` routes from being installed for all interfaces in the kernel. This is fixed in subsequent releases of IOS-XR.
We transport a cron job onto the router to fix this issue for this lab.

The next 3 tasks are used to check if Open/R is already running, clean up if it is and then launch the docker container using the `/misc/app_host/launch_openr.sh` we transfered using task 3. 



**Since this is the first time you're running this Ansible playbook and the Open/R docker image is not yet on the routers, the run command will invoke a download of the docker image on each router. So expect the last task of the above playbook to take some time for the first run**.
{:. notice--danger}
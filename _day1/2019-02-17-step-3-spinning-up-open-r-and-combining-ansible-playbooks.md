---
published: true
date: '2019-02-17 04:27 -0800'
title: 'Step 3: Spinning up Open/R and combining Ansible Playbooks'
author: Nanog75 Hackathon Team
excerpt: >-
  Spin up Open/R to bring up the BGP sessions and combine playbooks into a
  single playbook to automate Day1 operations using a single step.
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---


{% include toc %}
{% include base_path %} 



If you've accomplished the tasks in Step2, then you're one step closer to finishing off Day1 operations.

At the end of Step2, you should have:


1.  IBGP configuration applied to rtr1 and rtr4

2.  A Telemetry collector running inside a docker container collecting data over gNMI from a router of your choice and pushing the data to a kafka bus.



**Note**: Even if you were not able to complete the steps in Step 2, you can still proceed with the steps outlined below and go back to solving the problem statements in Step 2.



### Check if BGP neighbors are up on rtr1 and rtr4


Now, if Step2 was successful, you should have the following configuration on rtr1 and rtr4 for BGP:


**rtr1 BGP configuration**:

```
router bgp 65000
 bgp router-id 172.16.1.1
 address-family ipv4 unicast
 !
 neighbor-group IBGP
  remote-as 65000
  local address 172.16.1.1
  address-family ipv4 unicast
  !
 !
 neighbor 172.16.4.1
  use neighbor-group IBGP
  address-family ipv4 unicast
  !
 !
!



```

**rtr2 BGP configuration**:

```
router bgp 65000
 bgp router-id 172.16.4.1
 address-family ipv4 unicast
 !
 neighbor-group IBGP
  remote-as 65000
  local address 172.16.4.1
  address-family ipv4 unicast
  !
 !
 neighbor 172.16.1.1
  remote-as 65000
  use neighbor-group IBGP
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!

```

Let's check the BGP session state on rtr1:


```
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show bgp summary 
Sun Feb 17 12:55:18.724 UTC
BGP router identifier 172.16.1.1, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 2
BGP main routing table version 2
BGP NSR Initial initsync version 2 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker               2          2          2          2           2           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
172.16.4.1        0 65000      81      82        0    0    0 00:00:09 Idle

RP/0/RP0/CPU0:rtr1#

```

It's idle! Well this should be obvious, we don't have an IGP running to distribute the loopbacks  (loopback 0) that are used for BGP session configuration.

You could run an IGP of your choice (isis, ospf) using YDK or ncclient scripts, but let's make things interesting by using  Open/R as an IGP instead.


## Spinning up Open/R as an IGP on the routers


Open/R (<https://github.com/facebook/openr>) is a routing protocol stack that was open-sourced by Facebook in late 2017.

It was designed to be easily integratable with lower level APIs provided by Vendor stacks and it was therefore straightforward to integrate with IOS-XR's Service-Layer APIs to allow Open/R to co-exist with standard IOS-XR protocols while running as an application on IOS-XR.

The integration code can be found here:

><https://github.com/akshshar/openr-xr>


Open/R runs inside a docker container on IOS-XR as shown below:


![openr-xr-deploy.png]({{site.baseurl}}/images/openr-xr-deploy.png)



## Spinning up the docker container using Ansible


We've already automated this process for you. If you jump into the `code-samples/ansible/playbooks/openr_bringup` directory and dump the set of files available:


```
tesuto@dev1:~$ 
tesuto@dev1:~$ cd ~/code-samples/ansible/playbooks/openr_bringup
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ 
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ tree .
.
├── automate_docker_setup.py
├── docker_bringup.yml
└── openr
    ├── hosts_rtr1
    ├── hosts_rtr2
    ├── hosts_rtr3
    ├── hosts_rtr4
    ├── increment_ipv4_prefix1.py
    ├── increment_ipv4_prefix2.py
    ├── launch_openr_rtr1.sh
    ├── launch_openr_rtr2.sh
    ├── launch_openr_rtr3.sh
    ├── launch_openr_rtr4.sh
    ├── run_openr_rtr1.sh
    ├── run_openr_rtr2.sh
    ├── run_openr_rtr3.sh
    └── run_openr_rtr4.sh

1 directory, 16 files
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ 
```

You'll see that there is an ansible playbook (docker_bringup.yml) along with a few scripts.
The `automate_docker_setup.py` script is a simple python script that restarts the docker daemon on the routers and pulls the `akshshar/openr-xr` docker image onto each router.
The remaining files under the `openr` directory are associated with the basic launch and configuration settings of open/R for each router.  


The playbook is dumped below:

```

tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ 
tesuto@dev1:~/code-samples/ansible/playbooks/openr_bringup$ cat docker_bringup.yml 
---
- hosts: routers_shell
  gather_facts: no
  become: yes

  vars:
    connect_vars:
       host: "{{ ansible_host }}"
       username: "{{ ansible_user }}"
       password: "{{ ansible_ssh_pass }}"
    up: "sudo -i /misc/app_host/launch_openr_{{ inventory_hostname }}.sh"
    down: "sudo -i docker rm -f openr"

  tasks:

  - name: restart docker daemon and pull openr image
    script: ./automate_docker_setup.py
    register: output

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
    ignore_errors: yes
  - debug: var=output.stdout_lines


  - name: Bring up the docker container 
    shell: "{{ up }}"
    args:
      executable: /bin/bash
    register: output
    ignore_errors: yes
  - debug: var=output.stdout_lines  

  - name: Pause the playbook to allow routes to be distributed before running the reachability playbook
    pause:
       minutes: 2

```

*  Thus,  the playbook first executes the `automate_docker_setup.py` script on each router thereby pulling the required docker image on to each router.  
*  Next it transfers all the required configuration files and scripts to the `/misc/app_host` directory on the routers from where it would be easy to mount these files into the final running container on each router.
*  In the end, it checks if Open/R is already running and if so, it removes it before spinning up a new one.  


### Run the Ansible playbook for Open/R


```
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ ansible-playbook -i ansible_hosts playbooks/openr_bringup/docker_bringup.yml 

PLAY [routers_shell] **********************************************************************************************************************************

TASK [restart docker daemon and pull openr image] *****************************************************************************************************
changed: [rtr1]
changed: [rtr3]
changed: [rtr4]

TASK [Copy run_openr script to rtr] *******************************************************************************************************************
ok: [rtr4]
ok: [rtr1]
ok: [rtr3]

TASK [Copy hosts_r file to rtr] ***********************************************************************************************************************
ok: [rtr1]
ok: [rtr4]
ok: [rtr3]

TASK [Copy launch_openr script to rtr] ****************************************************************************************************************
ok: [rtr1]
ok: [rtr3]
ok: [rtr4]

TASK [Check docker container is running] **************************************************************************************************************
 [WARNING]: Consider using 'become', 'become_method', and 'become_user' rather than running sudo

changed: [rtr3]
fatal: [rtr1]: FAILED! => {"changed": true, "cmd": "sudo -i docker inspect --format={{.State.Running}} openr", "delta": "0:00:00.492998", "end": "2019-02-17 12:59:37.194077", "msg": "non-zero return code", "rc": 1, "start": "2019-02-17 12:59:36.701079", "stderr": "stty: standard input: Inappropriate ioctl for device\nstty: standard input: Inappropriate ioctl for device\nError: No such image or container: openr", "stderr_lines": ["stty: standard input: Inappropriate ioctl for device", "stty: standard input: Inappropriate ioctl for device", "Error: No such image or container: openr"], "stdout": "", "stdout_lines": []}
...ignoring
changed: [rtr4]
fatal: [rtr1]: FAILED! => {"changed": true, "cmd": "sudo -i docker inspect --format={{.State.Running}} openr", "delta": "0:00:00.492998", "end": "2019-02-17 12:59:37.194077", "msg": "non-zero return code", "rc": 1, "start": "2019-02-17 12:59:36.701079", "stderr": "stty: standard input: Inappropriate ioctl for device\nstty: standard input: Inappropriate ioctl for device\nError: No such image or container: openr", "stderr_lines": ["stty: standard input: Inappropriate ioctl for device", "stty: standard input: Inappropriate ioctl for device", "Error: No such image or container: openr"], "stdout": "", "stdout_lines": []}
...ignoring

TASK [debug] ******************************************************************************************************************************************
ok: [rtr1] => {
    "output.stdout_lines": [
        "", 
        "", 
        "####### Restart the docker daemon to make sure changes take effect######", 
        "", 
        "", 
        "###### Successfully restarted the docker daemon, response: ######", 
        "", 
        "['ztp-user connected from 127.0.0.1 using console on rtr1',", 
        " '\\x1b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart',", 
        " 'Sun Feb  17 12:58:34.906 UTC',", 
        " 'docker stop/waiting',", 
        " 'docker start/running, process 17801',", 
        " 'sysadmin-vm:0_RP0#']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"ztp-user connected from 127.0.0.1 using console on rtr1\", ", 
        "    \"\\u001b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart\", ", 
        "    \"Sun Feb  17 12:58:34.906 UTC\", ", 
        "    \"docker stop/waiting\", ", 
        "    \"docker start/running, process 17801\", ", 
        "    \"sysadmin-vm:0_RP0#\"", 
        "]", 
        "Sleeping for about 30 seconds, waiting for the docker daemon to be up", 
        "", 
        "#######Pulling the docker image for Open/R ######", 
        "", 
        "Successfully downloaded the docker image", 
        "Using default tag: latest", 
        "latest: Pulling from akshshar/openr-xr", 
        "Digest: sha256:0d81b575830fe776739f960870652c7d9da601eaf32f68fa5569e852a2c5d4b0", 
        "Status: Image is up to date for akshshar/openr-xr:latest", 
        ""
    ]
}
ok: [rtr3] => {
    "output.stdout_lines": [
        "", 
        "", 
        "####### Restart the docker daemon to make sure changes take effect######", 
        "", 
        "", 
        "###### Successfully restarted the docker daemon, response: ######", 
        "", 
        "['ztp-user connected from 127.0.0.1 using console on rtr3',", 
        " '\\x1b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart',", 
        " 'Sun Feb  17 12:58:35.224 UTC',", 
        " 'docker stop/waiting',", 
        " 'docker start/running, process 20285',", 
        " 'sysadmin-vm:0_RP0#']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"ztp-user connected from 127.0.0.1 using console on rtr3\", ", 
        "    \"\\u001b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart\", ", 
        "    \"Sun Feb  17 12:58:35.224 UTC\", ", 
        "    \"docker stop/waiting\", ", 
        "    \"docker start/running, process 20285\", ", 
        "    \"sysadmin-vm:0_RP0#\"", 
        "]", 
        "Sleeping for about 30 seconds, waiting for the docker daemon to be up", 
        "", 
        "#######Pulling the docker image for Open/R ######", 
        "", 
        "Successfully downloaded the docker image", 
        "Using default tag: latest", 
        "latest: Pulling from akshshar/openr-xr", 
        "Digest: sha256:0d81b575830fe776739f960870652c7d9da601eaf32f68fa5569e852a2c5d4b0", 
        "Status: Image is up to date for akshshar/openr-xr:latest", 
        ""
    ]
}
ok: [rtr4] => {
    "output.stdout_lines": [
        "", 
        "", 
        "####### Restart the docker daemon to make sure changes take effect######", 
        "", 
        "", 
        "###### Successfully restarted the docker daemon, response: ######", 
        "", 
        "['ztp-user connected from 127.0.0.1 using console on rtr4',", 
        " '\\x1b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart',", 
        " 'Sun Feb  17 12:58:35.200 UTC',", 
        " 'docker stop/waiting',", 
        " 'docker start/running, process 14704',", 
        " 'sysadmin-vm:0_RP0#']", 
        "", 
        "###### return value in json: ######", 
        "", 
        "[", 
        "    \"ztp-user connected from 127.0.0.1 using console on rtr4\", ", 
        "    \"\\u001b[?7hsysadmin-vm:0_RP0# run ssh 10.0.2.16 service docker restart\", ", 
        "    \"Sun Feb  17 12:58:35.200 UTC\", ", 
        "    \"docker stop/waiting\", ", 
        "    \"docker start/running, process 14704\", ", 
        "    \"sysadmin-vm:0_RP0#\"", 
        "]", 
        "Sleeping for about 30 seconds, waiting for the docker daemon to be up", 
        "", 
        "#######Pulling the docker image for Open/R ######", 
        "", 
        "Successfully downloaded the docker image", 
        "Using default tag: latest", 
        "latest: Pulling from akshshar/openr-xr", 
        "Digest: sha256:0d81b575830fe776739f960870652c7d9da601eaf32f68fa5569e852a2c5d4b0", 
        "Status: Image is up to date for akshshar/openr-xr:latest", 
        ""
    ]
}

TASK [Clean up docker container if running] ***********************************************************************************************************
skipping: [rtr1]
changed: [rtr3]
changed: [rtr4]

TASK [debug] ******************************************************************************************************************************************
ok: [rtr3] => {
    "output.stdout_lines": [
        "openr"
    ]
}
ok: [rtr1] => {
    "output.stdout_lines": "VARIABLE IS NOT DEFINED!"
}
ok: [rtr4] => {
    "output.stdout_lines": [
        "openr"
    ]
}

TASK [Bring up the docker container] ******************************************************************************************************************
changed: [rtr3]
changed: [rtr4]
changed: [rtr1]

TASK [debug] ******************************************************************************************************************************************
ok: [rtr4] => {
    "output.stdout_lines": [
        "e53a77599eee5461b8ba7cf2651f5b486f7ba662d44d9e0d2caa0b018227ba46"
    ]
}
ok: [rtr1] => {
    "output.stdout_lines": [
        "74f2b43076353c47f3370bf7efb5b41257a311597813954680b5a7b32694e552"
    ]
}
ok: [rtr3] => {
    "output.stdout_lines": [
        "ba6c8bdb070c3f695ac706e385bca5c50d12b589b4bf6b781a2644815b647904"
    ]
}

TASK [Pause the playbook to allow routes to be distributed before running the reachability playbook] **************************************************
Pausing for 120 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
ok: [rtr1]

PLAY RECAP ********************************************************************************************************************************************
rtr1                       : ok=10   changed=3    unreachable=0    failed=0   
rtr3                       : ok=10   changed=4    unreachable=0    failed=0   
rtr4                       : ok=10   changed=4    unreachable=0    failed=0   

tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ 
```


Now if we check the BGP session state on rtr1, we see it has now been established:

```
RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show bgp summary 
Sun Feb 17 13:02:35.113 UTC
BGP router identifier 172.16.1.1, local AS number 65000
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 2
BGP main routing table version 2
BGP NSR Initial initsync version 2 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker               2          2          2          2           2           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
172.16.4.1        0 65000      86      87        2    0    0 00:02:32          0

RP/0/RP0/CPU0:rtr1#
RP/0/RP0/CPU0:rtr1#show bgp sessions 
Sun Feb 17 13:03:56.340 UTC

Neighbor        VRF                   Spk    AS   InQ  OutQ  NBRState     NSRState
172.16.4.1      default                 0 65000     0     0  Established  None
RP/0/RP0/CPU0:rtr1#


```



## Task 3: Check reachability for Loopback IPs distributed by Open/R


As part of the Ansible based rollout we're trying to achieve, it would be very useful to be ascertain that reachability has been established to a set of IPs based on the configuration pushed or the applications spun up.

Here, Open/R is configured to distribute the loopback0 IP addresses of each router to its neighbors, thereby allowing the BGP session to come up between rtr1 and rtr4.

Thus we utilize Ansible to check pings to each loopback0 IP from each router. Since we're dealing with IOS-XR, the API is exposed through the `Cisco-IOS-XR-ping-act.yang` model as shown below:


```
tesuto@dev1:~$ 
tesuto@dev1:~$ cd /home/tesuto/yang/
tesuto@dev1:~/yang$ pyang -f tree Cisco-IOS-XR-ping-act.yang 
module: Cisco-IOS-XR-ping-act

  rpcs:
    +---x ping
       +---w input
       |  +---w destination!
       |  |  +---w destination           string
       |  |  +---w repeat-count?         uint64
       |  |  +---w data-size?            uint64
       |  |  +---w timeout?              uint64
       |  |  +---w interval?             uint32
       |  |  +---w pattern?              xr:Hex-integer
       |  |  +---w sweep?                boolean
       |  |  +---w vrf-name?             string
       |  |  +---w source?               string
       |  |  +---w verbose?              boolean
       |  |  +---w type-of-service?      uint8
       |  |  +---w do-not-frag?          boolean
       |  |  +---w validate?             boolean
       |  |  +---w priority?             uint8
       |  |  +---w outgoing-interface?   string
       |  +---w ipv4* [destination]
       |  |  +---w destination        string
       |  |  +---w repeat-count?      uint64
       |  |  +---w data-size?         uint64
       |  |  +---w timeout?           uint64
       |  |  +---w interval?          uint32
       |  |  +---w pattern?           xr:Hex-integer
       |  |  +---w sweep?             boolean
       |  |  +---w vrf-name?          string
       |  |  +---w source?            string
       |  |  +---w verbose?           boolean
       |  |  +---w type-of-service?   uint8
       |  |  +---w do-not-frag?       boolean
       |  |  +---w validate?          boolean
       |  +---w ipv6!
       |     +---w destination           string
       |     +---w repeat-count?         uint64
       |     +---w data-size?            uint64
       |     +---w timeout?              uint64
       |     +---w interval?             uint32
       |     +---w pattern?              xr:Hex-integer
       |     +---w sweep?                boolean
       |     +---w vrf-name?             string
       |     +---w source?               string
       |     +---w verbose?              boolean
       |     +---w priority?             uint8
       |     +---w outgoing-interface?   string
       +--ro output
          +--ro ping-response
             +--ro ipv4* [destination]
             |  +--ro destination            string
             |  +--ro repeat-count?          uint64
             |  +--ro data-size?             uint64
             |  +--ro timeout?               uint64
             |  +--ro interval?              uint32
             |  +--ro pattern?               xr:Hex-integer
             |  +--ro sweep?                 boolean
             |  +--ro replies
             |  |  +--ro reply* [reply-index]
             |  |     +--ro reply-index                  uint64
             |  |     +--ro result?                      string
             |  |     +--ro broadcast-reply-addresses
             |  |        +--ro broadcast-reply-address* [reply-address]
             |  |           +--ro reply-address    string
             |  |           +--ro result?          string
             |  +--ro hits?                  uint64
             |  +--ro total?                 uint64
             |  +--ro success-rate?          uint64
             |  +--ro rtt-min?               uint64
             |  +--ro rtt-avg?               uint64
             |  +--ro rtt-max?               uint64
             |  +--ro sweep-min?             uint32
             |  +--ro sweep-max?             uint64
             |  +--ro rotate-pattern?        boolean
             |  +--ro ping-error-response?   string
             +--ro ipv6!
                +--ro destination       string
                +--ro repeat-count?     uint64
                +--ro data-size?        uint64
                +--ro timeout?          uint64
                +--ro interval?         uint32
                +--ro pattern?          xr:Hex-integer
                +--ro sweep?            boolean
                +--ro sweep-min?        uint32
                +--ro sweep-max?        uint64
                +--ro rotate-pattern?   boolean
                +--ro replies
                |  +--ro reply* [reply-index]
                |     +--ro reply-index    uint64
                |     +--ro result?        string
                +--ro hits?             uint64
                +--ro total?            uint64
                +--ro success-rate?     uint64
                +--ro rtt-min?          uint64
                +--ro rtt-avg?          uint64
                +--ro rtt-max?          uint64
tesuto@dev1:~/yang$ 


```

In the same way that we created an Ansible module for BGP configuration in Step-2, we have a pre-created Ansible Module to test IP reachability to specified destinations using the `Cisco-IOS-XR-ping-act.yang` model by leveraging ydk again. This can be seen in the following directory: 

```
tesuto@dev1:~$ cd code-samples/ansible/playbooks/reachability_check/
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ tree .
.
├── ip_dest_reachable_ydk.yml
├── library
│   └── ip_destination_reachable.py
└── ping_variables.yml

1 directory, 3 files
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ 

```

Dumping the Ansible module provided under `library/`:

```
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ cat library/ip_destination_reachable.py 
#!/usr/bin/env python3
#
# Copyright 2019 Cisco Systems, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

ANSIBLE_METADATA = {
    'metadata_version': '1.0',
    'supported_by': 'community',
    'status': ['preview']
}

DOCUMENTATION = """
---
module: ip_destination_reachable
short_description: Verify rechability of an IP destination
description:
    - Uses IP ICMP echo messages to probe the reachability of an IP prefix.
    - The module relies on NETCONF/YANG to execute a Ping RPC.
author: "Santiago Alvarez (@111pontes)"

options:
    host:
        description:
            - IP source to send rechability probes from
        type: str
        required: true
        default: null
        choices: []
        aliases: []

    destination:
        description:
            - IP destination to send reachability probes to
        type: str
        required: true
        default: null
        choices: []
        aliases: []

    repeat_count:
        description:
            - Number of probes to test for reachability
        type: int
        required: false
        default: 5
        choices: []
        aliases: []

    vrf_name:
        description:
            - Name of VRF used as context for reachability test
        type: str
        required: false
        default: null
        choices: []
        aliases: []

    min_success_rate:
        description:
            - Percentage threshold to declare destination as reachable
        type: int
        required: true
        default: null
        choices: []
        aliases: []

requirements:
    - ydk 0.6.3 (python)
    - ydk-models-cisco-ios-xr 6.3.1 (python)
"""

EXAMPLES = """
- ip_destination_reachable:
    host: '10.0.0.1'
    destination: '10.0.0.2'
    min_success_rate: 100
    vrf_name: 'default'
"""

RETURN = """
success_rate:
  description: Percentage of successful reachability probes
  returned: ping RPC succeeds
  type: int
rtt_min:
  description: minimum round trip time of all probes
  returned: ping RPC succeeds
  type: int
rtt_avg:
  description: average round trip time of all probes
  returned: ping RPC succeeds
  type: int
rtt_max:
  description: maximum round trip time of all probes
  returned: ping RPC succeeds
  type: int
"""


from ansible.module_utils.basic import AnsibleModule

from ydk.services import ExecutorService
from ydk.providers import NetconfServiceProvider
from ydk.models.cisco_ios_xr import Cisco_IOS_XR_ping_act as xr_ping_act

def ping(host="", 
         destination="", 
         repeat_count=5, 
         vrf_name="", 
         source_ip="",
         netconf_port=830,
         username="",
         password=""):
    """Execute Ping RPC over NETCONF."""

    # create NETCONF provider
    provider = NetconfServiceProvider(address=host,
                                      port=int(netconf_port),
                                      username=username,
                                      password=password,
                                      protocol='ssh')
    executor = ExecutorService()  # create executor service

    ping = xr_ping_act.Ping()  # create ping RPC object

    ping.input.destination.destination = destination
    ping.input.destination.repeat_count = repeat_count
    ping.input.destination.vrf_name = vrf_name
    ping.input.destination.source = source_ip

    ping.output = executor.execute_rpc(provider, ping, ping.output)

    rtt_min = ping.output.ping_response.ipv4[0].rtt_min
    rtt_avg = ping.output.ping_response.ipv4[0].rtt_avg
    rtt_max = ping.output.ping_response.ipv4[0].rtt_max

    return dict(success_rate=int(str(ping.output.ping_response.ipv4[0].success_rate)),
                rtt_min=int(0 if rtt_min is None else str(rtt_min)),
                rtt_avg=int(0 if rtt_avg is None else str(rtt_avg)),
                rtt_max=int(0 if rtt_max is None else str(rtt_max)))


def main():
    """Ansible module to verify IP reachability using Ping RPC over NETCONF."""
    module = AnsibleModule(
        argument_spec=dict(
            host=dict(type='str', required=True),
            destination=dict(type='str', required=True),
            repeat_count=dict(type='int', default=5),
            vrf_name=dict(type='str', default="default"),
            source_ip=dict(type='str'),
            netconf_port=dict(type='str'),
            username=dict(type='str', required=True),
            password=dict(type='str', required=True, no_log=True),
            min_success_rate=dict(type='int', default=100)
        ),
        supports_check_mode=True
    )

    if module.check_mode:
        module.exit_json(changed=False)

    try:
        retvals = ping(module.params['host'],
                    module.params['destination'],
                    module.params['repeat_count'],
                    module.params['vrf_name'],
                    module.params['source_ip'],
                    module.params['netconf_port'],
                    module.params['username'],
                    module.params['password'])
    except Exception as exc:
        module.fail_json(msg='Reachability validation failed ({})'.format(exc))

    retvals['changed'] = False

    if retvals['success_rate'] >= module.params['min_success_rate']:
        module.exit_json(**retvals)
    else:
        module.fail_json(msg=('Success rate lower than expected ({}<{})').
                              format(retvals['success_rate'],
                                     module.params['min_success_rate']))


if __name__ == '__main__':
    """Execute main program."""
    main()
# End of module
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ 


```


We can see that it leverages the ExecutorService() provided as part of the YDK toolset to perform netconf based actions using the ping action model.


Let's execute the playbook that leverages this module. The playbook is shown below:


```
tesuto@dev1:~$ cd code-samples/ansible/playbooks/reachability_check/
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ ls
ip_dest_reachable_ydk.yml  library  ping_variables.yml
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ cat ip_dest_reachable_ydk.yml 
---

- name: Verify IPv4 connectivity to routes learnt via Open/R 
  hosts: routers_netconf
  connection: local
  gather_facts: no

  vars_files:
    - './ping_variables.yml'
  tasks:
    - name: Reachability of loopbacks learnt via Open/R 
      ip_destination_reachable:
        host: '{{ ansible_host }}'
        destination: '{{ item }}'
        min_success_rate: 100
        source_ip: '{{ loopback0_ip }}'
        netconf_port: '{{ netconf_port }}'
        username: '{{ ansible_user }}'
        password: '{{ ansible_password }}'
      loop: '{{ loopback_list }}'
# End of playbook
tesuto@dev1:~/code-samples/ansible/playbooks/reachability_check$ 

```

Executing it:


```
tesuto@dev1:~/code-samples/ansible$ 
tesuto@dev1:~/code-samples/ansible$ ansible-playbook -i ansible_hosts playbooks/reachability_check/ip_dest_reachable_ydk.yml 

PLAY [Verify IPv4 connectivity to routes learnt via Open/R] *******************************************************************************************

TASK [Reachability of loopbacks learnt via Open/R] ****************************************************************************************************
ok: [rtr1] => (item=172.16.1.1)
ok: [rtr3] => (item=172.16.1.1)
ok: [rtr4] => (item=172.16.1.1)
ok: [rtr1] => (item=172.16.3.1)
ok: [rtr3] => (item=172.16.3.1)
ok: [rtr4] => (item=172.16.3.1)
ok: [rtr3] => (item=172.16.4.1)
ok: [rtr1] => (item=172.16.4.1)
ok: [rtr4] => (item=172.16.4.1)

PLAY RECAP ********************************************************************************************************************************************
rtr1                       : ok=1    changed=0    unreachable=0    failed=0   
rtr3                       : ok=1    changed=0    unreachable=0    failed=0   
rtr4                       : ok=1    changed=0    unreachable=0    failed=0   

tesuto@dev1:~/code-samples/ansible$ 

```


Excellent! Each router was able to ping the loopback0 ip address of every other router.
{: .notice--success}



## Task 5: Integrate all the playbooks into a single playbook


Your last task as part of the Day1 is to integrate all the playbooks till now:

1.  `config_oc_bgp_ydk.yml`: To configure BGP using YDK
2.  The playbook required to spin up the gNMI Telemetry collector docker container
3.  `docker_bringup.yml`: To bringup Open/R on each router.
4.  `ip_dest_reachable_ydk.yml`:  To check reachability established through Open/R.

Create one single playbook from these playbooks and edit `ansible_hosts` file to uncomment rtr2 entries and be ready for rtr2 when it comes up post Day0 operations!




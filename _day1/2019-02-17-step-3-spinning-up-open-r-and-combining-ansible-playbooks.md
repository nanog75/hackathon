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






---
published: true
date: '2019-02-17 07:11 -0800'
title: 'Step 3: Creating YANG-XML from config snippets for ZTP'
author: Nanog75 Hackathon Team
excerpt: >-
  Create XML files for different configuration snippets and wrap up the ncclient
  ZTP script to provision the boxes
tags:
  - nanog75
  - iosxr
  - cisco
  - tesuto
---


{% include toc %}
{% include base_path %}


## Final intended configuration for the routers

The final configuration required for each of the routers post ZTP is already present on the ZTP node.

Ssh to the ZTP node and drop into the `/var/www/html/configs/` directory:


```
tesuto@ztp:~$ cd /var/www/html/configs/
tesuto@ztp:/var/www/html/configs$ tree .
.
├── rtr1.config
├── rtr2.config
├── rtr3.config
└── rtr4.config

0 directories, 4 files
tesuto@ztp:/var/www/html/configs$ 

```

Since Day0 group is only dealing with rtr2 initially, let's look at rtr2 configuration:

```
tesuto@ztp:/var/www/html/configs$ cat rtr2.config 
!! IOS XR Configuration version = 6.5.2.28I
!! Last configuration change at Mon Feb 11 04:15:57 2019 by ZTP
!
hostname rtr2
domain name cisco.local
domain name-server 8.8.8.8
username rtrdev
 group root-lr
 group cisco-support
 secret 5 $1$mtK/$tVi/gbwfgZu6imOoriwxV.
!
tpa
 vrf default
  address-family ipv4
   default-route mgmt
   update-source dataports MgmtEth0/RP0/CPU0/0
  !
  address-family ipv6
   default-route mgmt
   update-source dataports MgmtEth0/RP0/CPU0/0
  !
 !
!
call-home
 service active
 contact smart-licensing
 profile CiscoTAC-1
  active
  destination transport-method http
 !
!
interface Loopback0
 ipv4 address 172.16.2.1 255.255.255.255
!
interface MgmtEth0/RP0/CPU0/0
 ipv4 address 100.96.0.16 255.240.0.0
 no shutdown
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.2.1.20 255.255.255.0
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.4.1.10 255.255.255.0
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.5.1.10 255.255.255.0
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/3
 shutdown
!
interface GigabitEthernet0/0/0/4
 shutdown
!
interface GigabitEthernet0/0/0/5
 shutdown
!
interface GigabitEthernet0/0/0/6
 shutdown
!
interface GigabitEthernet0/0/0/7
 shutdown
!
interface GigabitEthernet0/0/0/8
 shutdown
!
interface GigabitEthernet0/0/0/9
 shutdown
!
mpls static
 interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/1
 interface GigabitEthernet0/0/0/2
!
router static
 address-family ipv4 unicast
  0.0.0.0/0 100.96.0.1
 !
!
grpc
 port 57777
 no-tls
 service-layer
 !
!
netconf-yang agent
 ssh
!
lldp
!
ssh server v2
ssh server vrf default
ssh server netconf vrf default
end
tesuto@ztp:/var/www/html/configs$ 


```

This is the final configuration that must be achieved on rtr2 as a result of ZTP.

## Base Configuration on each router

We put a base configuration (username, tpa config) on each router and certain parts of the configuration (like ssh and netconf) are enabled during the init process for ncclient. Thus the common base configuration for all the routers is:


```

!! IOS XR Configuration version = 6.5.2.28I
!! Last configuration change at Mon Feb 11 04:15:57 2019 by ZTP
!
domain name cisco.local
domain name-server 8.8.8.8
username rtrdev
 group root-lr
 group cisco-support
 secret 5 $1$mtK/$tVi/gbwfgZu6imOoriwxV.
!
tpa
 vrf default
  address-family ipv4
   default-route mgmt
   update-source dataports MgmtEth0/RP0/CPU0/0
  !
  address-family ipv6
   default-route mgmt
   update-source dataports MgmtEth0/RP0/CPU0/0
  !
 !
!
call-home
 service active
 contact smart-licensing
 profile CiscoTAC-1
  active
  destination transport-method http
 !
 !
interface MgmtEth0/RP0/CPU0/0
 shutdown
!
interface GigabitEthernet0/0/0/3
 shutdown
!
interface GigabitEthernet0/0/0/4
 shutdown
!
interface GigabitEthernet0/0/0/5
 shutdown
!
interface GigabitEthernet0/0/0/6
 shutdown
!
interface GigabitEthernet0/0/0/7
 shutdown
!
interface GigabitEthernet0/0/0/8
 shutdown
!
interface GigabitEthernet0/0/0/9
 shutdown
!
!
!
netconf-yang agent
 ssh
!
!
ssh server v2
ssh server vrf default
ssh server netconf vrf default
end



```



## Config Snippet to be applied


Since , the only CLI configuration that must be converted to XML is:


rtr2.config.diff


```
hostname rtr2
!
interface Loopback0
 ipv4 address 172.16.2.1 255.255.255.255
!
interface MgmtEth0/RP0/CPU0/0
 ipv4 address 100.96.0.16 255.240.0.0
 no shutdown
!
interface GigabitEthernet0/0/0/0
 ipv4 address 10.2.1.20 255.255.255.0
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/1
 ipv4 address 10.4.1.10 255.255.255.0
 ipv6 enable
 no shutdown
!
interface GigabitEthernet0/0/0/2
 ipv4 address 10.5.1.10 255.255.255.0
 ipv6 enable
 no shutdown
!
mpls static
 interface GigabitEthernet0/0/0/0
 interface GigabitEthernet0/0/0/1
 interface GigabitEthernet0/0/0/2
!
grpc
 port 57777
 no-tls
 service-layer
 !
!
lldp
!
end

```






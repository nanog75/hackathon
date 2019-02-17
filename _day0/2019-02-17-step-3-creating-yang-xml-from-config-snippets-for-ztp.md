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


The Management port configuration, default route and domain-name settings are applied by the DHCP server's response, so we ignore these as well. 

Taking a diff between expected CLI and the base CLI, we get the following:


**rtr2.config.diff**


```
hostname rtr2
!
interface Loopback0
 ipv4 address 172.16.2.1 255.255.255.255
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

This is the configuration that must be applied to rtr2 by the ZTP script.


## Converting the required CLI into XML 


Let's break down the required CLI configuration into  individual Yang models that we need to use:


1. For the hostname CLI:

   ```
   !
   hostname rtr2
   ```

   The yang model to be used is: `Cisco-IOS-XR-shellutil-cfg.yang`
   
   Since this is an XR specific model, we've already converted the CLI snippet into XML:
   
   ```
   <config>
	<host-names xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-shellutil-cfg">
		<host-name>rtr2</host-name>
	</host-names>
   </config>
   
   ```
   
   This XML snippet is stored as `hostname.xml` on the webserver at `/var/www/html/xml/rtr2/`
   
   
2. For the grpc CLI:

   ```
   !
   grpc
    port 57777
    no-tls
    service-layer
    !
   !
   ```
   
   The Yang model to be used is :  `Cisco-IOS-XR-man-ems-cfg.yang`
   
   Again, being an XR specific model, we've generated the XML RPC for you:
   
   ```
   <config>
	<grpc xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-man-ems-cfg">
		<port>57777</port>
		<no-tls></no-tls>
		<enable></enable>
		<service-layer>
			<enable></enable>
		</service-layer>
	</grpc>
   </config>
   ```
   
   This XML snippet is stored as `grpc_config.xml` on the webserver at `/var/www/html/xml/rtr2/`
   
3. For the mpls static CLI:

   ```
   !
   mpls static
   interface GigabitEthernet0/0/0/0
   interface GigabitEthernet0/0/0/1
   interface GigabitEthernet0/0/0/2
   !
   
   ```
   The Yang model to be used is : `Cisco-IOS-XR-mpls-static-cfg.yang`
   
   Being an XR specific model, we've generated the XML for you:
   
   ```
   <config>
    <mpls-static xmlns="http://cisco.com/ns/yang/Cisco-IOS-XR-mpls-static-cfg">
		<enable></enable>
		<interfaces>
			<interface>
				<interface-name>GigabitEthernet0/0/0/0</interface-name>
			</interface>
			<interface>
				<interface-name>GigabitEthernet0/0/0/1</interface-name>
			</interface>
			<interface>
                 <interface-name>GigabitEthernet0/0/0/2</interface-name>
               </interface>
		</interfaces>
	</mpls-static>
   </config> 
   ```
   This XML snippet is stored as `mpls_static.xml` on the webserver at `/var/www/html/xml/rtr2/`
   
   
This leaves us with two CLI snippets:

lldp configuration:

```
!
lldp
!

```

and data port configurations:

```
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


```
   
Both of these CLI snippets have Openconfig Models available to generate the XML for them.
These are:

* LLDP Openconfig model:  `openconfig-lldp.yang`
* Interface Openconfig Model:  `openconfig-interfaces.yang`




## Getting Access to Yang models


Clone the `code-samples` repository to get access to the Yang models compatible with the release of the vendor with appropriate deviations implemented:


```
tesuto@ztp:~$ 
tesuto@ztp:~$ git clone https://github.com/nanog75/code-samples
Cloning into 'code-samples'...
remote: Enumerating objects: 1603, done.
remote: Counting objects: 100% (1603/1603), done.
remote: Compressing objects: 100% (625/625), done.
remote: Total 1603 (delta 1001), reused 1551 (delta 964), pack-reused 0
Receiving objects: 100% (1603/1603), 12.74 MiB | 7.52 MiB/s, done.
Resolving deltas: 100% (1001/1001), done.
Checking out files: 100% (2650/2650), done.
tesuto@ztp:~$ 
tesuto@ztp:~$ 
tesuto@ztp:~$ cd code-samples/ztp/yang/
tesuto@ztp:~/code-samples/ztp/yang$ 
tesuto@ztp:~/code-samples/ztp/yang$ 
tesuto@ztp:~/code-samples/ztp/yang$ 
tesuto@ztp:~/code-samples/ztp/yang$ ls
CISCO-ENTITY-FRU-CONTROL-MIB.yang                     Cisco-IOS-XR-manageability-object-tracking-cfg.yang
Cisco-IOS-XR-PRIVATE-ocacl-ipv4-oper-sub1.yang        Cisco-IOS-XR-manageability-object-tracking-datatypes.yang
Cisco-IOS-XR-PRIVATE-ocacl-ipv4-oper.yang             Cisco-IOS-XR-manageability-object-tracking-oper-sub1.yang
Cisco-IOS-XR-PRIVATE-ocacl-ipv6-oper-sub1.yang        Cisco-IOS-XR-manageability-object-tracking-oper.yang
Cisco-IOS-XR-PRIVATE-ocacl-ipv6-oper.yang             Cisco-IOS-XR-manageability-perfmgmt-cfg.yang
Cisco-IOS-XR-PRIVATE-ocacl-l2-oper-sub1.yang          Cisco-IOS-XR-manageability-perfmgmt-datatypes.yang
Cisco-IOS-XR-PRIVATE-ocacl-l2-oper.yang               Cisco-IOS-XR-manageability-perfmgmt-oper-sub1.yang
Cisco-IOS-XR-PRIVATE-ocni-bgp-oper-sub1.yang          Cisco-IOS-XR-manageability-perfmgmt-oper.yang
Cisco-IOS-XR-PRIVATE-ocni-bgp-oper.yang               Cisco-IOS-XR-mdrv-lib-cfg.yang
Cisco-IOS-XR-PRIVATE-ocni-bpm-oper-sub1.yang          Cisco-IOS-XR-mediasvr-linux-oper-sub1.yang
Cisco-IOS-XR-PRIVATE-ocni-bpm-oper.yang               Cisco-IOS-XR-mediasvr-linux-oper.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-rsvp-oper-sub1.yang    Cisco-IOS-XR-mpls-io-cfg.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-rsvp-oper.yang         Cisco-IOS-XR-mpls-io-oper-sub1.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-static-oper-sub1.yang  Cisco-IOS-XR-mpls-io-oper.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-static-oper.yang       Cisco-IOS-XR-mpls-ldp-cfg-datatypes.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-te-oper-sub1.yang      Cisco-IOS-XR-mpls-ldp-cfg.yang
Cisco-IOS-XR-PRIVATE-ocni-mpls-te-oper.yang           Cisco-IOS-XR-mpls-ldp-oper-datatypes.yang
Cisco-IOS-XR-Subscriber-infra-subdb-oper-sub1.yang    Cisco-IOS-XR-mpls-ldp-oper-sub1.yang
Cisco-IOS-XR-Subscriber-infra-subdb-oper-sub2.yang    Cisco-IOS-XR-mpls-ldp-oper-sub2.yang
Cisco-IOS-XR-Subscriber-infra-subdb-oper.yang         Cisco-IOS-XR-mpls-ldp-oper-sub3.yang
Cisco-IOS-XR-aaa-aaacore-cfg.yang                     Cisco-IOS-XR-mpls-ldp-oper.yang
Cisco-IOS-XR-aaa-diameter-base-mib-cfg.yang           Cisco-IOS-XR-mpls-lsd-cfg.yang
Cisco-IOS-XR-aaa-diameter-cfg.yang                    Cisco-IOS-XR-mpls-lsd-oper-sub1.yang
Cisco-IOS-XR-aaa-diameter-oper-sub1.yang              Cisco-IOS-XR-mpls-lsd-oper.yang
Cisco-IOS-XR-aaa-diameter-oper.yang                   Cisco-IOS-XR-mpls-oam-cfg.yang
Cisco-IOS-XR-aaa-lib-cfg.yang                         Cisco-IOS-XR-mpls-oam-oper-sub1.yang
Cisco-IOS-XR-aaa-lib-datatypes.yang                   Cisco-IOS-XR-mpls-oam-oper.yang
Cisco-IOS-XR-aaa-locald-admin-cfg.yang                Cisco-IOS-XR-mpls-static-cfg.yang
Cisco-IOS-XR-aaa-locald-cfg.yang                      Cisco-IOS-XR-mpls-static-oper-sub1.yang
Cisco-IOS-XR-aaa-locald-oper-sub1.yang                Cisco-IOS-XR-mpls-static-oper.yang
Cisco-IOS-XR-aaa-locald-oper.yang                     Cisco-IOS-XR-mpls-te-cfg.yang
Cisco-IOS-XR-aaa-nacm-cfg.yang                        Cisco-IOS-XR-mpls-te-datatypes.yang
Cisco-IOS-XR-aaa-nacm-oper-sub1.yang                  Cisco-IOS-XR-mpls-te-oper-sub1.yang
Cisco-IOS-XR-aaa-nacm-oper.yang                       Cisco-IOS-XR-mpls-te-oper-sub2.yang
Cisco-IOS-XR-aaa-protocol-radius-cfg.yang             Cisco-IOS-XR-mpls-te-oper-sub3.yang
Cisco-IOS-XR-aaa-protocol-radius-oper-sub1.yang       Cisco-IOS-XR-mpls-te-oper-sub4.yang
Cisco-IOS-XR-aaa-protocol-radius-oper-sub2.yang       Cisco-IOS-XR-mpls-te-oper-sub5.yang
Cisco-IOS-XR-aaa-protocol-radius-oper.yang            Cisco-IOS-XR-mpls-te-oper-sub6.yang
Cisco-IOS-XR-aaa-tacacs-cfg.yang                      Cisco-IOS-XR-mpls-te-oper


..........


```

Install pyang:

```
tesuto@ztp:~$ 
tesuto@ztp:~$ sudo pip3 install pyang
The directory '/home/tesuto/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
The directory '/home/tesuto/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Collecting pyang
  Downloading https://files.pythonhosted.org/packages/43/d3/0cc5538d83db3216f4c5acbfa5849a601dfe2e80e5fade872d5e6d5ee7d7/pyang-1.7.8-py2.py3-none-any.whl (447kB)
    100% |████████████████████████████████| 450kB 2.6MB/s 
Requirement already satisfied: lxml in ./.local/lib/python3.6/site-packages (from pyang)
Installing collected packages: pyang
Successfully installed pyang-1.7.8
tesuto@ztp:~$ 

```

## Parse the yang model using pyang


Let's look at the lldp yang model for example:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
tesuto@ztp:~/code-samples/ztp/yang$ 
tesuto@ztp:~/code-samples/ztp/yang$ pyang -f tree openconfig-lldp.yang 
module: openconfig-lldp
 <mark> +--rw lldp
     +--rw config
     |  +--rw enabled?                      boolean</mark>
     |  +--rw hello-timer?                  uint64
     |  +--rw suppress-tlv-advertisement*   identityref
     |  +--rw system-name?                  string
     |  +--rw system-description?           string
     |  +--rw chassis-id?                   string
     |  +--rw chassis-id-type?              oc-lldp-types:chassis-id-type
     +--ro state
     |  +--ro enabled?                      boolean
     |  +--ro hello-timer?                  uint64
     |  +--ro suppress-tlv-advertisement*   identityref
     |  +--ro system-name?                  string
     |  +--ro system-description?           string
     |  +--ro chassis-id?                   string
     |  +--ro chassis-id-type?              oc-lldp-types:chassis-id-type
     |  +--ro counters
     |     +--ro frame-in?           yang:counter64
     |     +--ro frame-out?          yang:counter64
     |     +--ro frame-error-in?     yang:counter64
     |     +--ro frame-discard?      yang:counter64
     |     +--ro tlv-discard?        yang:counter64
     |     +--ro tlv-unknown?        yang:counter64
     |     +--ro last-clear?         yang:date-and-time
     |     +--ro tlv-accepted?       yang:counter64
     |     +--ro entries-aged-out?   yang:counter64
     +--rw interfaces
        +--rw interface* [name]
           +--rw name         -> ../config/name
           +--rw config
           |  +--rw name?      oc-if:base-interface-ref
           |  +--rw enabled?   boolean
           +--ro state
           |  +--ro name?       oc-if:base-interface-ref
           |  +--ro enabled?    boolean
           |  +--ro counters
           |     +--ro frame-in?          yang:counter64
           |     +--ro frame-out?         yang:counter64
           |     +--ro frame-error-in?    yang:counter64
           |     +--ro frame-discard?     yang:counter64
           |     +--ro tlv-discard?       yang:counter64
           |     +--ro tlv-unknown?       yang:counter64
           |     +--ro last-clear?        yang:date-and-time
           |     +--ro frame-error-out?   yang:counter64
           +--ro neighbors
              +--ro neighbor* [id]
                 +--ro id              -> ../state/id
                 +--ro config
                 +--ro state
                 |  +--ro system-name?               string
                 |  +--ro system-description?        string
                 |  +--ro chassis-id?                string
                 |  +--ro chassis-id-type?           oc-lldp-types:chassis-id-type
                 |  +--ro id?                        string
                 |  +--ro age?                       uint64
                 |  +--ro last-update?               int64
                 |  +--ro port-id?                   string
                 |  +--ro port-id-type?              oc-lldp-types:port-id-type
                 |  +--ro port-description?          string
                 |  +--ro management-address?        string
                 |  +--ro management-address-type?   string
                 +--ro custom-tlvs
                 |  +--ro tlv* [type oui oui-subtype]
                 |     +--ro type           -> ../state/type
                 |     +--ro oui            -> ../state/oui
                 |     +--ro oui-subtype    -> ../state/oui-subtype
                 |     +--ro config
                 |     +--ro state
                 |        +--ro type?          int32
                 |        +--ro oui?           string
                 |        +--ro oui-subtype?   string
                 |        +--ro value?         binary
                 +--ro capabilities
                    +--ro capability* [name]
                       +--ro name      -> ../state/name
                       +--ro config
                       +--ro state
                          +--ro name?      identityref
                          +--ro enabled?   boolean
tesuto@ztp:~/code-samples/ztp/yang$ 
</code>
</pre>
</div>


The highlighted entries above indicate the exact fields we need to capture in the final XML snippet.
So this gets converted into:

```
<config>
        <lldp xmlns="http://openconfig.net/yang/lldp">
		<config>
			<enabled>true</enabled>
		</config>
	</lldp>
</config>

```

with `xmlns="http://openconfig.net/yang/` being the generic openconfig namespace denotion.

Do the same on `openconfig-interfaces.yang` and create the last XML snippet required.
{: .notice--success}




## Setting up the infra

So we have the following XML files, potentially for rtr2:

```
lldp_oc.xml, 
interfaces_oc.xml, 
mpls_static.xml, 
hostname.xml, 
grpc_config.xml
```

Store these files on the web server at `/var/www/html/xml/rtr2`

If you take a look at the existing script at `/var/www/html/scripts/ztp_ncclient.py`, there is already a Serial-Number to router name Map in each script.  

Use the map to determine unique urls per router. 






   
   


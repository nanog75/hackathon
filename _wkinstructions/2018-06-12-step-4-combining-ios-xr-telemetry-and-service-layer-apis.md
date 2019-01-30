---
published: true
date: '2018-06-12 10:13 -0400'
title: 'Step 3:  Combining IOS-XR Telemetry and Service Layer APIs'
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


### Service Layer APIs

These are APIs at the network infrastructure layer of the IOS-XR stack. These APIs enable model-driven access over gRPC to functionality verticals such as RIB , Label Switch Database, Interface Events and BFD events with more functionalities coming in the future.

Several key attributes of these APIs:
*  **Performance:** Highly Performant API enabling route programming at a rate of 20000-30000 routes/second
*  **Model Driven:** Protobuf IDL based model for RPC and data structure definitions
*  **Multi-Language Client support:** Choose any language binding supported by gRPC to write a service-layer API client

To learn more, head over to:

><https://xrdocs.io/cisco-service-layer/>

### Streaming Telemetry

Streaming Telemetry helps deliver a push-based technique for getting operational data out of an IOS-XR box - using model-driven YANG paths with the capacity to integrate with a wide variety of open source tools like pipeline, kafka, influxdb, ELK, prometheus and more or just write a gRPC telemetry client of your own and subscribe to the stream

Key features:

*  **Model Driven:** Structured data is sent back to collectors/clients allowing easy parsing and integration with larger remediation/alarm systems.
*  **Highly Scalable and Performant:** Telemetry has been shown to beat SNMP both at scale and at the rate at which data can be streamed out without negligible effect on the resources of the network device.
*  **Wide variety of integrations:** Open Source tools and integrations available for a vast set of monitoring tools in the industry.

To learn more, head over to:

><https://xrdocs.io/telemetry/>


>The vagrant environment consisting of two IOS-XRv9k instances must already be running. Verify >this by opening up a terminal and running the following commands:
>
>  ```
>  cd ~/topology
>  vagrant status
>
>  ```
> You should then see something like this:
>  ```
>  cisco@pod2:~/topology$ vagrant status
>  Current machine states:
>
>  rtr1                      running (virtualbox)
>  rtr2                      running (virtualbox)
>  devbox                    running (virtualbox)
>
>   This environment represents multiple VMs. The VMs are all listed
>   above with their current state. For more information about a specific
>   VM, run `vagrant status NAME`.
>   cisco@pod2:~/topology$ 
>  ```

{: .notice--info}


The topology we will be dealing with looks something like this:

![topology.png]({{site.baseurl}}/images/topology.png)


Ask the proctor for your credentials to access the pod. Click on the "VNC" option under Devnet_Pods/Podx where Podx is the pod assigned to you.
{: .notice--warning}


## Open up Sublime to view the code pieces we will be dealing with
![sublime.png]({{site.baseurl}}/images/sublime.png)

Keep this open to view the code at any point. Drop back into the terminal to continue.




##  Executing the combined Telemetry and SL-API code

This is the most complex code of this workshop. It is completely written in c++ and leverages gRPC to interact with IOS-XR for both the Telemetry Subscription (using Dial-in) as well as for Service Layer APIs.

The basic purpose of this code is:

1)  **Subscribe to IPv6 neighbor data**:  The Telemetry configuration we applied using ncclient in
Step 2 earlier, set up the internal telemetry agent in IOS-XR to be ready to send data to external collectors. This data is then used to check the state of the IPv6 neighbors of the designated active path interface:  `GigabitEthernet0/0/0/0`.

2) **Push Active-Path Routes to rtr1 RIB**:  The SL-API client connects to IOS-XR over gRPC in a separate thread and pushes an initial set of application routes into the RIB.

3) **Monitor the Active path's IPv6 neighbors**: Continuously monitor the telemetry stream received by the collector to note the current IPv6 neighbor on the active path interface and generate an event in case the neighbor goes missing (or returns after it is lost).


4) **Remediation Action**: If the pre-decided IPv6 neighbor is not found on this interface (because the local interface went down, or the remote interface went down or there was a break in connectivity in the intermediate layer 2 cloud), then the Telemetry client will trigger an event for the SL-API  (integrated code) client to update the routes in RIB to use the backup path as nexthop.
Bring back the IPv6 neighbor, and the router will come back to the original state.



This is illustrated in the figure below:


![xrtelemetry-slapi.png]({{site.baseurl}}/images/xrtelemetry-slapi.png)


### Drop into the relevant directory

On the devbox, cd into `/vagrant/code/advanced/xrtelemetry-slapi` directory:

```
vagrant@devbox$ cd /vagrant/code
vagrant@devbox:code$ ls
advanced                          ncclient                ydk
bigmuddy-network-telemetry-proto  service-layer-objmodel  ztp_cli_automation
vagrant@devbox:code$ cd advanced/
vagrant@devbox:advanced$ ls
xrtelemetry-slapi  ydk-slapi-remediation
vagrant@devbox:advanced$ cd xrtelemetry-slapi/
vagrant@devbox:xrtelemetry-slapi$ 
vagrant@devbox:xrtelemetry-slapi$ 
vagrant@devbox:xrtelemetry-slapi$ 


```

### Build the code 

Issue a `make clean` in the directory to make sure you start from a clean slate.

```
vagrant@devbox:xrtelemetry-slapi$ make clean
rm -f iosxrtelemetrysubmain  IosxrTelemetrySub.o IosxrTelemetryDecode.o IosxrTelemetryAction.o ServiceLayerRoute.o ServiceLayerAsyncInit.o IosxrTelemetryMain.o
vagrant@devbox:xrtelemetry-slapi$
```


Now build the code by issuing a `make`


```
vagrant@devbox:xrtelemetry-slapi$ make
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o IosxrTelemetrySub.o IosxrTelemetrySub.cpp
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o IosxrTelemetryDecode.o IosxrTelemetryDecode.cpp
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o IosxrTelemetryAction.o IosxrTelemetryAction.cpp
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o ServiceLayerRoute.o ServiceLayerRoute.cpp
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o ServiceLayerAsyncInit.o ServiceLayerAsyncInit.cpp
g++ -g -std=c++14 -I/usr/local/include  -I/usr/local/include/xrtelemetry  -pthread -c -o IosxrTelemetryMain.o IosxrTelemetryMain.cpp
g++ IosxrTelemetrySub.o IosxrTelemetryDecode.o IosxrTelemetryAction.o ServiceLayerRoute.o ServiceLayerAsyncInit.o IosxrTelemetryMain.o -L/usr/local/lib -I/usr/local/include -I/usr/local/include/xrtelemetry -lfolly  -lgrpc++_unsecure -lgrpc -lprotobuf -lpthread -ldl -liosxrsl -lglog -lxrtelemetry  -o iosxrtelemetrysubmain
vagrant@devbox:xrtelemetry-slapi$ 


```



You should see a binary created called `iosxrtelemetrysubmain`. We will use this to execute our code:

```
vagrant@devbox:xrtelemetry-slapi$ 
vagrant@devbox:xrtelemetry-slapi$ ls
IosxrTelemetryAction.cpp   IosxrTelemetryMain.o   ServiceLayerAsyncInit.cpp
IosxrTelemetryAction.h     IosxrTelemetrySub.cpp  ServiceLayerAsyncInit.h
IosxrTelemetryAction.o     IosxrTelemetrySub.h    ServiceLayerAsyncInit.o
IosxrTelemetryDecode.cpp   iosxrtelemetrysubmain  ServiceLayerException.h
IosxrTelemetryDecode.h     IosxrTelemetrySub.o    ServiceLayerRoute.cpp
IosxrTelemetryDecode.o     Makefile               ServiceLayerRoute.h
IosxrTelemetryException.h  quickstart.h           ServiceLayerRoute.o
IosxrTelemetryMain.cpp     README.md
vagrant@devbox:xrtelemetry-slapi$ ls iosxrtelemetrysubmain 
iosxrtelemetrysubmain
vagrant@devbox:xrtelemetry-slapi$ 

```

### Set the gRPC server and port Environment Variables

    
```
vagrant@devbox:xrtelemetry-slapi$ export SERVER_IP=20.1.1.10
vagrant@devbox:xrtelemetry-slapi$ export SERVER_PORT=57777
vagrant@devbox:xrtelemetry-slapi$ 

```
    
    
### Run the code


```
vagrant@devbox:xrtelemetry-slapi$ 
vagrant@devbox:xrtelemetry-slapi$ GLOG_v=3 ./iosxrtelemetrysubmain
vagrant@devbox:xrtelemetry-slapi$ 

```


### Create an event

Since the Telemetry code is monitoring the IPv6 neighbor information on the port Gig0/0/0/0, create events by shutting down the interface Gig0/0/0.
You will see that the code processes that the Ipv6 neighbor on the interface is lost, causing a failover to the backup path using Service-Layer APIs to inject application routes.
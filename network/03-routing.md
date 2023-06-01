---
layout: post
title: 03 Routing
date: 2023-05-29 23:55:00
giscus_comments: true
description: 
tags: router port NAT PAT
# categories: sample-posts
---


## Introducing Routers

* Router - a box that connects network IDs
* **Routers** filter and forward based on **IP address**
  * **Switches** filter and forward based on **MAC address**
* A routing table has at least four columns

| Address     | Subnet      | Gateway     | Interface   |
| ----------- | ----------- | ----------- | ----------- |
|192.168.15.0 |255.255.255.0| 0.0.0.0     |192.168.15.1 |
|232.25.201.0 |255.255.255.0| 0.0.0.0     |232.25.201.1 |

* Four zeros in the gateway means this router is directly connected to that network
* Interface is the side of the router each network is plugged into. 
* Each interface has an IP address of its network **ending with 1**
  * WAN(Wide Area Network) interface faces the Internet and has a public IP address 
  * LAN(Local Area Network) interface faces the home network's computers and has a private IP address

| Address     | Subnet      | Gateway     | Interface   |
| ----------- | ----------- | ----------- | ----------- |
|192.168.15.0 |255.255.255.0| 0.0.0.0     |192.168.15.1 |
|232.25.201.0 |255.255.255.0| 0.0.0.0     |232.25.201.1 |
|**0.0.0.0**  |**0.0.0.0**  |**232.25.201.11**|**232.25.201.1**|

* Default route
  * If it doesn't meet any other criteria, it always send it there.
  * Address and subnet start with the all zeros
  * send it out on this interface(232.25.201.1, coming from the Internet service provider) but send it directly to the next gateway(232.25.201.11)
  * The ISP has an upstream router, which IP address is going to be the next gateway(232.25.201.11)
  * There are many other routers attached to the ISP's upstream router (232.25.201.11/24) to provide customers with the internet service. 
    * In this case, 232.25.201.1 is assigned to my router.
    * My next door would have a router with an IP address of 232.25.201.2
    * This entire 232.25.201.0/24 network doesn't have any computers on it at all.
  
* Example: get a packet that needs to go to 232.25.201.66
  1. The router sees that the gateway is all zeros (232.25.201.66 matches the address 232.25.201.0/24 on the routing table)
  2. It tells the router that it is directly connected to 232.25.201.66
  3. The router knows it can ARP that system
  4. Your router will go out on interface 232.25.201.1 and ask for MAC address for 232.25.201.66 with ARP
  5. That device reponds back with the MAC address
  6. The router can now put on the Ethernet information and shoots it out to 232.25.201.1
  7. And that computer gets the packet
* Example2: gets a packet for 3.4.5.6
  1. The only route that's going to work is the default route
  2. It's going to ARP the gateway(232.25.201.11)
  3. That upstream router responds and sends his MAC address
  4. Now it knows how to send it and shoots it up to the router

* The only thing the routers do
  * read the IP address
  * change the MAC address depending where they want to send it to

* Assuming another network is plugged to the router

| Address     | Subnet      | Gateway     | Interface   | Metric      |
| ----------- | ----------- | ----------- | ----------- | ----------- |
|192.168.15.0 |255.255.255.0| 0.0.0.0     |192.168.15.1 |100          |
|232.25.201.0 |255.255.255.0| 0.0.0.0     |232.25.201.1 |100          |
|75.29.6.0    |255.255.255.0| 0.0.0.0     |75.29.6.144  |100          |
|0.0.0.0      |0.0.0.0      |232.25.201.11|232.25.201.1 |10           |
|0.0.0.0      |0.0.0.0      |75.29.6.1    |75.29.6.144  |11           |

* Metric
  * a relative value that gives your router an idea if it has more than one choice to do something which way does it go
  * The lower metric value the higher priority it has
  * If the router has something to send on the default route, it's oging to use the one with a lower metric value first

<br>

## Understanding Ports

| dest IP  | srce IP    | dest Port   | srce Port   |

* The destination port number is set by the type of application you are using
  * Web server : 80
  * FTP server : 21
  * email server : 110 
* **Port numbers 0 to 1023** are called **well known ports**, used for fixed applications and locked in stone
* The **source port number** is generated as an ephemeral port by the client
  * It's incrementally generated
  * It's got to be a number **1024 up to 65,000**

<br>

## NAT(Network Address Translation)

* NAT translates a private IP address to a public one
* NAT allows us to have lots of devices that are on the internet without using legitimate IP addresses
* Built into routers
* A router plugs his IP address to the private IP address
* NAT vs PAT
  * NAT(Network Address Translation) only modifies the L3 header(IP)
    * Ex: 10.4.4.42 -> 73.8.2.44
  * PAT(Port Address Translation) modifies both L3(IP) and L4(port) header
    * Ex: 10.4.4.41:8080 -> 73.8.2.44:80
    * Ex: 10.4.4.42:443 -> 73.8.2.44:443
* Static vs Dynamic
  * Static - explicit mapping between PRE-translation and POST-translation
  * Dynamic - a translation device makes a decision which host gets which IP address from a IP address pool 
    * Example: translate anything in 10.6.6.0/24 to 72.9.4.22, 72.9.4.23, or 72.9.4.24

<br>

## Forwarding Ports

* Port forwarding allows external devices to access to the internal server behind the router
  * Ex: 1.1.1.1:8181 -> 192.168.5.13:80
* Port range forward
  * Ex: 1.1.1.1:12001-12027 -> 192.168.5.11
* **Port triggering** will open an alternative assigned port when the initial port is contacted (e.g. FTP)
* SOHO DMZ
  * expose one computer to the entire internet
  * anything that comes in unsolicited from the internet, send it to a particular computer
  * able to see what kind of evil people are trying to do

<br>

## Static Routes

### Routing table 1

|Network Destination | Netmask        | Gateway     | Interface   | Metric      |
|:-----------        |:-----------    |:----------- | ----------- | ----------- |
|0.0.0.0             |0.0.0.0         | 192.168.4.1 |192.168.4.76 |25           |
|127.0.0.0           |255.0.0.0       | On-link     |127.0.0.1    |331          |
|127.0.0.1           |255.255.255.255 | On-link     |127.0.0.1    |331          |
|127.255.255.255     |255.255.255.255 | On-link     |127.0.0.1    |331          |
|192.168.4.0         |255.255.255.0   | On-link     |192.168.4.76 |281          |
|192.168.4.76        |255.255.255.255 | On-link     |192.168.4.76 |281          |
|192.168.4.255       |255.255.255.255 | On-link     |192.168.4.76 |281          |
|224.0.0.0           |240.0.0.0       | On-link     |127.0.0.1    |331          |
|224.0.0.0           |240.0.0.0       | On-link     |192.168.4.76 |281          |


1. I don't care where it's going to. Send it out on my gateway(192.168.4.1) through my network card(192.168.4.76)
2. **Unless** the destination address starts with 127, don't go to the gateway and just send it to my loopback(127.0.0.1)
3. **Unless** the destination starts with 192.168.4, do not send it out the gateway, just send it out my NIC(192.168.4.76)
4. 224 stands for multicast. Anything starts with 224 is a class D. It allows a computer to take on a second IP address that starts with a 224. For example, it is used to allow multi users to watch a video at the same time.

<br>

### Routing table 2

|Network Destination | Netmask        | Gateway     | Interface   |
|:-----------        |:-----------    |:----------- | ----------- |
|17.18.19.1          |255.255.255.255 |0.0.0.0      |WAN          |
|192.168.4.0         |255.255.255.0   |0.0.0.0      |LAN & WLAN   |
|17.18.19.0          |255.255.255.0   |0.0.0.0      |WAN          |
|169.254.0.0         |255.255.0.0     |0.0.0.0      |LAN & WLAN   |
|0.0.0.0             |0.0.0.0         |17.18.19.1   |WAN          |

1. If the destination IP address is 17.18.19.1, don't go out to the gateway, send it to the WAN.
2. If the destination IP address starts with 192.168.4, send it up to that 192.168.4.x/24 network through the LAN connection
3. Anything for 17.18.19.x, no gateway and send it out through the WAN
4. skip for now
5. This is a default route. Send everything over to the other system(17.18.19.1)
---
layout: post
title: 02 TCP/IP Basics
date: 2023-05-28 14:55:00
giscus_comments: true
description: 
tags: IP ARP "MAC address" "subnet mask" CIDR DHCP
# categories: sample-posts
---


## IP Addressing and Binary

* An IP address is represented as four values separated by three dots (`172.0.0.1`)
* A real IP address is 32 ones and zeros (`10101100.00000000.00000000.00000001`) 
* Dots don't exist in IP addresses (`10101100 00000000 00000000 00000001`) 
* Each octet has 2^8 (256) combinations (0 ~ 255)

<br>

## ARP (Address Resolution Protocol)

* ARP resolves MAC addresses from IP addresses
* ARP is used to construct an entire frame by finding a **destination MAC address**, which is going to be attached into IP packet(`dest IP` `srce IP` `dest Port` `srce Port` `Data`)
* `arp -a` command shows ARP cache

<br>

## Classful Addressing

* IANA(Internet Assigned Numbers Authority) keeps track of all of the IP addresses and passes them out to those who need them
* In fact, IANA passes out chunks of IP addresses to RIRs(Regional Internet Registries). e.g. ARIN: the American Registry for Internet Numbers
* RIRs pass out big chunks of IP addresses to ISPs(Internet Service Providers)
* Class licenses
  * Class A : 0-126/8, 2^24 (16,777,216) IP addresses, usually large organization like Comcast
  * Class B : 128-191/16, 2^16 (65,536) IP addresses 
  * Class C : 192-223/24, 2^8 (256) IP addresses
  * **Subnets don't have to be on the dots!**

<br>

## Subnet Masks

* Network ID: A part of the network numbering system that has to be identical for every computer on a particular network
* Host ID: A part following the network ID that changes for every individual computer in a network 
* Subnet Mask: a string of ones followed by a certain number of zeros
* 232.25.208.0/255.255.255.0
  * 232.25.208 -> Network ID
  * 0 -> Host ID
  * 255.255.255.0 -> Subnet Mask, simply say 24 in this case
* The host uses the subnet mask to know if the destination is on the local network or a remote network
* Each host knows the default gateway so that it can forward traffic to remote networks

<br>

## Subnetting with CIDR(Classless Inter-Domain Routing)

* 160.25.208.1 -> 10100000 11001000 11010000 00000001
* 208.25.160.0-255/24 (256 IP addresses available)
  * By moving the subnet mask over one bit from /24 to /25, two totally separate subnets are created from one class C address
    * Last octet starts from 0: 208.25.160.0-127/25 (128 IP addresses available)
    * Last octet starts from 1: 208.25.160.128-255/25 (another 128 IP addresses available)
  * If the subnet mask is 26, four subnets are created
    * Last octet starts from 00: 208.25.160.0-63/26 (64 IP addresses available)
    * Last octet starts from 01: 208.25.160.64-127/26 (64 IP addresses available)
    * Last octet starts from 10: 208.25.160.128-191/26 (64 IP addresses available)
    * Last octet starts from 11: 208.25.160.192-255/26 (64 IP addresses available)
  * The more you subnet something, the less hosts you have

<br>

## More CIDR Subnetting Practice

* /24 = 254 hosts (=2^8-2), the network ID is up to the third octet
* /25 = 126 hosts (=2^7-2), the network ID is up to the first number in the last octet
* /26 = 62  hosts (=2^6-2), the network ID is up to the second number in the last octet
* /27 = 30  hosts (=2^5-2), the network ID is up to the third number in the last octet
* /28 = 14  hosts (=2^4-2), the network ID is up to the fourth number in the last octet
* /29 = 6   hosts (=2^3-2), the network ID is up to the fifth number in the last octet
* /30 = 2   hosts (=2^2-2), the network ID is up to the sixth number in the last octet
* /31 = 0   hosts (=2^1-2), the network ID is up to the seventh number in the last octet

<br>

## Dynamic and Static IP Addressing

* Three IP address settings for a host - IP address / subnet mask / a default gateway
* When your computer first boots up, it doesn't have any IP information at all
* A DHCP(Dynamic Host Configuration Protocol) server supplies them to the computer
* A DHCP server is usually located in a separate server in a local network or built in a router attached to the local network  
* DHCP process
  * When your computer first boots up, he will beging sending out a broadcast called a `DHCP Discover` (a broadcast on the MAC address of all F's to all computers in the local networ, looking for a DHCP server)
  * A DHCP server, upon receiving the broadcast, sends unicast traffic back with a `DHCP Offer`
    * A DHCP Offer has all three essential information(IP addr, subnet mask, default gateway) and other stuffs
  * Your computer, upon receiving a DHCP Offer, sends a `DHCP request` back to the DHCP server
    * Basically telling the DHCP server that I am going to take and use this information that you are giving me
  * Once the DHCP server hears that, he sends a `DHCP acknowledgment`
  * The DHCP server will store all of this information and keep track of all of the different clients that are using DHCP
* Each broadcast domain must have only one DHCP server
* Every modern operating system compes with DHCP enabled by default
* DHCP Relay enables a single DHCP server to service more than one broadcast domain

<br>

## Rogue DHCP Servers

* APIPA (Automatic Private IP Addressing)
  * built into all of your DHCP clients and designed as a fallback if you can't find a DHCP server
  * If your DHCP server IP address starts with `169.254`(APIPA address), telling that your client cannot connect to a DHCP server
* If you get an APIPA address, check to see if you are connected to a DHCP server
* If you are connected to a DHCP server and still get an APIPA address, make sure the DHCP server is working
* If you get an IP address other than your correct network ID, you may have a rouge DHCP server

<br>

## Special IP Addresses

* Private IP addresses - only for serving a local network
  * `10.x.x.x`
  * `172.16.x.x` - `172.31.x.x`
  * `192.168.x.x`
* Loop back - talk to yourself
  * `172.0.0.1`(IPv4)/`::1`(IPv6)
  * the request goes out and come back in to test a network card (old days)
  * used to test their own internal circuitry (these days)
  * In reality, 127.x.x.x will always loop back
* APIPA
  * `169.254.x.x`
  * If a DHCP server's down APIPA will be given so at least the host can talk locally
  * You won't be able to get on the internet but you'd be able to share folders and files

<br>

## IP Addressing Scenarios

* Duplicate IP addresses
  * caused by a rouge DHCP server
  * The system is going to turn off your static IP address and try to set itself to DHCP
* Duplicate MAC addresses
  * happens when you're working with virtual machines
  * For VM's MAC address, I can type in whatever I want, accidently leading to the exact same MAC address for two completely different VMs
* Incorrect gateway
  * You can't get out of your local network
  * You still can share files and folders within the local network
  * Remember that your gateway is your router's IP address
* Incorrect subnet mask
  * Keep in mind that all of the computers within the same broadcast domain will always have the same subnet mask. No exception
  * A host with subnet mask of 255.255.0.0 is able to ping another host with 255.255.255.0, but the opposite direction is impossible
* Expired IP address
  * When a computer goes to a DHCP server it gets a lease. The period is often 8 days
  * Once it passes half of the lease period, the computer tries to re-establish the lease by interacting with the DHCP server
  * If the DHCP server dies, you have expired your DHCP lease
  * In Windows, the computer will keep that IP address at least through the time of the DHCP lease
  * Or the current IP address will be switched into APIPA(`169.254.x.x`)

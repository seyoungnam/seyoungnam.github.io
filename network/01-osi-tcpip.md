---
layout: post
title: 01 Networking Models
date: 2023-05-20 18:00:00
giscus_comments: true
description: 
tags: OSI TCP/IP Networking
# categories: sample-posts
---


## The OSI Model

1. **Physical** - what types of cables do I use?
2. **Data Link** - anything that works with a MAC address(network cards, switches)
3. **Network** - has to do with logical addresses(IP addresses)(routers)
4. **Transport** - disassembles packets but make sure the packets get to the other system in good order
5. **Session** - connects a server to a client on a remote system(i.e. TCP connection), defines what's taking place in terms of how that connectivity really works
6. **Presentation** - used to convert data into a format that your application can read
7. **Application** - the smarts in the applications that make the network aware

<br>

## The TCP/IP Model

1. **Network Interface** - covers all the physical cables, MAC addresses, network cards... tied to the physical & data link layer for the OSI
2. **Internet** - routers and anything it has to do with an IP address works at the Internet layer
3. **Transport** - has to do with connection to the other system and does all assembly and disassembly(i.e. TCP and UDP)
4. **Application** - everything that has to do with the application itself works at this layer(the OSP application, presentation, and session layer) (i.e. email, FTP, or telnet)

<br>

## Meet the Frame

Devices on a network send and receive data in discrete chunks called **frames**n (or **packets**).

A frame is generated inside the network card. Data comes down into the network card, which creates a frame and shoots it out to the network. Equally, as a frame comes into a network card, the data is pulled away sent up to whatever software needs it.

Frames are **a maximum of 1,500 bytes**(around 10,000 bits) in size and they have a discrete beginning and a discrete end.

<br>

## The MAC address

The physical MAC address is **a unique 48-bit identifier a the network card(NIC)**.

> The way to find your MAC address([link](https://www.cmu.edu/computing/services/endpoint/network-access/mac-address.html) or [description with image](https://ecs.rutgers.edu/how-find-your-physical-mac-address))

### How it's look like `87:67:5a:24:50:31`

* six pairs of a hexadecimal value(base 16)
* each hexadecimal character(16) represents four binary characters(2^4), so the MAC address is represented 48 binary characteers (4*12)
* The first three pairs(`87:67:5a`) : the OEM(Original Equipment Manufacturer) numbers
* The last three pairs(`24:50:31`) : The unique ID, burned into each card at the factory and each card gets a different value


### How the MAC addresses are added in the frame

| dest MAC addr | source MAC addr | Data | CRC(Cyclic Redundancy Check) |

Both destination and source MAC address are added to the data front. And for the data verification, CRC is added after the data. As the frame leaves the computer and comes into the hub, the hub creates as many copies as necessary to represent all the different computers it's connected to. And the hub sends them down the line to all the individual computers. As these frames come into the computer, the network card looks at the frame. If its MAC address is for him, then it's going to strip away extra info(MAC addresses and CRC) and send the core data up into the software of the system. If the destination MAC address is not for him, he just makes it disappear and doesn't do anything with it.

> The network cards use MAC addresses to decide whether or not to process a frame

<br>

## Broadcast vs Unicast

* Unicast transmission is addressed to a single device on a network
* A broadcast transmission is sent to every device in a broadcast domain(a group of computers that can hear each other's broadcast)
* A broadcast address looks like this: `FF-FF-FF-FF-FF-FF`

<br>

## Introduction to IP Addressing

MAC addresses don't identify in any way that a group of computers are all part of a single local area network. In order to make a big network work, we have to add a new type of addressing called logical addressing -- IP addressing. The idea behind IP addresses is that unlike MAC addresses they are not fixed with a network card. So IP addresses can actually be used to identify a particular network. For example, in a IP address (i.e. `31.44.17.231`), on the first three numbers all of the computers on this network will have the same first three values. The fourth value will be different for every individual computer. 

<br><br>

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.html path="https://static.javatpoint.com/tutorial/computer-network/images/switch-vs-router5.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

<br><br>

Two networks are interconnected by a **router**. To allow a computer in one network to communicate with another computer in the other network, we need to put the IP addresses to the frame.

* frame : | dest MAC | srce MAC | **dest IP** | **srce IP** | Data | CRC(Cyclic Redundancy Check) |
* IP packet : | dest IP | srce IP | Data |

A computer that receives the IP packet is going to look at the `destination IP` address. If he realizes that it's not part of his network, he will put a frame(**`dest MAC` of the router**, **`srce MAC` of the computer itself**, `CRC`) around the IP packet. This frame gets sent into the switch, and the switch sends it to the router.

Once it gets to the router, the router strips away the frame stuff leaving just the IP packet. The router performs a routing table lookup, gets the MAC address of the receiving computer as well as his MAC address, puts the entire frame together, and sends the frame into the receiving computer.

* frame : | **dest MAC of the receiving computer** | **srce MAC of the router** | dest IP | srce IP | Data | CRC |

The routing table tells, based on whatever the network information is, where to send data.

* 110.14.56 -> Use 110.14.56.1
* 32.44.17 -> Use 32.44.17.1
* 0.0.0.0 -> Use 76.4.22.123

> Keep in mind that it's routers that look at IP addresses to send data to the proper networks and that IP packets always sit within frames.

<br>

## Packets and Ports

Port numbers exist to send data to individual applications. Both destination and source port are added to a IP packet.

* IP packet: | dest IP | srce IP | **dest Port** | **srce Port** | Data |
* **dest Port**: a server's port the request is sent into
* **srce Port**: my computer's port to receive the response

The port number ranges from 0 up to 65,535. Among them, the first 1024 are reserved. For example,
* **Port 80**: HTTP
* **Port 20, 20**: FTP

### TCP(Transmission Control Protocol)
Data inside of a IP packet is a piece of the entire file. For example, a piece of a MS Word document or a part of email. TCP comes into play to put each piece together and turn them into an entire file. TCP is a connection oriented conversation between two computers to make sure that the data gets to you **whole complete** and **in order**.

* Sequencing number: allows you to reassemble data properly
* Acknowledgement number: the number that the server responds with, usually sequencing number + length

### UDP(User Datagram Protocol)
It's not connection oriented. One computer just sends the data and hopes that you're ready for it.

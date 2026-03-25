---
layout: distill
title: 02 Networking
description: 'EKS Study note'
giscus_comments: true
date: 2026-03-26
authors:
  - name: Seyoung Nam
    url: "https://www.linkedin.com/in/seyoungnam/"
toc:
  - name: Summary
  - name: 1. Kubernetes Networking Model
  - name: 2. Pod Network
    subsections:
      - name: Multiple IPs in a single node
      - name: Communication within a node
      - name: Communication between nodes
        subsections:
          - name: Overlay (Flannel, VXLAN)
          - name: BGP (Calico)
          - name: Cloud Native (AWS VPC CNI)
  - name: 3. Service Network
  - name: 4. Gateway Network
---

## Summary

_blank_

## 1. Kubernetes Networking Model
The [Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking) page in the k8s official doc describes the following requirements for Kubernetes networking:
- `Pod`: All pods can communicate with all other pods, **whether they are on the same node or on different nodes**. Pods can communicate with each other directly, **without** the use of proxies or address translation (**NAT**).
- `Service`: The Service API lets you provide a stable (long lived) IP address or hostname for a service implemented by one or more backend pods, where the individual pods making up the service can change over time.
- `Gateway`: The Gateway API (or its predecessor, Ingress) allows you to make Services accessible to clients that are outside the cluster.
- `NetworkPolicy`: NetworkPolicy is a built-in Kubernetes API that allows you to control traffic between pods, or between pods and the outside world.

The following sections are about how AWS EKS implements these requirements. Let's dive in the pod network.

## 2. Pod Network
Let's revisit the requirement for the pod networking:

> All pods can communicate with all other pods, **whether they are on the same node or on different nodes**. Pods can communicate with each other directly, **without** the use of proxies or address translation (**NAT**).

### Multiple IPs in a single node
In computer networking, if a host wants to talk to others, an IP address should be assigned to the host. Since a pod is the smallest unit for k8s networking, each pod should be assigned an IP address. But if you recall a networking 101 class you ever took in the past, a unit for the IP address is usually a single device(server, PC, router, etc), and thus programs in my laptop communicates each other by `localhost`(`127.0.0.1`). In k8s, a node is a single server and houses multiple pods. How could a single pod get assigned an IP and thus how does a single node end up with multiple IPs?

It turns out the smallest unit for the IP address assignment is not a single device but **Network Namespace**. Each network namespace has at least one network interface, an IP address, a port, and a routing table. In other words, **network namespace is the smallest networking unit**. Obviously, OS in a single node allows multiple network namespaces. In the k8s context, each pod is running under its own network namespace, with a set of network interface, IP addresses, ports, and routing tables.

{% include figure.html path="assets/img/eks/network-interface.jpeg" class="img-fluid rounded z-depth-1" %}

As described in the above image, a single IP address is assigned to each network namespace(or pod) in a node. With an IP address on each pod, how could we facilitate communications between them inside the node?

### Communication within a node

{% include figure.html path="assets/img/eks/communicate-in-node.jpeg" class="img-fluid rounded z-depth-1" %}

Two technologies are used to facilitate communication inside the node:
- `veth pair`: Think of it as a virtual ethernet cable. Each end is connected to a different network interface. If a packet comes into one end, it goes out the other end.
- `bridge interface`: It works as L2 switch, connecting pods within a single node. This is implemented by CNI plugins. 

If the above diagram looks confusing, try to understand it this way:
- each virtual network interface(`veth0` and `veth1`) is the representation of a pod in a node.
- the physical network interface in the node(`eth0`) is a router in a node.
- They are connected by the network switch(bridge interface).
- Overall, the entire diagram looks like a home network.

{% include figure.html path="assets/img/eks/communicate-in-node-2.jpeg" class="img-fluid rounded z-depth-1" %}

Each pod has its own IP address and they are connected via the bridge interface, which makes possible the communication between pods in the same node. We have confirmed the principle of pod networking on the same node. But kubernetes networking requires more than that. We need to ensure pod networking **on different nodes without Network Address Translation(NAT)**.

### Communication between nodes
We need to understand the meaning of **without NAT** first. NAT is a technique to allow a host in a local network to talk to another host in the different network. Please refer to [05 Network Address Translation(NAT)](/network/05-nat/) page for more details. The way to avoid NAT is to assign each pod to non-overlapping IP address across nodes. If each pod has a unique IP address in the cluster, we can regard the entire cluster as a single local network, and thus NAT is not required.

Then how can we ensure a unique IP address for each pod across nodes in the same cluster? There are three ways:

#### Overlay (Flannel, VXLAN)
{% include figure.html path="assets/img/eks/overlay.jpeg" class="img-fluid rounded z-depth-1" %}
An overlay is used to bridge a routing gap between the Pod network and the physical network. Because the physical network doesn't know which node owns which pod IP, the traffic must be **encapsulated** to get across. When a Pod sends a packet, the node wraps it in a new packet (typically UDP/VXLAN) addressed to the destination node. The receiving node then unwraps it and delivers it to the destination Pod. This provides great flexibility but adds a small amount of CPU overhead and reduces the effective MTU.

#### BGP (Calico)
{% include figure.html path="assets/img/eks/bgp.jpeg" class="img-fluid rounded z-depth-1" %}
BGP-based networking **avoids encapsulation** by making the underlying network fabric aware of Pod IP addresses. Nodes use the Border Gateway Protocol (BGP) to **advertise the Pod IP ranges** they are currently hosting **to other nodes and routers**. This allows packets to be routed natively between nodes without being wrapped in extra headers. It offers high performance and easier debugging, but requires a network infrastructure that can handle a large number of dynamic routes and potentially support BGP.


#### Cloud Native (AWS VPC CNI)
{% include figure.html path="assets/img/eks/aws-vpc-cni.jpeg" class="img-fluid rounded z-depth-1" %}
In AWS EKS, the default is the VPC CNI, which treats Pods as "first-class citizens" of the VPC. **Each Pod is assigned a real secondary private IP address from the VPC's own subnets**. Because the AWS VPC fabric natively understands these IPs, no overlays or BGP are required. Traffic is routed directly through the AWS infrastructure at native speeds. This also allows Pods to use VPC features like Security Groups and Flow Logs directly. However, it can consume VPC IP addresses quickly and is subject to ENI/IP limits per instance type.


## 3. Service Network

## 4. Gateway Network


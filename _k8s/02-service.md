---
layout: distill
title: 02 Connecting Pods over the network with Services
description: Most content in this document comes from "Learn Kubernetes in a Month of Lunches".
giscus_comments: true
date: 2022-01-08

authors:
  - name: Elton Stoneman
    url: "https://www.linkedin.com/in/eltonstoneman/?originalSubdomain=uk"
    affiliations:
      name: Sixeyed Consulting

# bibliography: 2018-12-22-distill.bib

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).
toc:
  - name: 1. Why are Services necessary for routing traffic in Kubernetes
    # if a section has subsections, you can add them as follows:
    # subsections:
      # - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2
  - name: 2. How Services route traffic
  - name: 3. The simplest Service YAML spec
    - name: 3-1. Routing traffic between Pods
    - name: 3-2. Routing external traffic to Pods
    - name: 3-3. Routing traffic outside Kubernetes
  - name: 4. Understanding Kubernetes Service resolution
  # - name: 5. Working with applications in Pods
  # - name: 6. Understanding Kubernetes resource management

# Below is an example of injecting additional post-specific styles.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }

---

## 1. Why are Services necessary for routing traffic in Kubernetes
Pods can communicate via IP address and they just use the standard protocols(TCP/UDP). However, it is not a good approach to solely rely on an IP address for communication. Because Pods are managed by Deployment controllers, it is hard to keep track of a new IP address whenever the Pod is replaced. The ideal solution would be to assign a static IP address to the ever-chaning Pod, which is why the concept of **Service** is invented and come to play in Kubernetes.


## 2. How Services route traffic
Like the internet, Kubernetes have its own DNS(Domain Name System, called `kube-dns`), mapping friendly names to IP addresses. Creating a Service effectively registers it with the DNS server, using its Service name and IP address. When a DNS receives **Service name** it returns **the Service's IP address, which is static for the life of the Service**. With the selector label that bonds the Service to the Pod(e.g. `app: sleep-2`), Kubernetes routes the traffic to the Service and then toss it to the Pod.


## 3. The simplest Service YAML spec
{% highlight yaml %}
apiVersion: v1    # Services use the core v1 API.
kind: Service
metadata:
  name: sleep-2   # The name of a Service is used as the DNS domain name.
# The specification requires a selector and a list of ports.
spec:
  selector:
    app: sleep-2  # Matches all Pods with an app label set to sleep-2.
  ports:
    - port: 80    # Listens on port 80 and sends to port 80 on the Pod
{% endhighlight %}


## 3. Three scenarios of routing traffic
There are roughly three types of services, depending on scenarios :
 * routing traffic between pods in the same cluster
 * routing external traffic to internal pods
 * routing traffic outside a Kubernetes cluster.


### 3-1. Routing traffic between Pods
This is the default type of Service and it is called **ClusterIP**. The assigned IP address works only within the cluster, so ClusterIP Services are useful only for communicating between Pods. Below is the basic YAML spec for ClusterIP Service.

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: numbers-api     # The Service uses the domain name numbers-api.
spec:
  ports:
    - port: 80          # The port the Service listens on
  selector:
    app: numbers-api    #  Traffic is routed to Pods with this label.
  type: ClusterIP       #  This Service is available only to other Pods.
{% endhighlight %}

After deploying a Service called `numbers-api`, the Service has its own IP address. However, an external IP is not yet assigned because the ClusterIP Service is available only within the cluster.

{% highlight shell %}
❯ kubectl apply -f numbers/api-service.yaml
service/numbers-api created
❯ kubectl get svc numbers-api
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.104.95.136   <none>        80/TCP    8s
{% endhighlight %}


### 3-2. Routing external traffic to Pods
A **LoadBalancer** Service integrates with an exteranl load balancer, which sends traffic to the cluster. The Service sends the traffic to a Pod, using the same label-selector mechanism to identify a target Pod. You might have many Pods that match the label selector for the Service, so the cluster needs to choose a node to send the traffic to and then choose a Pod on that node. But this issue is taken care of by Kubernetes. All you need to do is deploy a LoadBalancer Service. Below is the basic LoadBalancer Service YAML spec.

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: numbers-web
spec:
  ports:
    - port: 8080          # The port the Service listens on
      targetPort: 80      # The port the traffic is sent to on the Pod
  selector:
    app: numbers-web
  type: LoadBalancer      # This Service is available for external traffic.
{% endhighlight %}

Once setting this Service up, the external IP is assigned so you are no longer required to do port forward.

{% highlight shell %}
❯ kubectl apply -f numbers/web-service.yaml
service/numbers-web created
❯ kubectl get svc numbers-web
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
numbers-web   LoadBalancer   10.107.214.57   localhost     8080:32007/TCP   10s
{% endhighlight %}


### 3-3. Routing traffic outside Kubernetes
When you want to integerate some resources(e.g. database) running outside your cluster with any nodes running within the cluster, you need to use an **ExternalName Service**. ExternalName Services create a domain name alias and register it in the DNS server. When the Pod makes a lookup request using the local name, the DNS server resolves it to a fully qualified external name. Below is an example of the Service YAML spec.

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: numbers-api   # The local domain name of the Service in the cluster
spec:
  type: ExternalName
  externalName: raw.githubusercontent.com   # The domain to resolve
{% endhighlight %}

Check the Service configuration.

{% highlight shell %}
❯ kubectl apply -f numbers-services/api-service-externalName.yaml
service/numbers-api created
❯ kubectl get svc numbers-api
NAME          TYPE           CLUSTER-IP   EXTERNAL-IP                 PORT(S)   AGE
numbers-api   ExternalName   <none>       raw.githubusercontent.com   <none>    3m11s
{% endhighlight %}

If you want to route to an IP address rather than a domain name, you can use another option called **headless Services**, which are the ClusterIP type of Service but without a label selector so they will never match any Pods. Instead, the service is deployed with an **endpoint** resource that explicitly lists the IP addresses the Service should resolve. Please refer to an YAML spec example below.

{% highlight yaml %}
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  type: ClusterIP      # No selector field makes this a headless Service.
  ports:
    - port: 80
---
kind: Endpoints        # The endpoint is a separate resource.
apiVersion: v1
metadata:
  name: numbers-api
subsets: 
  - addresses:         # It has a static list of IP addresses . . . 
      - ip: 192.168.123.234
    ports:
      - port: 80       # and the ports they listen on.
{% endhighlight %}

Replace the ExternalName Service with the headless Service, check the Service and endpoints, and verify the DNS lookup.

{% highlight shell %}
❯ kubectl delete svc numbers-api
service "numbers-api" deleted
❯ kubectl apply -f numbers-services/api-service-headless.yaml
service/numbers-api created
endpoints/numbers-api created
❯ kubectl get svc numbers-api
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
numbers-api   ClusterIP   10.108.78.48   <none>        80/TCP    17s
❯ kubectl get endpoints numbers-api
NAME          ENDPOINTS            AGE
numbers-api   192.168.123.234:80   26s
❯ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
numbers-api   1/1     1            1           65m
numbers-web   1/1     1            1           65m
❯ kubectl exec deploy/numbers-web -- sh -c 'nslookup numbers-api | grep "^[^*]"'
Server:		10.96.0.10
Address:	10.96.0.10:53
Name:	numbers-api.default.svc.cluster.local
Address: 10.108.78.48
{% endhighlight %}

If you look the DNS lookup output above, you will find the two interesting facts. First, the resolved IP address(`10.108.78.48`) is not the endpoint but the ClusterIP address. Second, the domain name ends with `.default.svc.cluster.local`. We will get answers below.


## 4. Understanding Kubernetes Service resolution
To answer the first question, we need to keep in mind that Services have the up-to-date endpoint list. Therefore, clients need to visit the Service first to get the latest endpoint, and this is why the `numbers-api` resolves to its Service IP address.

Pods access the network through the a network proxy called `kube-proxy`, another internal Kubernetes component, and that uses packet filtering to send the Service's ClusterIP to the real endpoint. The reason `kube-proxy` has the up-to-date endpoint list is that Services have a controller that keeps the endpoint list updated whenever there are changes to Pods. Thus, `kube-proxy` is able to refresh the endpoint list whenever clients visits the static ClusterIP address.

For the second question, we need to understand a new Kubernetes resource - **namespace**. Every Kubernetes resource lives inside a namespace, which is a resource you can use to group other resources. Namespaces are a way to logically partition a Kubernetes cluster. In the above example, because the Service is in the `default` namespace, the suffix of the domain name begins with `.default.svc.cluster.local`. In the same manner, the full domain name for `kube-dns` in the `kube-system` namespace is `kube-dns.kube-system.svc.cluster.local`, as described below.

{% highlight shell %}
❯ kubectl get svc -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   4d21h
❯ kubectl exec deploy/numbers-web -- sh -c 'nslookup kube-dns.kube-system.svc.cluster.local | grep "^[^*]"'
Server:		10.96.0.10
Address:	10.96.0.10:53
Name:	kube-dns.kube-system.svc.cluster.local
Address: 10.96.0.10
{% endhighlight %}



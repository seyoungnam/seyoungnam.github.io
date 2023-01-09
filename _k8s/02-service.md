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
  - name: 1. Two core concepts
    # if a section has subsections, you can add them as follows:
    # subsections:
    #   - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2
  - name: 2. Containers vs Pods vs Nodes
  - name: 3. Running Pods with controllers
  - name: 4. Defining Deployments in application manifests
  - name: 5. Working with applications in Pods
  - name: 6. Understanding Kubernetes resource management

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

Kubernetes supports the standard networking protocols, TCP and UDP. Both protocols use IP addresses to route traffic, but IP addresses change when Pods are replaced, so Kubernetes provides a network address discovery mechanism with **Services**.

**Note:** Please `git clone` the official remote repo to your local machine, if you have not done it.
<d-code block language="shell">
  git clone https://github.com/sixeyed/kiamol
</d-code>


## 1. How Kubernetes routes network traffic

If you deploy two Pods, you can ping one Pod from the other.

{% highlight shell %}
# start up your lab environment--run Docker Desktop if it's not running--
# and switch to this chapter’s directory in your copy of the source code:
cd ch03

# create two Deployments, which each run one Pod:
kubectl apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml

# wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod -l app=sleep-2

# check the IP address of the second Pod:
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'

# use that address to ping the second Pod from the first:
kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')
{% endhighlight %}

We can learn from the above commands that Pods can reach each other over the network using an IP address. But the issue of using the Pod's IP address for communication is that Pods are usually managed by **Deployment** controllers. The IP address changes as its Pod is replaced with a new one. 

{% highlight shell %}
# check the current Pod’s IP address:
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
# 10.1.1.128% 

# delete the Pod so the Deployment replaces it:
kubectl delete pods -l app=sleep-2

# check the IP address of the replacement Pod:
kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
# 10.1.1.129%  
{% endhighlight %}

If we use the Pod's IP address for communication, we need to keep track of every changes in Pod's IP address, which is very cumbersome. Obviously, it is not a good idea to directly use the Pod's IP address to deal with an incoming request. The solution in Kubernetes is to introduce the concept of **Service**. The Service is an abstraction that houses a Pod and its network address. And it is coupled to the Pod using a label selector(e.g. `app: sleep-2`). It is the **Service** that is sitting in front of the Pod and  receives the incoming request using its **static** IP address. The Service then tosses the received request to the Pod linked with the same label selector(Both the Service and the Pod have `app: sleep-2` as a label selector). By utilizing the Service's **static IP address** and the **label selector** shared by the Pod, we don't need to keep track of change in Pod's IP address.

Below shows the minimal YAML spec for a Service, using the app label to identify the Pod which is the ultimate target of the network traffic.

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

You deploy a Service using a YAML file and the usual kubectl `apply` command. Deploy the Service, and verify the network traffic is routed to the Pod.

{% highlight shell %}
# deploy the Service defined in listing 3.1:
kubectl apply -f sleep/sleep2-service.yaml

# show the basic details of the Service:
kubectl get svc sleep-2

# run a ping command to check connectivity--this will fail:
kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2
# 1 packets transmitted, 0 packets received, 100% packet loss
{% endhighlight %}

The `sleep-1` Pod does a domain name lookup and receives the Service IP address from the Kubernetes DNS server. The `ping` command fails because it uses the ICMP protocol, which Kubernetes Services don't support. Services do support standard TCP and UDP traffic.



## 2. Routing traffic between Pods
The default type of Service in Kubernetes is called **ClusterIP**. The IP address **works only within the cluster**, so ClusterIP Services are useful only for communicating between Pods.

Run two Deployments, one for the web application and one for the API. This app has no Services yet, and it won’t work correctly because the website can’t find the API.

{% highlight shell %}
# run the website and API as separate Deployments: 
kubectl apply -f numbers/api.yaml -f numbers/web.yaml

# wait for the Pod to be ready:
kubectl wait --for=condition=Ready pod -l app=numbers-web

# forward a port to the web app:
kubectl port-forward deploy/numbers-web 8080:80

# browse to the site at http://localhost:8080 and click the Go button
# --you'll see an error message

# exit the port forward:
ctrl-c
{% endhighlight %}

The domain name it's using for the API is `numbers-api`, a local domain that doesn't exist in the Kubernetes DNS server. We need to create a Service with the name `numbers-api` and the label selector that matches the API Pod(`app: numbers-api`). Below is the Service spec.

{% highlight yaml %}
apiVersion: v1
kind: Service

metadata:
  name: numbers-api     # The Service uses the domain name numbers-api.

spec:
  ports:
    - port: 80
  selector:
    app: numbers-api    #  Traffic is routed to Pods with this label.
  type: ClusterIP       #  This Service is available only to other Pods.
{% endhighlight %}

Create a Service for the API so the domain lookup works and traffic is sent from the web Pod to the API Pod.
{% highlight shell %}
# deploy the Service from listing 3.2:
kubectl apply -f numbers/api-service.yaml

# check the Service details:
kubectl get svc numbers-api

# forward a port to the web app:
kubectl port-forward deploy/numbers-web 8080:80

# browse to the site at http://localhost:8080 and click the Go button
# I still can't generate a random number by clicking the Go button. Troubleshoot it later. 

# exit the port forward:
ctrl-c
{% endhighlight %}


## 3. Routing external traffic to Pods

Until now, we have dealt with routing traffic within the same cluster. To route traffic from external environment to a Pod, it is necessary to introduce another type of a Service called **LoadBalancer** is necessary. A **LoadBalancer Service** intergrates with an external load balancer, which sends traffic to the cluster. The Service sends the traffic to a Pod, using the same label-selector mechanism to identify a target Pod.

Below is the LoadBalancer Service for the web application.
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

Deploy the Service, and then use kubectl to find the address of the Service.
{% highlight shell %}
# deploy the LoadBalancer Service for the website--if your firewall checks 
# that you want to allow traffic, then it is OK to say yes:
kubectl apply -f numbers/web-service.yaml

# check the details of the Service:
kubectl get svc numbers-web
# NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# numbers-web   LoadBalancer   10.103.72.243   localhost     8080:32037/TCP   19s

# use formatting to get the app URL from the EXTERNAL-IP field:
kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
# http://localhost:8080% 
{% endhighlight %}

LoadBalancer Services listen on an exteranl IP address to route traffic to the cluster. In the above case, I am using Docker Desktop, which routes using the localhost address. With the LoadBalancer Service, now I can access the web application running in the Pod without needing a port forward from kubectl.




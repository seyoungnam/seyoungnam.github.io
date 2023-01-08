---
layout: distill
title: 01 Understanding Kubernetes
description: Most content in this document comes from "Learn Kubernetes in a Month of Lunches".
giscus_comments: true
date: 2022-01-06

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

## 1. Two core concepts

Kubernetes is a platform for running containers. The two core concepts in Kubernetes are ...

* **API** where to define your applications
* **cluster**, where to run your applications. A cluster is a single logical unit composed of many server nodes

A cluster is a single logical unit composed of many individual servers, called **nodes** in Kubernetes. Some nodes run the Kubernetes API, whereas others run application workloads, all in containers.

The way to interact with the Kubernetes cluster is through YAML files. You write your app specs in YAML files and send them to the Kubernetes API. Kubernetes compare them to the current app state/spec running in the cluster, and makes any changes to get to the desired state. 

Important concepts in Kubernetes.
  * `application manifests` : YAML files
  * `Resources` : components that go into shipping the app
    - `Services` : Kubernetes objects for managing network access
    - `Pods` : a unit of compute that runs one or more containers, has its own virtual IP address
    - `Deployments` : manage pods
    - `ReplicaSets`
    - `ConfigMaps`
    - `Secrets`

## 2. Containers vs Pods vs Nodes

  * Containers are packages of applications and execution environments.
  * Pods are collections of closely-related or tightly coupled containers.
  * Nodes are computing resources that house pods to execute workloads. Their responsiblity is to manage the pods and its containers by working with container runtime using a known API called the Container Runtime Interface(CRI). 
  (source: <a href="https://www.containiq.com/post/kubernetes-nodes-vs-pods-vs-containers" target="_blank">Kubernetes: Nodes vs. Pods vs. Containers
</a>)
  
  Kubernetes does not run containers. Pods are allocated to one node, and it's that node's responsibility to manage the Pod and its containers. It does that by working with the container runtime using a known API called the Container Runtime Interface(CRI).


## 3. Running Pods with controllers

If a node goes offline, the Pod is lost, and Kubernetes does not replace it. In this case, the `Deployment`, one of many `controllers` in Kubernetes, works with the Kubernetes API to watch the current state of the system, compares that to the desired state of its resources, and makes any changes necessary.

You can create Deployment resources with kubectl, specifying the container image you want to run and any other configuration for the Pod. Kubernetes creates the Deployment, and the Deployment creates the Pod.

```
# create a Deployment called "hello-kiamol-2", running the same web app:
kubectl create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol

# list all the Pods:
kubectl get pods
```

The Deployment specification described the Pod you wanted, and the Deployment created that Pod by interacting with the Kubernetes API. 

The Deployment adds labels to Pods it manages. In other words, a Deployment and related Pods share the same key-value label(e.g. app=x).


## 4. Defining Deployments in application manifests

Below YAML script is the simplest app manifest you can write.

**pod.yaml**
{% highlight shell %}
# Manifests always specify the version of the Kubernetes API
# and the type of resource.
apiVersion: v1
kind: Pod

# Metadata for the resource includes the name (mandatory) 
# and labels (optional).
metadata:
  name: hello-kiamol-3

# The spec is the actual specification for the resource.
# For a Pod the minimum is the container(s) to run, 
# with the container name and image.
spec:
  containers:
    - name: web
      image: kiamol/ch02-hello-kiamol
{% endhighlight %}

Kubectl doesn’t even need a local copy of a manifest file. It can read the contents from any public URL. Deploy the same Pod definition direct from the file on GitHub.

{% highlight shell %}
# deploy the application from the manifest file:
kubectl apply -f https://raw.githubusercontent.com/sixeyed/kiamol/master/ch02/pod.yaml
{% endhighlight %}

The Deployment definition is a composite that includes the Pod spec.
{% highlight shell %}
# Deployments are part of the apps version 1 API spec.
apiVersion: apps/v1
kind: Deployment
# The Deployment needs a name.
metadata:
  name: hello-kiamol-4
# The spec includes the label selector the Deployment uses 
# to find its own managed resources--I’m using the app label,
# but this could be any combination of key-value pairs.
spec:
  selector:
    matchLabels:
      app: hello-kiamol-4
# The template is used when the Deployment creates a Pod template.
# Pods in a Deployment don’t have a name, 
# but they need to specify labels that match the selector
# metadata.
  template:
    metadata:
      labels:
        app: hello-kiamol-4
# The Pod spec lists the container name and image spec.
    spec:
      containers:
        - name: web
          image: kiamol/ch02-hello-kiamol
{% endhighlight %}


## 5. Working with applications in Pods

You can run commands inside containers with kubectl and connect a terminal session, so you can connect into a Pod’s container as though you were connecting to a remote machine.

{% highlight shell %}
# check the internal IP address of the first Pod we ran: 
kubectl get pod hello-kiamol -o custom-columns=NAME:metadata.name,POD_IP:status.podIP

# run an interactive shell command in the Pod:
kubectl exec -it hello-kiamol -- sh

# inside the Pod, check the IP address:
hostname -i

# and test the web app:
wget -O - http://localhost | head -n 4

# leave the shell:
exit
{% endhighlight %}

For troubleshooting, the simple option is to read the application logs by accessing to the container runtime with `kubectl logs`.

{% highlight shell %}
# print the latest container logs from Kubernetes:
kubectl logs --tail=2 hello-kiamol

# and compare the actual container logs--if you’re using Docker:
docker container logs --tail=2 $(docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol)
{% endhighlight %}

You can run commands in Pods that are managed by a Deployment without knowing the Pod name, and you can view the logs of all Pods that match a label selector.

{% highlight shell %}
# make a call to the web app inside the container for the 
# Pod we created from the Deployment YAML file: 
kubectl exec deploy/hello-kiamol-4 -- sh -c 'wget -O - http://localhost > /dev/null'

# and check that Pod’s logs:
kubectl logs --tail=1 -l app=hello-kiamol-4
{% endhighlight %}

Kubectl lets you copy files between your local machine and containers in Pods. Create a temporary directory on your machine, and copy a file into it from the Pod container.

{% highlight shell %}
# create the local directory:
mkdir -p /tmp/kiamol/ch02

# copy the web page from the Pod:
kubectl cp hello-kiamol:usr/share/nginx/html/index.html /tmp/kiamol/ch02/index.html

# check the local file contents:
cat /tmp/kiamol/ch02/index.html
{% endhighlight %}


## 6. Understanding Kubernetes resource management

If you try to delete all Pods by using `kubectl delete pods --all`, you can see that Pods created by Deployment controllers are deleted and replaced with new ones.

{% highlight shell %}
# list all running Pods:
kubectl get pods

# delete all Pods:
kubectl delete pods --all

# check again:
kubectl get pods
{% endhighlight %}

If you want to delete a resource that is managed by a controller, you need to delete the controller instead. Controllers clean up their resources when they are deleted, so removing a Deployment is like a cascading delete that removes all the Deployment’s Pods, too.

{% highlight shell %}
# view Deployments:
kubectl get deploy

# delete all Deployments:
kubectl delete deploy --all

# view Pods:
kubectl get pods

# check all resources:
kubectl get all
{% endhighlight %}



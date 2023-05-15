---
layout: distill
title: 03 ConfigMaps and Secrets
description: Most content in this document comes from "Learn Kubernetes in a Month of Lunches".
giscus_comments: true
date: 2022-05-15

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
  - name: 4. Three scenarios of routing traffic
    # - name: 3-1. Routing traffic between Pods
    # - name: 3-2. Routing external traffic to Pods
    # - name: 3-3. Routing traffic outside Kubernetes
  - name: 5. Understanding Kubernetes Service resolution
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

## 1. Environment variables, ConfigMaps, and Secrets

Environment variables are variables applied to your program/application dynamically during runtime, based on which the way your program/application acts is different. Examples of environment variables are a password of the database running for your app, paths for packages/libraries that your code depends on, and go build variables (GOARCH, GOOS) to create a binary file that conforms to your system.

ConfigMaps and Secrets are kubernetes resources to supply environment variables to the Pod your application is running on. They are just storage units intended for small amounts of data. Those storage units can be loaded into a Pod, becoming part of the container environment, so that application in the container can read the data.


## 2. Four ways to add environment variables in the Pod

* Directly specify environment variables in the Pod spec
* Create a ConfigMap from literal values and load it into the Pod
* Create a ConfigMap populated from the environment file and load it into the Pod
* Create a ConfigMap containing key-value pairs and load it into the Pod

### 2.1. Directly specify environment variables in the Pod spec

In the below example, `KIAMOL_CHAPTER` is specified at `spec.template.spec.containers.env` of the Deployment yaml spec.

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:
          - name: KIAMOL_CHAPTER
            value: "04"
{% endhighlight %}

It is the most straight-forward way to supply environment variables to the Pod. A downside of it is that if you need to make configuration changes you need to perform an update with a replacement Pod, which will results in frequent Pod replacements.

### 2.2. Create ConfigMaps from literal values and load it into the Pod

Real application usually have more complex configuration requirements, which is when you use ConfigMaps. As mentioned above, ConfigMaps are just a storage unit that has data in the form of key-value pairs. To load a ConfigMap into a Pod, the ConfigMap needs to exist before you deploy the Pod. Below is the kubectl command to create a ConfigMap.

{% highlight shell %}
# create a ConfigMap with data from the command line:
❯ kubectl create configmap sleep-config-literal --from-literal=kiamol.section='4.1'
error: failed to create configmap: configmaps "sleep-config-literal" already exists

# check the ConfigMap details:
❯ kubectl get cm sleep-config-literal
NAME                   DATA   AGE
sleep-config-literal   1      47h
{% endhighlight %}

Then deploy the updated app to use the ConfigMap.

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:
          - name: KIAMOL_CHAPTER
            value: "04"
          - name: KIAMOL_SECTION              # a new env var from ConfigMap 
            valueFrom:
              configMapKeyRef:              
                name: sleep-config-literal    # ConfigMap name
                key: kiamol.section           # the data item to load
{% endhighlight %}

After deploying the above yaml spec, check the `KIAMOL_SECTION` environment variable.

{% highlight shell %}
# create a ConfigMap with data from the command line:
❯ kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
KIAMOL_CHAPTER=04
{% endhighlight %}


### 2.3. Create a new ConfigMap populated from the environment file and load it into the Pod

To supply a lot of configuration data in one scoop, create an environment file that can be loaded to create a ConfigMap is a good option to consider. For example, save a file `ch04.env` with the following content.

```
# Environment files use a new line for each variable.
KIAMOL_CHAPTER=ch04
KIAMOL_SECTION=ch04-4.1
KIAMOL_EXERCISE=try it now
```

Then create a ConfigMap with this env file.

{% highlight shell %}
# load an environment variable into a new ConfigMap:
❯ kubectl create configmap sleep-config-env-file --from-env-file=sleep/ch04.env
error: failed to create configmap: configmaps "sleep-config-env-file" already exists
# check the details of the ConfigMap:
❯ kubectl get cm sleep-config-env-file
NAME                    DATA   AGE
sleep-config-env-file   3      47h
{% endhighlight %}

Next, update the Pod to use the new ConfigMap.

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          envFrom:                            # newly added block to reference a ConfigMap
          - configMapRef:                     # newly added block to refernece a ConfigMap
              name: sleep-config-env-file     # newly added block to refernece a ConfigMap
          env:
          - name: KIAMOL_CHAPTER
            value: "04"
          - name: KIAMOL_SECTION
            valueFrom:
              configMapKeyRef:              
                name: sleep-config-literal
                key: kiamol.section
{% endhighlight %}

After deploying a new Deployment yaml spec, print out the environment variables. You will meet the interesting facts.

{% highlight shell %}
# update the Pod to use the new ConfigMap:
❯ kubectl apply -f sleep/sleep-with-configMap-env-file.yaml
deployment.apps/sleep configured
# check the values in the container:
❯ kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
error: unable to upgrade connection: container not found ("sleep")
snam@C02FL4Y9MD6T ch04 % kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
KIAMOL_EXERCISE=try it now
KIAMOL_SECTION=4.1
KIAMOL_CHAPTER=04
{% endhighlight %}

> When the same environment variable is supplied by multiple sources(one by ConfigMap defined under `env` and the other under `envFrom` in this example), the environment variables defined with `env` in the Pod spec override the values defined with `envFrom`.


### 2.3. Create a ConfigMap containing key-value pairs and load it into the Pod

This will be the most common way to use a ConfigMap. Key-value pairs for environment variables will be specified in the `data` section inside of ConfigMap. Please refer to the below example.

{% highlight yaml %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: todo-web-config-dev
data:
  config.json: |-
    {
      "ConfigController": {
        "Enabled" : true
      }
    }
{% endhighlight %}

The Pod bound with `todo-web` Deployment(see the spec below) will load this ConfigMap and mount `config.json` file onto the path `/app/config` in the container. The ConfigMap is treated like a directory.

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-web
spec:
  selector:
    matchLabels:
      app: todo-web
  template:
    metadata:
      labels:
        app: todo-web
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list 
          volumeMounts:
            - name: config
              mountPath: "/app/config"      # ConfigMap mounted path 
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: todo-web-config-dev       # Names a ConfigMap to load
{% endhighlight %}



## 3. Hierarchy of configuration sources

1. Default settings are baked into the container image.
2. The actual settings for each environment are stored in a ConfigMap and surfaced into the container filesystem.
3. Any settings that need to be tweaked can be applied as environment variables in the Pod spec for the Deployment.


## 4. Two ways to configure sensitive data with Secrets

* Reference a Secret object directly in the Pod
* Load a sensitive data into a file and reference the file location in the Pod



### 4.1. Reference a Secret object directly in the Pod

{% highlight shell %}
# now create a secret from a plain text literal:
❯ kubectl create secret generic sleep-secret-literal --from-literal=secret=shh...
deployment.apps/todo-web created

# show the friendly details of the Secret:
❯ kubectl describe secret sleep-secret-literal
Name:         sleep-secret-literal
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
secret:  6 bytes

# retrieve the encoded Secret value:
❯ kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}'
c2hoLi4u%  

# and decode the data:
❯ kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}' | base64 -d
shh...%  
{% endhighlight %}

> Creating Secrets from a literal value is equivalent to base64 encoding, which isn't really a security feature but just does prevent accidental exposure of secrets to someone looking over your shoulder.

The way to load Secrets to the Pod is almost the same with ConfigMaps, as you can see below.

{% highlight yaml %}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:                                # Environment variables
          - name: KIAMOL_SECRET               # Variable name in the container
            valueFrom:                        # loaded from an external source  
              secretKeyRef:                   # which is a Secret       
                name: sleep-secret-literal    # Names the Secret
                key: secret                   # Key of the Secret data item
{% endhighlight %}

Confirm that `KIAMOL_SECRET` is supplied to the Pod.

{% highlight shell %}
# update the sleep Deployment:
❯ kubectl apply -f sleep/sleep-with-secret.yaml
deployment.apps/sleep configured

# check the environment variable in the Pod:
❯ kubectl exec deploy/sleep -- printenv KIAMOL_SECRET
shh...
{% endhighlight %}

As you can observe in this case, the sensitive data provided by a literal Secret is easily exposed. Providing a sensitive data in a Secret yaml spec like below has the same issue.

{% highlight yaml %}
apiVersion: v1
kind: Secret                            # Secret is the resource type.
metadata:
 name: todo-db-secret-test             # Names the Secret
type: Opaque                            # Opaque secrets are for text data.
stringData:                             # stringData is for plain text.
 POSTGRES_PASSWORD: "kiamol-2*2*"      # The secret key and value.
{% endhighlight %}


### 4.2. Load a sensitive data into a file and reference the file location in the Pod
{% highlight yaml %}
apiVersion: v1
kind: Secret
metadata:
  name: todo-db-secret-test
type: Opaque
stringData:
  POSTGRES_PASSWORD: "kiamol-2*2*"
{% endhighlight %}


{% highlight yaml %}
    spec:
      containers:
        - name: db
          image: postgres:11.6-alpine
          env:
          - name: POSTGRES_PASSWORD_FILE          # Sets the path to the file
            value: /secrets/postgres_password
          volumeMounts:                           # Mounts a Secret volume
            - name: secret                        # Names the volume
              mountPath: "/secrets"
      volumes:
        - name: secret
          secret:                                 # Volume loaded from a Secret 
            secretName: todo-db-secret-test       # Secret name
            defaultMode: 0400                     # Permissions to set for files
            items:                                # Optionally names the data items
            - key: POSTGRES_PASSWORD
              path: postgres_password
{% endhighlight %}

Look at the above Pod spec. The value for `POSTGRES_PASSWORD_FILE` is not a sensitive data itself but a path where the data is stored. When this Pod is deployed, Kubernetes loads the value of the Secret item into a file at the path `/secrets/postgres_password`. That file will be set with 0400 permissions, which means it can be read by the container user but not by any other users, which effectively minimizes the exposure of the sensitive data.









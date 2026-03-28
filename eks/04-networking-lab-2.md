---
layout: distill
title: 04 EKS Networking Lab 2
description: 'EKS Study note'
giscus_comments: true
date: 2026-03-26
authors:
  - name: Seyoung Nam
    url: "https://www.linkedin.com/in/seyoungnam/"
toc:

---

> This lab is the continuation of 03 EKS Networking Lab 1. You need to provision an EKS cluster by following the instruction in the lab 1. 


## 1. AWS VPC CNI Configuration Change

Check the current environment of `aws-node` DaemonSet:
```bash
kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'
...
  {
    "name": "WARM_ENI_TARGET",
    "value": "1"
  },
...
```

Turn off `WARM_ENI_TARGET` and turn on `WARM_IP_TARGET` and `MINIMUM_IP_TARGET` in `eks.tf`:
```tf
  addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
      before_compute = true
      configuration_values = jsonencode({
        env = {
          # WARM_ENI_TARGET = "1" # 현재 ENI 외에 여유 ENI 1개를 항상 확보
          WARM_IP_TARGET  = "5" # 현재 사용 중인 IP 외에 여유 IP 5개를 항상 유지, 설정 시 WARM_ENI_TARGET 무시됨
          MINIMUM_IP_TARGET   = "10" # 노드 시작 시 최소 확보해야 할 IP 총량 10개
          #ENABLE_PREFIX_DELEGATION = "true" 
          #WARM_PREFIX_TARGET = "1" # PREFIX_DELEGATION 사용 시, 1개의 여유 대역(/28) 유지
        }
      })
    }
  }
```

Terraform apply:
```bash
terraform plan
terraform apply -auto-approve
```

Confirm the changes in the console:
EKS > addon > vpc-cni
{% include figure.html path="assets/img/eks/04-networking-la-2/vpc-cni.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Check `env` in `aws-node` DaemonSet:
```bash
kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'
kubectl describe ds aws-node -n kube-system | grep -E "WARM_IP_TARGET|MINIMUM_IP_TARGET"
```

Confirm ENI is added to the node where no pods are scheduled. You can see that every node has both `ens5` and `ens6` network interface. Please refer to [4.1. Variable setting](/eks/03-networking-lab/#41-variable-setting) to secloud-native-aws-vpc-ct `N1`, `N2`, `N3` variables.

```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c addr; echo; done
```
That's because of the `MINIMUM_IP_TARGET = "10"`. Note that five secondary IP addresses are allocated per ENI in a warm pool.
{% include figure.html path="assets/img/eks/04-networking-lab-2/secondary-ips.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Check cni logs:
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i tree /var/log/aws-routed-eni ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/plugin.log | jq ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/ipamd.log | jq ; echo; done

# IpamD debugging commands  https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i curl -s http://localhost:61679/v1/enis | jq; echo; done
```

## 2. Pod Count Constraints on Node

Install `kube-ops-view`:
```bash
helm repo add geek-cookbook https://geek-cookbook.github.io/charts/
helm install kube-ops-view geek-cookbook/kube-ops-view --version 1.2.2 --set service.main.type=NodePort,service.main.ports.http.nodePort=30000 --set env.TZ="America/New_York" --namespace kube-system
```

Confirm the `kube-ops-view` deployment:
```bash
kubectl get deploy,pod,svc,ep -n kube-system -l app.kubernetes.io/instance=kube-ops-view
```

Load the webpage:
```bash
open "http://$N1:30000/#scale=1.5"
open "http://$N1:30000/#scale=1.3"
```

{% include figure.html path="assets/img/eks/04-networking-lab-2/kube-ops-view.png" class="img-fluid rounded z-depth-1" zoomable=true %}

### 2.1. Pod Count Constraints for `t3.medium` Instance Type
The max pod count is determined by **max ENI count per instance type** and **max secondary IP count per ENI**. the max pod count can be calculated with the following:

`(Number of network interfaces for the instance type) X (Number of IPs per network interface - 1) + 2`

Note:
- The reason **one is subtracted from the number of IPs per network interface** is one IP address should be allocated to the network interface.
- The reason **two is added** is for taking `aws-node` and `kube-proxy` pods into the calculation. They are using the host IP address, not requiring a new IP address.


{% include figure.html path="assets/img/eks/04-networking-lab-2/max-pod-count.jpeg" class="img-fluid rounded z-depth-1" zoomable=true %}


Taking `t3.medium` as an example:
- Number of network interfaces for the instance type = 3
- Number of secondary IPs per network interface = 6
- Max pod count = 3 x (6 - 1) + 2 = 17

Confirm the max allocatable pod count:
```bash
kubectl describe node | grep Allocatable: -A6
Allocatable:
  cpu:                1930m
  ephemeral-storage:  18181869946
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  memory:             3371436Ki
  pods:               17
```

### 2.2. Pod Count Constraints for Other Instance Type
Check `t3` instance types:
```bash
aws ec2 describe-instance-types --filters Name=instance-type,Values=t3.\* \
 --query "InstanceTypes[].{Type: InstanceType, MaxENI: NetworkInfo.MaximumNetworkInterfaces, IPv4addr: NetworkInfo.Ipv4AddressesPerInterface}" \
 --output table

 --------------------------------------
|        DescribeInstanceTypes       |
+----------+----------+--------------+
| IPv4addr | MaxENI   |    Type      |
+----------+----------+--------------+
|  15      |  4       |  t3.2xlarge  |
|  15      |  4       |  t3.xlarge   |
|  6       |  3       |  t3.medium   |
|  12      |  3       |  t3.large    |
|  2       |  2       |  t3.micro    |
|  2       |  2       |  t3.nano     |
|  4       |  3       |  t3.small    |
+----------+----------+--------------+
```

Check `c5` instance types:
```bash
aws ec2 describe-instance-types --filters Name=instance-type,Values=c5\*.\* \
 --query "InstanceTypes[].{Type: InstanceType, MaxENI: NetworkInfo.MaximumNetworkInterfaces, IPv4addr: NetworkInfo.Ipv4AddressesPerInterface}" \
 --output table

-----------------------------------------
|         DescribeInstanceTypes         |
+----------+----------+-----------------+
| IPv4addr | MaxENI   |      Type       |
+----------+----------+-----------------+
|  50      |  15      |  c5ad.24xlarge  |
|  10      |  3       |  c5d.large      |
|  15      |  4       |  c5n.2xlarge    |
|  30      |  8       |  c5ad.4xlarge   |
... 
```

### 2.3. Lab - Deploy max pods
Open your terminal on each node and run `ip addr show` command:
```bash
ssh ec2-user@$N1
while true; do ip -br -c addr show && echo "--------------" ; date "+%Y-%m-%d %H:%M:%S" ; sleep 1; done

ssh ec2-user@$N2
while true; do ip -br -c addr show && echo "--------------" ; date "+%Y-%m-%d %H:%M:%S" ; sleep 1; done

ssh ec2-user@$N3
while true; do ip -br -c addr show && echo "--------------" ; date "+%Y-%m-%d %H:%M:%S" ; sleep 1; done
```
{% include figure.html path="assets/img/eks/04-networking-lab-2/lab-terminals.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Open another terminal and watch pods:
```bash
watch -d 'kubectl get pods -o wide'
```

Open another terminal and deploy pods:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

Confirm those pods are deployed:

{% include figure.html path="assets/img/eks/04-networking-lab-2/nginx-deployed.png" class="img-fluid rounded z-depth-1" zoomable=true %}

```bash
Every 2.0s: kubectl get pods -o wide                                                                                                     MacBookPro: Sat Mar 28 12:40:12 2026
                                                                                                                                                                in 2.301s (0)
NAME                               READY   STATUS    RESTARTS   AGE     IP               NODE                            NOMINATED NODE   READINESS GATES
netshoot-pod-64fbf7fb5-56nnb       1/1     Running   0          35h     192.168.0.106    ip-192-168-1-220.ec2.internal   <none>           <none>
netshoot-pod-64fbf7fb5-bqrdj       1/1     Running   0          35h     192.168.7.161    ip-192-168-5-239.ec2.internal   <none>           <none>
netshoot-pod-64fbf7fb5-nhdrk       1/1     Running   0          35h     192.168.8.207    ip-192-168-11-43.ec2.internal   <none>           <none>
nginx-deployment-54fc99c8d-7psbm   1/1     Running   0          3m34s   192.168.5.5      ip-192-168-5-239.ec2.internal   <none>           <none>
nginx-deployment-54fc99c8d-nnq8c   1/1     Running   0          3m34s   192.168.10.224   ip-192-168-11-43.ec2.internal   <none>           <none>
nginx-deployment-54fc99c8d-vnp77   1/1     Running   0          3m34s   192.168.1.157    ip-192-168-1-220.ec2.internal   <none>           <none>
```

Now let's add more pods:
```bash
kubectl scale deployment nginx-deployment --replicas=8
```

{% include figure.html path="assets/img/eks/04-networking-lab-2/nginx-8.png" class="img-fluid rounded z-depth-1" zoomable=true %}

```bash
kubectl scale deployment nginx-deployment --replicas=15
```

{% include figure.html path="assets/img/eks/04-networking-lab-2/nginx-15.png" class="img-fluid rounded z-depth-1" zoomable=true %}

The third ENI(`ens7`) and `veth` are also created:
{% include figure.html path="assets/img/eks/04-networking-lab-2/veth-added.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Let's keep scaling up to 30:
```bash
kubectl scale deployment nginx-deployment --replicas=30
```

{% include figure.html path="assets/img/eks/04-networking-lab-2/nginx-30.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Some pods can't be allocated due to IP address exhaustion:
```bash
kubectl scale deployment nginx-deployment --replicas=50
```

{% include figure.html path="assets/img/eks/04-networking-lab-2/nginx-50.png" class="img-fluid rounded z-depth-1" zoomable=true %}

```bash
kubectl get pods | grep Pending
nginx-deployment-54fc99c8d-5k2gp   0/1     Pending   0          2m20s
nginx-deployment-54fc99c8d-5n7qm   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-5tmx6   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-bpvll   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-cngvc   0/1     Pending   0          2m20s
nginx-deployment-54fc99c8d-cpqqk   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-csqbl   0/1     Pending   0          2m20s
nginx-deployment-54fc99c8d-jr7v4   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-lwhtl   0/1     Pending   0          2m20s
nginx-deployment-54fc99c8d-rh9rw   0/1     Pending   0          2m21s
nginx-deployment-54fc99c8d-rpnpt   0/1     Pending   0          2m20s
```

Check event logs:
```bash
kubectl events
...
3m31s                  Warning   FailedScheduling    Pod/nginx-deployment-54fc99c8d-5n7qm    0/3 nodes are available: 3 Too many pods. no new claims to deallocate, preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.
...
```

Check cni logs:
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i tree /var/log/aws-routed-eni; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/plugin.log | jq ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/ipamd.log | jq ; echo; done
...
{
  "level": "debug",
  "ts": "2026-03-27T10:44:12.451Z",
  "caller": "ipamd/ipamd.go:1624",
  "msg": "Found prefix pool count 0 for eni eni-0f2231da7b2f29b4b\n"
}
...
```

IpamD debugging commands:  https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i curl -s http://localhost:61679/v1/enis | jq; echo; done | grep -E 'node|TotalIPs|AssignedIPs'
>> node 98.92.230.99 <<
    "TotalIPs": 15,
    "AssignedIPs": 15,
>> node 34.228.20.240 <<
    "TotalIPs": 15,
    "AssignedIPs": 15,
>> node 3.95.16.221 <<
    "TotalIPs": 15,
    "AssignedIPs": 15,
```

Delete `nginx` pods:
```bash
kubectl delete deploy nginx-deployment
```
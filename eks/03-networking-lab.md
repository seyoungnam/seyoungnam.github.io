---
layout: distill
title: 03 EKS Networking Lab
description: 'EKS Study note'
giscus_comments: true
date: 2026-03-26
authors:
  - name: Seyoung Nam
    url: "https://www.linkedin.com/in/seyoungnam/"
toc:
  - name: 1. Cluster Provisioning
  - name: 2. Browse Cluster Default Setting
    subsections:
      - name: 2.1. Checklist in console
      - name: 2.2. Cluster Info
  - name: 3. AWS VPC CNI
    subsections:
      - name: 3.1. Browse CNI associated files in a worker node
      - name: 3.2. Secondary IP mode overview
  - name: 4. Networking Configuration on a Worker Node
    subsections:
      - name: 4.1. Variable setting
      - name: 4.2. Browse networking configurations
---

## 1. Cluster Provisioning

Get terraform resources:
```bash
# pull code
git clone https://github.com/gasida/aews.git
cd aews/2w
```

Set terraform variables:
```bash
export TF_VAR_KeyName=$(aws ec2 describe-key-pairs --query "KeyPairs[].KeyName" --output text)
export TF_VAR_ssh_access_cidr=$(curl -s ipinfo.io/ip)/32
echo $TF_VAR_KeyName $TF_VAR_ssh_access_cidr
```

Deploy EKS. Takes about 12 mins.
```bash
terraform init
terraform plan
nohup sh -c "terraform apply -auto-approve" > create.log 2>&1 &
tail -f create.log
```

Confirm your credentials used to provision the EKS cluster and switch kubectl context to your cluster:
```bash
terraform output -raw configure_kubectl
# aws eks update-kubeconfig --region us-east-1 --name myeks

aws eks --region us-east-1 update-kubeconfig --name myeks
kubectl config rename-context $(cat ~/.kube/config | grep current-context | awk '{print $2}') myeks
```

## 2. Browse Cluster Default Setting

### 2.1. Checklist in console
- Overview: API server endpoint, OpenID Connect provider URL
- Compute: Node groups -> `myeks-1nd-node-group` -> Kubernetes labes -> `{tier: primary}` (defined [here](https://github.com/gasida/aews/blob/main/2w/eks.tf#L87))
- Networking: Service IPv4 range(`10.100.0.0/16`), Subnets
- Add-ons: `CoreDNS`, `Amazon VPC CNI`, `kube-proxy` (all defined [here](https://github.com/gasida/aews/blob/main/2w/eks.tf#L142))
- Access: IAM access entries

### 2.2. Cluster Info

Browse cluster info:
```bash
kubectl cluster-info
eksctl get cluster
```

Browse node info:
```bash
kubectl get node --label-columns=node.kubernetes.io/instance-type,eks.amazonaws.com/capacityType,topology.kubernetes.io/zone
kubectl get node -v=6

# get node by label
kubectl get node --show-labels
kubectl get node -l tier=primary
```

Browse pod info:
```bash
kubectl get pod -A
kubectl get pdb -n kube-system
# NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# coredns   N/A             1                 1                     25h
```

Check node group:
```bash
aws eks describe-nodegroup --cluster-name myeks --nodegroup-name myeks-1nd-node-group | jq
```

Browse addons:
```bash
aws eks list-addons --cluster-name myeks | jq
eksctl get addon --cluster myeks
# NAME            VERSION                 STATUS  ISSUES  IAMROLE UPDATE AVAILABLE        CONFIGURATION VALUES               NAMESPACE       POD IDENTITY ASSOCIATION ROLES
# coredns         v1.13.2-eksbuild.3      ACTIVE  0                                         kube-system
# kube-proxy      v1.34.5-eksbuild.2      ACTIVE  0                                         kube-system
# vpc-cni         v1.21.1-eksbuild.5      ACTIVE  0                                       {"env":{"WARM_ENI_TARGET":"1"}}    kube-system
```

## 3. AWS VPC CNI

Please refer to [2.3.3. Cloud Native (AWS VPC CNI)](/eks/02-networking/#233-cloud-native-aws-vpc-cni) section to understand AWS VPC CNI.

### 3.1. Browse CNI associated files in a worker node
```bash
cat /etc/cni/net.d/10-aws.conflist | jq
# {
#   "cniVersion": "0.4.0",
#   "name": "aws-cni",
#   "disableCheck": true,
#   "plugins": [
#     {
#       "name": "aws-cni",
#       "type": "aws-cni",
#       "vethPrefix": "eni",
#       "mtu": "9001",
#       "podSGEnforcingMode": "strict",
#       "pluginLogFile": "/var/log/aws-routed-eni/plugin.log",
#       "pluginLogLevel": "DEBUG",
#       "capabilities": {
#         "io.kubernetes.cri.pod-annotations": true
#       }
#     },
#     {
#       "name": "egress-cni",
#       "type": "egress-cni",
#       "mtu": "9001",
#       "enabled": "false",
#       "randomizeSNAT": "prng",
#       "nodeIP": "",
#       "ipam": {
#         "type": "host-local",
#         "ranges": [
#           [
#             {
#               "subnet": "fd00::ac:00/118"
#             }
#           ]
#         ],
#         "routes": [
#           {
#             "dst": "::/0"
#           }
#         ],
#         "dataDir": "/run/cni/v4pd/egress-v6-ipam"
#       },
#       "pluginLogFile": "/var/log/aws-routed-eni/egress-v6-plugin.log",
#       "pluginLogLevel": "DEBUG"
#     },
#     {
#       "type": "portmap",
#       "capabilities": {
#         "portMappings": true
#       },
#       "snat": true
#     }
#   ]
# }

tree /opt/cni/bin
# /opt/cni/bin
# ├── LICENSE
# ├── aws-cni
# ├── aws-cni-support.sh
# ├── bandwidth
# ├── bridge
# ├── dhcp
# ├── dummy
# ├── egress-cni
# ├── firewall
# ├── host-device
# ├── host-local
# ├── ipvlan
# ├── loopback
# ├── macvlan
# ├── portmap
# ├── ptp
# ├── sbr
# ├── static
# ├── tap
# ├── tuning
# ├── vlan
# └── vrf
```

### 3.2. Secondary IP mode overview

As mentioned in [2.3.3. Cloud Native (AWS VPC CNI)](/eks/02-networking/#233-cloud-native-aws-vpc-cni), `Secondary IP mode` is the default mode for AWS VPC CNI. It is deployed as a `DaemonSet` named `aws-node`. The following is the IP address allocation process conducted by CNI:
1. When a worker node is provisioned, a primary ENI is attached to the node. 
2. The CNI allocates **a warm pool of ENIs** and **secondary IP addresses** from the subnet attached to the node's primary ENI. 
3. By default, `ipamd` attempts to allocate an additional ENI to the node. `ipamd` allocates this extra ENI as soon as a single Pod is scheduled and assigned a secondary IP address from the primary ENI. This **warm ENI enables faster Pod networking**.
4. When the pool of secondary IP addresses is exhausted, the CNI adds another ENI to allocate more.


The number of ENIs and IP addresses in the pool is configured through environment variables called [WARM_ENI_TARGET, WARM_IP_TARGET, MINIMUM_IP_TARGET](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/eni-and-ip-target.md).

| Item | WARM_ENI_TARGET | WARM_IP_TARGET | MINIMUM_IP_TARGET |
| :--- | :--- | :--- | :--- |
| **Control Unit** | ENI | IP | IP |
| **Usage** | Simple / Aggressive | Fine-grained control | Initial allocation |
| **Recommendation** | ❌ (Set to 0 to disable) | ✅ | ✅ |
| **Scaling Response** | Very Fast | Fast | Fast only initially |
| **Resource Efficiency** | Low | High | Medium |

`WARM_ENI_TARGET` is set to 1:
```bash
eksctl get addon --name vpc-cni --cluster myeks
# NAME    VERSION                 STATUS  ISSUES  IAMROLE UPDATE AVAILABLE        CONFIGURATION VALUES            NAMESPACE       POD IDENTITY ASSOCIATION ROLES
# vpc-cni v1.21.1-eksbuild.5      ACTIVE  0                                       {"env":{"WARM_ENI_TARGET":"1"}} kube-system

kubectl get daemonset aws-node --show-managed-fields -n kube-system -o yaml
        # - name: ENABLE_SUBNET_DISCOVERY
        #   value: "true"
        # - name: NETWORK_POLICY_ENFORCING_MODE
        #   value: standard
        # - name: VPC_CNI_VERSION
        #   value: v1.21.1
        # - name: VPC_ID
        #   value: vpc-010235c6a899415ba
        # - name: WARM_ENI_TARGET
        #   value: "1"
        # - name: WARM_PREFIX_TARGET
        #   value: "1"
```

Five secondary IP addresses per ENI are allocated in a warm pool. This matches the behavior where `WARM_ENI_TARGET=1` ensures that at least one extra "warm" ENI is always available with its full capacity of secondary IPs pre-allocated to the node's pool, reducing the latency when a new Pod needs an IP.
{% include figure.html path="assets/img/eks/03-networking-lab/secondary-ip.png" class="img-fluid rounded z-depth-1" %}

## 4. Networking Configuration on a Worker Node

### 4.1. Variable setting
EC2 ENI IP addresses:
```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table

# ------------------------------------------------------------------------
# |                           DescribeInstances                          |
# +-----------------------+------------------+----------------+----------+
# |     InstanceName      |  PrivateIPAdd    |  PublicIPAdd   | Status   |
# +-----------------------+------------------+----------------+----------+
# |  myeks-1nd-node-group |  192.168.7.237   |  54.204.76.91  |  running |
# |  myeks-1nd-node-group |  192.168.11.105  |  54.84.190.100 |  running |
# |  myeks-1nd-node-group |  192.168.2.202   |  44.201.60.214 |  running |
# +-----------------------+------------------+----------------+----------+
```

Set variables:
```bash
N1=54.204.76.91
N2=54.84.190.100
N3=44.201.60.214
```

ssh into worker nodes:
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh -o StrictHostKeyChecking=no ec2-user@$i hostname; echo; done
```

### 4.2. Browse networking configurations

Browse `aws-node` DaemonSet:
```bash
kubectl get daemonset aws-node --namespace kube-system -owide

kubectl get ds aws-node -n kube-system -o json | jq '.spec.template.spec.containers[0].env'
``` 
Check `kube-proxy` config:
```bash
kubectl describe cm -n kube-system kube-proxy-config
# mode: "iptables"

kubectl describe cm -n kube-system kube-proxy-config | grep iptables: -A5
# iptables:
#   masqueradeAll: false
#   masqueradeBit: 14
#   minSyncPeriod: 0s
#   syncPeriod: 30s
# ipvs:
```

Check IPs:
```bash
# Node IPs
aws ec2 describe-instances --query "Reservations[*].Instances[*].{PublicIPAdd:PublicIpAddress,PrivateIPAdd:PrivateIpAddress,InstanceName:Tags[?Key=='Name']|[0].Value,Status:State.Name}" --filters Name=instance-state-name,Values=running --output table

# Pod IPs
kubectl get pod -n kube-system -o=custom-columns=NAME:.metadata.name,IP:.status.podIP,STATUS:.status.phase
```

Browse CNI associated logs with the following commands:
```bash
# /var/log/aws-routed-eni contains log files
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i tree /var/log/aws-routed-eni ; echo; done

# plugin.log and ipamd.log collects logs about IP address allocation activities
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/plugin.log | jq ; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo cat /var/log/aws-routed-eni/ipamd.log | jq ; echo; done
{
  "level": "debug",
  "ts": "2026-03-25T09:23:46.634Z",
  "caller": "ipamd/ipamd.go:1669",
  "msg": "IP pool stats for network card 0: Total IPs/Prefixes = 10/0, AssignedIPs/CooldownIPs: 1/0, c.maxIPsPerENI = 5"
}
```

Browse Network interface on each node.
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -br -c addr; echo; done
>> node 54.204.76.91 <<
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens5             UP             192.168.7.237/22 metric 512 fe80::10dc:96ff:fe8a:51c7/64 

>> node 54.84.190.100 <<
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens5             UP             192.168.11.105/22 metric 512 fe80::8ff:fdff:fe95:3205/64 
eni19248b3518e@if3 UP             fe80::c055:29ff:fe4a:bf70/64 
ens6             UP             192.168.10.47/22 fe80::8ff:c2ff:fec6:ec69/64 

>> node 44.201.60.214 <<
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens5             UP             192.168.2.202/22 metric 512 fe80::3b:7aff:fe60:26af/64 
enic4269060f34@if3 UP             fe80::7c29:9cff:feab:7a89/64 
ens6             UP             192.168.0.137/22 fe80::c4:b6ff:feaf:5339/64 

```
Above output indicates that one node has only one ENI while others have multiple ENIs, because the first node has no pod assigned a new IP address. 
```bash
kubectl get pod -n kube-system -l k8s-app=kube-dns -owide
NAME                       READY   STATUS    RESTARTS   AGE   IP              NODE                             NOMINATED NODE   READINESS GATES
coredns-6d58b7d47c-m5mq4   1/1     Running   0          41h   192.168.8.88    ip-192-168-11-105.ec2.internal   <none>           <none>
coredns-6d58b7d47c-njdm8   1/1     Running   0          41h   192.168.0.112   ip-192-168-2-202.ec2.internal    <none>           <none>

k get po -A -owide
NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE                             NOMINATED NODE   READINESS GATES
kube-system   aws-node-4m5rm             2/2     Running   0          42h   192.168.2.202    ip-192-168-2-202.ec2.internal    <none>           <none>
kube-system   aws-node-rk8rf             2/2     Running   0          42h   192.168.7.237    ip-192-168-7-237.ec2.internal    <none>           <none>
kube-system   aws-node-w69rz             2/2     Running   0          42h   192.168.11.105   ip-192-168-11-105.ec2.internal   <none>           <none>
kube-system   coredns-6d58b7d47c-m5mq4   1/1     Running   0          42h   192.168.8.88     ip-192-168-11-105.ec2.internal   <none>           <none>
kube-system   coredns-6d58b7d47c-njdm8   1/1     Running   0          42h   192.168.0.112    ip-192-168-2-202.ec2.internal    <none>           <none>
kube-system   kube-proxy-5qmhn           1/1     Running   0          42h   192.168.2.202    ip-192-168-2-202.ec2.internal    <none>           <none>
kube-system   kube-proxy-kfh4m           1/1     Running   0          42h   192.168.7.237    ip-192-168-7-237.ec2.internal    <none>           <none>
kube-system   kube-proxy-q998k           1/1     Running   0          42h   192.168.11.105   ip-192-168-11-105.ec2.internal   <none>           <none>
```
`kubectl get pods` tells you only `core-dns` pods got assigned a new IP address and are running in nodes with multiple ENIs. Other pods such as `aws-node` and `kube-proxy` are configured with `hostNetwork: true` and thus using the node IP address. Why is that so?
- `aws-node` (VPC CNI): It needs to manage the node's Elastic Network Interfaces (ENIs) and assign secondary IPs to other pods. It cannot do this from an isolated network namespace. It needs direct access to the host's hardware and network stack.
- `kube-proxy`: It manages `iptables` or `IPVS` rules directly in the host's Linux kernel to route Service traffic. To modify the host's kernel networking rules, it must run in the host's network namespace.
CoreDNS is a standard application (a DNS server) that doesn't need to modify the underlying host's networking stack.

{% include figure.html path="assets/img/eks/03-networking-lab/worker-node-network.jpeg" class="img-fluid rounded z-depth-1" %}

<!-- 
```bash
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c addr; echo; done
for i in $N1 $N2 $N3; do echo ">> node $i <<"; ssh ec2-user@$i sudo ip -c route; echo; done

ssh ec2-user@$N1 sudo iptables -t nat -S
ssh ec2-user@$N1 sudo iptables -t nat -L -n -v
``` -->

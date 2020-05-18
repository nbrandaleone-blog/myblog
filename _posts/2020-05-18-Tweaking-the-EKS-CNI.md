---
layout: post
title:  "Tweaking the CNI on EKS"
date:   2020-05-18 08:00:00 -0400
categories: eks
---
![cni logo](/images/cni-logo.png)

## The Kubernetes network model

Every `Pod` gets its own IP address. This means you do not need to explicitly create links between `Pods` and you almost never need to deal with mapping container ports to host ports. This creates a clean, backwards-compatible model where `Pods` can be treated much like VMs or physical hosts from the perspectives of port allocation, naming, service discovery, load balancing, application configuration, and migration. This model is detailed [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/).

Now, in order for every pod to get its own address each Kubernetes cluster uses a plug-and-play CNI, or Container Network Interface, to assist with allocating IP addresses and setting up each `Pods` network. The [spec](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration) is quite straghtforward, and it is even possible to create your own CNI using [BASH](https://www.altoros.com/blog/kubernetes-networking-writing-your-own-simple-cni-plug-in-with-bash/).

While there are many excellent CNI modules available:
- [Flannel](https://github.com/coreos/flannel#flannel)
- [Calico](https://docs.projectcalico.org/introduction/)
- [Weave Net](https://www.weave.works/oss/net/)
- [Cilium](https://github.com/cilium/cilium)

it is recommended that one use the AWS CNI when operating EKS.  This CNI is fully supported by AWS, and has the advantage of using native `VPC` constructs when allocating IP addresses. So, each `Pod` gets an IP address (secondary address from one or more ENI's attached to the instance) from the VPC range that the cluster sits in. The AWS CNI is known as `aws-node`, running in the kube-system namespace.

## Upgrading your CNI

Many customers do not realize that the EKS CNI is not automatically upgraded during a cluster upgrade. Both the CNI and DNS are handled by daemon sets on each worker node.  The daemon sets must be upgraded independently of the cluster.

### Check your CNI version

``` sh
kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
```

If your CNI is less than 1.6.1 (as of May 2020), you should upgrade it.  The instructions are [here](https://docs.aws.amazon.com/eks/latest/userguide/cni-upgrades.html).

``` sh
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.6/aws-k8s-cni-cn.yaml
```

This will trigger a rolling update for your daemon sets.  Please note that your instance may not be able to allocate new pods for about 30 seconds per worker node during the upgrade process. So, this should probably be done off-hours for production clusters.

## Toggling SNAT functionality

When a pod communicated off-network, the source IP address is altered. It is source NATed, to become the primary private IP address of the instance. There are times when you want the pod IP to be preserved when communicating out of the VPC. 

By altering one of of the environmental variables which the CNI tracks, we can turn this functionality on and off.  Let's turn it on, to preserve the pod's source IP address.

``` sh
$ kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_EXTERNALSNAT=true
daemonset.apps/aws-node env updated

$ kubectl rollout status ds/aws-node -n kube-system
Waiting for daemon set "aws-node" rollout to finish: 0 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 0 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 1 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 2 out of 3 new pods have been updated...
Waiting for daemon set "aws-node" rollout to finish: 2 of 3 updated pods are available...
daemon set "aws-node" successfully rolled out
```

### It is `ip tables` all the way down..
> "In any team you need a tank, a healer, a damage dealer, someone with crowd control abilities, and another who knows iptables"
-- Jerome Petazzoni, on Twitter

I can confirm this by viewing the IP Tables on my worker nodes. When the default setting is on, this is what it looks like:

``` sh
# iptables -t nat -S AWS-SNAT-CHAIN-0
-N AWS-SNAT-CHAIN-0
-A AWS-SNAT-CHAIN-0 ! -d 172.31.0.0/16 -m comment --comment "AWS SNAT CHAN" -j AWS-SNAT-CHAIN-1

# iptables -t nat -S AWS-SNAT-CHAIN-1
-N AWS-SNAT-CHAIN-1
-A AWS-SNAT-CHAIN-1 -m comment --comment "AWS, SNAT" -m addrtype ! --dst-type LOCAL -j SNAT --to-source 172.31.22.247 --random
```

The logic states that when destination traffic is off of my VPC (not 172.31.0.0/16) and not going to the local interfaces, source NAT using the primary ENI (172.31.22.247) using random TCP port.

After we alter `AWS_VPC_K8S_CNI_EXTERNALSNAT`, this ip tables rule is deleted. Therefore, the pod source IP address is not altered as the packet leaves the worker node.

If you are curious as to how the worker node directs the traffic internally, check out the local ip routing table.  You will find that every pod has an individual route to a `veth`, or virtual ethernet adapter.

#### Pod with IP address 172.31.71.83 has local route
``` sh
# ip route
...
172.31.71.83 dev enic440f455693 scope link
...
```

One thing which might strike you as odd, is that all kubernetes services use another IP CIDR range. This is normal, in that kubernetes services are `virtual` by design, and are also handled by IP Tables manipulations.  For example, the kubernetes cluster IP is typically 10.100.0.10/32.  This is not part of my VPC.

``` sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   17h
```

#### On worker node, as root user
```
# iptables -t nat -S KUBE-SERVICES
-N KUBE-SERVICES
-A KUBE-SERVICES -d 10.100.0.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
...
```

You will find similar entries for all services. This is why all worker nodes have yet another daemon set, called `kube-proxy`, which is responsible for setting up and managing all the IP Tables rules on each worker node.

## Controlling how aggressive EKS allocates IP addresses

The number of ip addresses available per worker node is dependent upon its size. Each instance type/size has a limitation in how many ENI's can be attached, and how many secondary IP addresses each ENI can support.  Upon boot-up, the instance will _pre allocate_ IP addresses before they are needed.  This implies that I can quickly suffer from IP address exhaustion by simply booting up a large number of instances, even if they are **NOT** running a single `Pod`! This is why it is recommended that EKS be deployed into a relatively large VPC CIDR block.

The number of IP addresses supported by instances type can be seen [here](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt). 

``` sh
### Max IP addresses per EKS worker node instance class
# Mapping is calculated from AWS ENI documentation, with the following modifications:
# * First IP on each ENI is not used for pods
# * 2 additional host-networking pods (AWS ENI and kube-proxy) are accounted for
#
# # of ENI * (# of IPv4 per ENI - 1)  + 2
#
# https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
#
# If f1.16xlarge, g3.16xlarge, h1.16xlarge, i3.16xlarge, and r4.16xlarge
# instances use more than 31 IPv4 or IPv6 addresses per interface, they cannot
# access the instance metadata, VPC DNS, and Time Sync services from the 32nd IP
# address onwards. If access to these services is needed from all IP addresses
# on the interface, we recommend using a maximum of 31 IP addresses per interface.
a1.medium 8
a1.large 29
a1.xlarge 58
a1.2xlarge 58
a1.4xlarge 234
c1.medium 12
c1.xlarge 58
c3.large 29
c3.xlarge 58
c3.2xlarge 58
c3.4xlarge 234
c3.8xlarge 234
c4.large 29
c4.xlarge 58
c4.2xlarge 58
c4.4xlarge 234
c4.8xlarge 234
c5.large 29
c5.xlarge 58
c5.2xlarge 58
c5.4xlarge 234
c5.9xlarge 234
c5.12xlarge 234
c5.18xlarge 737
c5.24xlarge 737
c5.metal 737
c5d.large 29
c5d.xlarge 58
c5d.2xlarge 58
c5d.4xlarge 234
c5d.9xlarge 234
c5d.12xlarge 234
c5d.18xlarge 737
c5d.24xlarge 737
c5d.metal 737
c5n.large 29
c5n.xlarge 58
c5n.2xlarge 58
c5n.4xlarge 234
c5n.9xlarge 234
c5n.18xlarge 737
cc2.8xlarge 234
cr1.8xlarge 234
d2.xlarge 58
d2.2xlarge 58
d2.4xlarge 234
d2.8xlarge 234
f1.2xlarge 58
f1.4xlarge 234
f1.16xlarge 242
g2.2xlarge 58
g2.8xlarge 234
g3s.xlarge 58
g3.4xlarge 234
g3.8xlarge 234
g3.16xlarge 452
g4dn.xlarge 29
g4dn.2xlarge 29
g4dn.4xlarge 29
g4dn.8xlarge 58
g4dn.16xlarge 58
g4dn.12xlarge 234
g4dn.metal 737
h1.2xlarge 58
h1.4xlarge 234
h1.8xlarge 234
h1.16xlarge 452
hs1.8xlarge 234
i2.xlarge 58
i2.2xlarge 58
i2.4xlarge 234
i2.8xlarge 234
i3.large 29
i3.xlarge 58
i3.2xlarge 58
i3.4xlarge 234
i3.8xlarge 234
i3.16xlarge 452
i3.metal 737
i3en.large 29
i3en.xlarge 58
i3en.2xlarge 58
i3en.3xlarge 58
i3en.6xlarge 234
i3en.12xlarge 234
i3en.24xlarge 737
inf1.xlarge 38
inf1.2xlarge 38
inf1.6xlarge 234
inf1.24xlarge 437
m1.small 8
m1.medium 12
m1.large 29
m1.xlarge 58
m2.xlarge 58
m2.2xlarge 118
m2.4xlarge 234
m3.medium 12
m3.large 29
m3.xlarge 58
m3.2xlarge 118
m4.large 20
m4.xlarge 58
m4.2xlarge 58
m4.4xlarge 234
m4.10xlarge 234
m4.16xlarge 234
m5.large 29
m5.xlarge 58
m5.2xlarge 58
m5.4xlarge 234
m5.8xlarge 234
m5.12xlarge 234
m5.16xlarge 737
m5.24xlarge 737
m5.metal 737
m5a.large 29
m5a.xlarge 58
m5a.2xlarge 58
m5a.4xlarge 234
m5a.8xlarge 234
m5a.12xlarge 234
m5a.16xlarge 737
m5a.24xlarge 737
m5ad.large 29
m5ad.xlarge 58
m5ad.2xlarge 58
m5ad.4xlarge 234
m5ad.12xlarge 234
m5ad.24xlarge 737
m5d.large 29
m5d.xlarge 58
m5d.2xlarge 58
m5d.4xlarge 234
m5d.8xlarge 234
m5d.12xlarge 234
m5d.16xlarge 737
m5d.24xlarge 737
m5d.metal 737
m5dn.large 29
m5dn.xlarge 58
m5dn.2xlarge 58
m5dn.4xlarge 234
m5dn.8xlarge 234
m5dn.12xlarge 234
m5dn.16xlarge 737
m5dn.24xlarge 737
m5n.large 29
m5n.xlarge 58
m5n.2xlarge 58
m5n.4xlarge 234
m5n.8xlarge 234
m5n.12xlarge 234
m5n.16xlarge 737
m5n.24xlarge 737
m6g.medium 8
m6g.large 29
m6g.xlarge 58
m6g.2xlarge 58
m6g.4xlarge 234
m6g.8xlarge 234
m6g.12xlarge 234
m6g.16xlarge 737
p2.xlarge 58
p2.8xlarge 234
p2.16xlarge 234
p3.2xlarge 58
p3.8xlarge 234
p3.16xlarge 234
p3dn.24xlarge 737
r3.large 29
r3.xlarge 58
r3.2xlarge 58
r3.4xlarge 234
r3.8xlarge 234
r4.large 29
r4.xlarge 58
r4.2xlarge 58
r4.4xlarge 234
r4.8xlarge 234
r4.16xlarge 452
r5.large 29
r5.xlarge 58
r5.2xlarge 58
r5.4xlarge 234
r5.8xlarge 234
r5.12xlarge 234
r5.16xlarge 737
r5.24xlarge 737
r5.metal 737
r5a.large 29
r5a.xlarge 58
r5a.2xlarge 58
r5a.4xlarge 234
r5a.8xlarge 234
r5a.12xlarge 234
r5a.16xlarge 737
r5a.24xlarge 737
r5ad.large 29
r5ad.xlarge 58
r5ad.2xlarge 58
r5ad.4xlarge 234
r5ad.12xlarge 234
r5ad.24xlarge 737
r5d.large 29
r5d.xlarge 58
r5d.2xlarge 58
r5d.4xlarge 234
r5d.8xlarge 234
r5d.12xlarge 234
r5d.16xlarge 737
r5d.24xlarge 737
r5d.metal 737
r5dn.large 29
r5dn.xlarge 58
r5dn.2xlarge 58
r5dn.4xlarge 234
r5dn.8xlarge 234
r5dn.12xlarge 234
r5dn.16xlarge 737
r5dn.24xlarge 737
r5n.large 29
r5n.xlarge 58
r5n.2xlarge 58
r5n.4xlarge 234
r5n.8xlarge 234
r5n.12xlarge 234
r5n.16xlarge 737
r5n.24xlarge 737
t1.micro 4
t2.nano 4
t2.micro 4
t2.small 11
t2.medium 17
t2.large 35
t2.xlarge 44
t2.2xlarge 44
t3.nano 4
t3.micro 4
t3.small 11
t3.medium 17
t3.large 35
t3.xlarge 58
t3.2xlarge 58
t3a.nano 4
t3a.micro 4
t3a.small 8
t3a.medium 17
t3a.large 35
t3a.xlarge 58
t3a.2xlarge 58
u-6tb1.metal 147
u-9tb1.metal 147
u-12tb1.metal 147
x1.16xlarge 234
x1.32xlarge 234
x1e.xlarge 29
x1e.2xlarge 58
x1e.4xlarge 58
x1e.8xlarge 58
x1e.16xlarge 234
x1e.32xlarge 234
z1d.large 29
z1d.xlarge 58
z1d.2xlarge 58
z1d.3xlarge 234
z1d.6xlarge 234
z1d.12xlarge 737
z1d.metal 737
```

 By default, ipamD attempts to keep one elastic network interface and all of its IP addresses available for pod assignment. However, lets assume one might want to have only 10 warm IP addresses pre allocated.

 ```sh
kubectl set env daemonset -n kube-system aws-node WARM_IP_TARGET=10

# Confirm that env variable is set properly
kubectl exec pod/aws-node-m4gdb -n kube-system -- printenv | grep WARM_IP_TARGET
WARM_IP_TARGET=10
```

Altering these environmental variables requires a daemon set restart, since env variables are only examined upon pod bootup.  So, again be careful if this is done in the middle of the day.

Check out the [docs](https://docs.aws.amazon.com/eks/latest/userguide/cni-env-vars.html) to see what variables can be tweaked.  There is a new variable called `MINIMUM_IP_TARGET`, which can be set alongside `WARM_IP_TARGET` to tightly control how many IP addresses are consumed.

## CNI Custom Networking

It is unfortunately common in Enterprise environments to run out of IP addresses. This even happens when using Cloud Providers. The EKS CNI has two features which can allow it to grow beyond the original VPC design.

- Use custom networking on a per worker-node basis
- Use an overlay network just for the `Pods` IP addressing (100.64.0.0/10 and 198.19.0.0/16)

**NOTE**: Pod density is lower with custom networking. Enabling a custom network effectively removes an available elastic network interface (and all of its available IP addresses for pods) from each worker node that uses it. The primary network interface for the worker node is not used for pod placement when a custom network is enabled.

 Due to this limitation, it is probably best **NOT** to use custom networking unless you absolutely need to.

**Another NOTE**: Managed worker nodes cannot use custom networking at this time.

#### To configure CNI custom networking

1. Associate a secondary CIDR block to your cluster's VPC. 
2. Create a subnet in your VPC for each Availability Zone, using your secondary CIDR block. Your custom subnets must be from a different VPC CIDR block than the subnet that your worker nodes were launched into. 
3. Set the `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true` environment variable to true in the aws-node DaemonSet:

``` sh
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

The steps here get a little involved, so I will reference the following documents and labs.

- AWS [docs](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)
- AWS [workshop](https://eksworkshop.com/beginner/160_advanced-networking/secondary_cidr/)
- AWS [support](https://aws.amazon.com/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/)

#### Use an overlay network

The overlay is really the same concept as the VPC expansion.  AWS is just recommending the following CIDR blocks (100.64.0.0/10 and 198.19.0.0/16) for exclusive pod networking. These blocks are considered [reserved](https://en.wikipedia.org/wiki/Reserved_IP_addresses), so they can only be used by the `Pods`, and not for direct communication between VPC, VPN's or other external networks.

*****

### References

* [EKS CNI docs](https://docs.aws.amazon.com/eks/latest/userguide/cni-env-vars.html)
* [Setting the SNAT toggle](https://docs.aws.amazon.com/eks/latest/userguide/external-snat.html)
* [Jeremy Blog on EKS CNI networking](https://medium.com/@jeremy.i.cowan/custom-networking-with-the-aws-vpc-cni-plug-in-c6eebb105220)
* [Jeremy's follow up blog](https://medium.com/@jeremy.i.cowan/the-impacts-of-using-custom-networking-with-the-aws-vpc-cni-65f109d245be)
* [EKS conservation patterns in a hybrid network](https://aws.amazon.com/blogs/containers/eks-vpc-routable-ip-address-conservation/)
* [EKS networking workshop](https://eksworkshop.com/beginner/160_advanced-networking/secondary_cidr/)
* [EKS supports additional VPC CIDR blocks](https://aws.amazon.com/about-aws/whats-new/2018/10/amazon-eks-now-supports-additional-vpc-cidr-blocks/)
* [CNI metrics helper](https://docs.aws.amazon.com/eks/latest/userguide/cni-metrics-helper.html)
* [VPC Flow Logs to capture EKS communication](https://aws.amazon.com/blogs/networking-and-content-delivery/using-vpc-flow-logs-to-capture-and-query-eks-network-communications/)
* [Examine EKS cluster networking](https://aws.amazon.com/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
* [Troubleshooting CNI issues](https://aws.amazon.com/premiumsupport/knowledge-center/eks-cni-plugin-troubleshooting/)
* [Using multiple CIDR ranges with EKS](https://aws.amazon.com/premiumsupport/knowledge-center/eks-multiple-cidr-ranges/)

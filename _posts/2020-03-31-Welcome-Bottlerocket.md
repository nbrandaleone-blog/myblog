---
layout: post
title:  "Welcome Bottlerocket!"
date:   2020-03-31 08:00:00 -0400
comments: true
categories: kubernetes
---
![Bottlerocket logo](/images/bottlerocket.jpg)

## What is Bottlerocket?

Bottlerocket is a free and open-source Linux-based operating system meant for hosting containers. Bottlerocket focuses on security and maintainability, providing a reliable, consistent, and safe platform for container-based workloads. 

### Alternatives
Bottlerocket is somewhat unique in the container ecosystem. It is not quite a microVM like [firecracker](https://aws.amazon.com/blogs/aws/firecracker-lightweight-virtualization-for-serverless-computing/), but it is signficantly different than a typical Linux distro. Bottlerocket may be considered as a replacement for customers currently running *CoreOS Container Linux*, which will reach its end of life on May 26, 2020. There is also an open-source fork of CoreOS, knows as **flatcar**. Flatcar has its own [website](https://www.flatcar-linux.org), and the project is being supported by [Kinvolk](https://kinvolk.io).

## Philosophy behind Bottlerocket
The base operating system has just what you need to run containers reliably, and is built with standard open-source components. Bottlerocket-specific additions focus on reliable updates and on the API. Instead of making configuration changes manually, you can change settings with an API call, and these changes are automatically migrated through updates.

### *Translation*
The OS is meant to be immutable, or as immutable as possible.  You do not `yum update` packages, you update the entire OS at once. Therefore, there are available tools to easily self-update the entire OS. So, while you gain a minimally sized OS (small attack surface), with all sorts of cool security goodies, you may need to upgrade the OS even more often than required by an Orchestrator (i.e. Kubernetes fleets are usually updated at least once every three months).

## Core components of Bottlerockets
* Minimal OS that includes the Linux kernel (5.4), system software, and containerd as the container runtime.
* Atomic update mechanism to apply and rollback OS updates in a single step. 
* Integrations with container orchestrators such as Amazon EKS to manage and orchestrate updates.
* Control container which runs outside of the orchestrator in a separate instance of containerd. This container runs the AWS SSM agent that lets you run commands, or start shell sessions, on Bottlerocket instances in EC2.
* Admin container that can be optionally run for advanced troubleshooting and debugging.

# Demonstration
The easiest way to run Bottlerocket is to use `eksctl`. This is the command tool which was built to specifically manage EKS clusters. In the latest release (0.15.0), there is built-in support for Bottlerocket.

## Launch a cluster
``` shell
eksctl create cluster -f bottlerocket.yaml
```

where the config files looks like this:
``` shell
$ cat bottlerocket.yaml
# A simple example of ClusterConfig object with Bottlerocket settings:
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: bottlerocket
  region: us-west-2
  version: '1.15'

vpc:
  id: "vpc-f31a4797"
  cidr: "172.31.0.0/16"
  subnets:
    public:
      us-west-2a:
        id: "subnet-da32a9ac"
        cidr: "172.31.32.0/20"
      us-west-2b:
        id: "subnet-b1f0b3d5"
        cidr: "172.31.16.0/20"
      us-west-2c:
        id: "subnet-fd5ef8a5"
        cidr: "172.31.0.0/20"

nodeGroups:
  - name: ng1-public
    instanceType: m5.xlarge
    desiredCapacity: 2
    amiFamily: Bottlerocket
    ami: auto-ssm
    labels:
      "network-locality.example.com/public": "true"
    bottlerocket:
      enableAdminContainer: true
      settings:
        motd: "Hello, eksctl!"

  - name: ng2-public-ssh
    instanceType: m5.xlarge
    desiredCapacity: 2
    amiFamily: Bottlerocket
    ami: auto-ssm
    ssh:
      # Enable ssh access (via the admin container)
      allow: true
      publicKeyName: aws-key
```

### Update CNI
After the cluster comes up, you are advised to update the Container Network Interface to the latest version.

``` shell
$ kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/release-1.6/config/v1.6/aws-k8s-cni.yaml

$ kubectl describe daemonset aws-node --namespace kube-system | grep Image | cut -d "/" -f 2
amazon-k8s-cni:v1.6.0
```

## Test, test, test
I ran many basic workloads (i.e. nginx, redis, small web-servers), and I did not run into any issues at all.  Kubernets/EKS talked to the nodes without a problem.

### Verifying the worker nodes are running Bottlerocket

```shell
$ kubectl get nodes -o wide
NAME                                          STATUS   ROLES    AGE     VERSION    INTERNAL-IP     EXTERNAL-IP      OS-IMAGE                KERNEL-VERSION   CONTAINER-RUNTIME
ip-172-31-22-96.us-west-2.compute.internal    Ready    <none>   4h36m   v1.15.10   172.31.22.96    34.222.151.220   Bottlerocket OS 0.3.1   5.4.16           containerd://1.3.3+unknown
ip-172-31-26-118.us-west-2.compute.internal   Ready    <none>   4h36m   v1.15.10   172.31.26.118   34.222.122.20    Bottlerocket OS 0.3.1   5.4.16           containerd://1.3.3+unknown
ip-172-31-3-151.us-west-2.compute.internal    Ready    <none>   4h36m   v1.15.10   172.31.3.151    34.211.122.61    Bottlerocket OS 0.3.1   5.4.16           containerd://1.3.3+unknown
ip-172-31-4-202.us-west-2.compute.internal    Ready    <none>   4h36m   v1.15.10   172.31.4.202    54.149.42.253    Bottlerocket OS 0.3.1   5.4.16           containerd://1.3.3+unknown
```

Here is snapshot of my Bottlerocket cluster, using one of my favorite tools, `kube-ops-view`, by [Henning Jacobs](https://github.com/hjacobs/kube-ops-view) of [Zalando](https://zalando.com). The documentation is [here](https://kubernetes-operational-view.readthedocs.io/en/latest/).

![dashboard of running pods](/images/kov.png)

### Logging into Bottlerocket
Since it is very hard to login to a Bottlerocket instance, I thought I would show you how to do it. There are two ways to do it:

1. Use AWS SSM [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) to login into the nodes. For security reasons, you must add the IAM policy `AmazonSSMManagedInstanceCore` to the instance you wish to access (update appropriate instance role). I also prefer to use the MacOS [plug-in](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html#install-plugin-macos) when working with SSM.
2. SSH into the Admin container of Bottlerocket. From here you can `su`, or the Bottlerocket equivalent: `sudo sheltie`. This gives you a full root shell. For security reasons, this shell has to be enabled from the Control container, or configured during boot-up. I did so for node group `ng-2` in my `eksctl` config file.

#### Using SSM/Session Manager to login into a **control** container
``` shell
$ aws ssm start-session --target i-053e9d0a9018175e3

Starting session with SessionId: nick-macpro-023bcf5f7f19b20bc
Welcome to Bottlerocket's control container!

This container gives you access to the Bottlerocket API, which in turn lets you
inspect and configure the system.  You'll probably want to use the `apiclient`
tool for that; for example, to inspect the system:

   apiclient -u /settings

You can run `apiclient --help` for usage details, and check the main
Bottlerocket documentation for descriptions of all settings and examples of
changing them.

If you need to debug the system further, you can enable the admin container.
This enables SSH access to the system using the key you specified when you
launched the instance.  This environment has more debugging tools installed,
and allows you to get root access to the host.

To enable the admin container, run:

   enable-admin-container

[ssm-user@ip-172-31-26-118 /]$ exit
exit


Exiting session with sessionId: nick-macpro-023bcf5f7f19b20bc.
```

#### Logging into the **admin** container via SSH
``` shell
$ ssh -i ~/.ssh/aws-key.pem ec2-user@34.222.151.220
The authenticity of host '34.222.151.220 (34.222.151.220)' can't be established.
ECDSA key fingerprint is SHA256:/1oFWYxpGkHSvGJISqrJubiMSlbEG3Dl6dLU/DS2iYE.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '34.222.151.220' (ECDSA) to the list of known hosts.
Welcome to Bottlerocket's admin container!

This container provides access to the Bottlerocket host filesystems (see
/.bottlerocket/rootfs) and contains common tools for inspection and
troubleshooting.  It is based on Amazon Linux 2, and most things are in the
same places you would find them on an AL2 host.

To permit more intrusive troubleshooting, including actions that mutate the
running state of the Bottlerocket host, we provide a tool called "sheltie"
(`sudo sheltie`).  When run, this tool drops you into a root shell in the
Bottlerocket host's root filesystem.
[ec2-user@ip-172-31-22-96 ~]$ sudo sheltie
bash-5.0# ls
bin   etc   lib64	media  proc  sbin  tmp	x86_64-bottlerocket-linux-gnu
boot  home  local	mnt    root  srv   usr
dev   lib   lost+found	opt    run   sys   var
bash-5.0# ls etc
chrony.conf  host-containerd  kubernetes   mtab		  resolv.conf  wicked
cni	     host-containers  ld.so.cache  nsswitch.conf  selinux
containerd   hosts	      ld.so.conf   os-release	  shadow
group	     iproute2	      machine-id   passwd	  systemd
gshadow      issue	      motd	   pki		  updog.toml
bash-5.0# exit
exit
[ec2-user@ip-172-31-22-96 ~]$ logout
```

What is interesting, is that neither the control or admin containers show up in the `pod` view of the managing EKS cluster. These containers run outside of the kubelet and kubernetes.

## Updating Bottlerocket
Since it is not possible to update packages, Bottlerocket has a tool to assist with the potentially frequent updates of the OS. Fortunately, there is an `operator` for EKS/Kubernetes.  Information on the tool can be found in its Github [repo](https://github.com/bottlerocket-os/bottlerocket-update-operator). I have not used it, so I can't address its benefits or drawbacks.  For now, I think that I will simply update the node groups, like any other kubernetes worker nodes. Perhaps AWS will make Bottlerocket a possible default image for the **managed worker node** feature?!

### Summary
I quickly explored setting up an EKS cluster using Bottlerocket as the container OS for the worker nodes.  Bottlerocket is a highly secure container OS, with minimal size and attack surface.  

Bottlerocket is in Preview, and should **not** be used for production workloads yet.  It is available in the following regions:
* us-east-1
* us-west-2
* eu-central-1
* ap-northeast-1

![Bottlerocket logo](/images/bottlerocket.png)

### References

* [Bottlerocket blog post](https://aws.amazon.com/blogs/aws/bottlerocket-open-source-os-for-container-hosting/)
* [Bottlerocket Engineering Considerations](https://aws.amazon.com/blogs/containers/bottlerocket-a-special-purpose-container-operating-system/)
* [Bottlerocket FAQ](https://aws.amazon.com/bottlerocket/faqs/)
* [Bottlerocket Github Repo](https://github.com/bottlerocket-os/bottlerocket)
* [Bottlerocket Quickstart](https://github.com/bottlerocket-os/bottlerocket/blob/develop/QUICKSTART.md)
* [Bottlerocket Roadmap](https://github.com/orgs/bottlerocket-os/projects/1)
* [Sample EKSCTL config](https://github.com/weaveworks/eksctl/blob/master/examples/20-bottlerocket.yaml)
* [Weaveworks GitOps with Bottlerocket](https://www.weave.works/blog/bottlerocket-with-fork-clone-run)

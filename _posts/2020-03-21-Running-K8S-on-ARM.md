---
layout: post
title:  "Running K8s on ARM"
date:   2020-03-21 08:00:00 -0400
comments: true
categories: kubernetes
---
![graviton performance](/images/aws-graviton-spec-3.png)

## Running Kubernetes on ARM processors

ARM processors may be the future of computing.  They are based upon a RISC architecture, which offers lower power consumption, improved heat dissipation, and favorable performance/vCPU benefits.  Until now, they have been seen mostly in mobile devices, smart-phones, IoT (i.e. Rasberry Pis), tablets and other embedded systems.

However, ARM processors are about to go mainstream. There are compelling reasons to believe that Apple will release ARM based laptops later this year.  AWS has already released its first generation of ARM based servers (A1 class), but announced its second generation roll-out at [ReInvent](https://www.zdnet.com/article/aws-graviton2-what-it-means-for-arm-in-the-data-center-cloud-enterprise-aws/) 2019.  It is possible to now [sign-up](https://pages.awscloud.com/m6gpreview.html) for the Preview for these Graviton2 processors, known as the EC2 M6g, C6g, R6g series. Please read about the announcement [here](https://aws.amazon.com/about-aws/whats-new/2019/12/announcing-new-amazon-ec2-m6g-c6g-and-r6g-instances-powered-by-next-generation-arm-based-aws-graviton2-processors/).

## But can I run Kubernetes on it?

Yes, you can. In fact, I will demonstrate two ways of running Kubernetes on A1 instances:

1. Rancher [K3s](https://k3s.io) - a lightweight Kubernetes distribution, which can run on ARM processors.
2. A BETA/Preview version of [EKS](https://docs.aws.amazon.com/eks/latest/userguide/arm-support.html) for ARM.

The current A1 processors are somewhat underpowered compared to their Intel counterparts. Therefore, I decided to install K3s. As mentioned above, K3s is a stripped down version of Kubernetes, yet fully compliant. It can be run in a HA mode, also making it a reasonable choice for production workloads (if you still want to manage your own control plane). While I won't go into what was ripped out in order to make K3s, I will mention it was over 3 million lines of Go code.  What do you get for all this slimmed-down goodness? A control plane that will launch in about 30 seconds.  It is pretty amazing.  See this KubeCon [talk](https://www.youtube.com/watch?v=-HchRyqNtkU) by Rancher Labs chief architect, Darren Sheperd for more details.

I will mention three items of interest that were removed in order to get the kubernetes binary down to about 50 MB.
- All Cloud integrations
- The ETCD database
- K3s uses `containerd` as the runtime, instead of Docker

Item number one means that typical integrations with Cloud Providers like LoadBalancers will not work.  They must be set-up manually. Item number 2 is simply interesting.  The traditional ETCD database was replaced with SQLite, and performance is is apparently fine, even under heavy load. I will mention due to customer demand, it is very likely that K3s will support Cloud Provider integration once again in the near future. Containerd is the future of container runtimes, and it is nice to see it included in this distribution.

## K3s installation

![K3s architecture](/images/how-it-works-k3s.svg)

I decided to run launch x3 a1.large instances for this demo. Each A1 instance costs about 5 cents/hour to run. I used the latest Amazon Linux 2 AMI, but checked the ARM radio button on the right to ensure the proper image. Also, ensure you install an SSH key, since we will leverage SSH for the installation process.

![ARM toggle box](/images/arm-toggle.png)

### Verify that I am running on an ARM processor

``` shell
[ec2-user@ip-172-31-2-154 ~]$ uname -a
Linux ip-172-31-2-154.us-west-2.compute.internal 4.14.171-136.231.amzn2.aarch64 #1 SMP Thu Feb 27 20:25:45 UTC 2020 aarch64 aarch64 aarch64 GNU/Linux
```

You should notice the *aarch64* tag in the kernel build.

> AArch64 is the 64-bit execution state of the ARMv8 ISA. A machine in this state executes operates on the A64 instruction set. This is in contrast to the AArch32 which describes the classical 32-bit ARM execution state.

I installed on my local laptop a small [program](https://github.com/alexellis/k3sup) used to automate K3s installations. It is called **k3sup**, but pronounced *ketchup*, and it made an easy process, even smoother.

![architecture of the k3sup program](/images/k3sup-cloud.png)

### Install k3sup

``` shell
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

k3sup --help
```

### Create the K3s master using k3sup

We need to set some variables in order to create the K3s cluster. Grab the public IP addresses of the three **a1** instances.

``` shell
$ export SERVER_IP=35.167.64.185 # public IP of my master node
$ k3sup install --ip $SERVER_IP --ssh-key ~/.ssh/aws-key.pem --user ec2-user

Running: k3sup install
Public IP: 35.167.64.185
ssh -i /Users/nbrand/.ssh/aws-key.pem -p 22 ec2-user@35.167.64.185
ssh: curl -sLS https://get.k3s.io | INSTALL_K3S_EXEC='server  --tls-san 35.167.64.185 ' INSTALL_K3S_VERSION='v1.17.2+k3s1' sh -

[INFO]  Using v1.17.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
chcon: can't apply partial context to unlabeled file ‘/usr/local/bin/k3s’
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s.service to /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s
Result: [INFO]  Using v1.17.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
chcon: can't apply partial context to unlabeled file ‘/usr/local/bin/k3s’
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
[INFO]  systemd: Starting k3s
...
```

The kubeconfig file has been saved to my local directory. Let's test it out.

``` shell
$ export KUBECONIFG=`pwd`/kubeconfig
$ kubectl get nodes

NAME                                         STATUS   ROLES    AGE    VERSION
ip-172-31-2-154.us-west-2.compute.internal   Ready    master   3m5s   v1.17.2+k3s1
```

### Adding worker nodes

Adding worker nodes is also a single line command. 

``` shell
export AGENT_IP1=54.245.73.14    # One must use the public IP addresses of the worker nodes
export AGENT_IP2=18.237.149.127

$ k3sup join --ip $AGENT_IP1 --ssh-key ~/.ssh/aws-key.pem --server-ip $SERVER_IP --user ec2-user
Running: k3sup join
Server IP: 35.167.64.185
ssh -i /Users/nbrand/.ssh/aws-key.pem -p 22 ec2-user@35.167.64.185
ssh: sudo cat /var/lib/rancher/k3s/server/node-token

K104c6eeaf792808815cef85acd5098ed8c24d580037817419f55f20b9898dae187::server:4296b46737167777328441d6a6e0f5c0
ssh: curl -sfL https://get.k3s.io/ | K3S_URL='https://35.167.64.185:6443' K3S_TOKEN='K104c6eeaf792808815cef85acd5098ed8c24d580037817419f55f20b9898dae187::server:4296b46737167777328441d6a6e0f5c0' INSTALL_K3S_VERSION='v1.17.2+k3s1' sh -s - 
[INFO]  Using v1.17.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
chcon: can't apply partial context to unlabeled file ‘/usr/local/bin/k3s’
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink from /etc/systemd/system/multi-user.target.wants/k3s-agent.service to /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent
Logs: Created symlink from /etc/systemd/system/multi-user.target.wants/k3s-agent.service to /etc/systemd/system/k3s-agent.service.
Output: [INFO]  Using v1.17.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/sha256sum-arm64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.17.2+k3s1/k3s-arm64
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
chcon: can't apply partial context to unlabeled file ‘/usr/local/bin/k3s’
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
[INFO]  systemd: Starting k3s-agent
```

Do the same for the last worker node:

``` shell
k3sup join --ip $AGENT_IP2 --ssh-key ~/.ssh/aws-key.pem --server-ip $SERVER_IP --user ec2-user
...
```

### Test out the cluster
``` shell
$ kubectl get nodes

NAME                                         STATUS   ROLES    AGE     VERSION
ip-172-31-2-13.us-west-2.compute.internal    Ready    <none>   4m17s   v1.17.2+k3s1
ip-172-31-2-154.us-west-2.compute.internal   Ready    master   16m     v1.17.2+k3s1
ip-172-31-13-6.us-west-2.compute.internal    Ready    <none>   18s     v1.17.2+k3s1
```

### Install a workload
I will create a simple web workload on the cluster. While K3s does not support Cloud load balancers (they can still be created manually), it does support the software [Traefik](https://docs.traefik.io/v1.3/) load balancer by default.  This means that we can hit any of the nodes on port 80, and the Traefik load balancer which is running as a daemon set, will send traffic to the proper pods. I altered the default port of nginx to 8080, since I ran into a port conflict with the software load balancer. If I had more experience with Traefik I may be able to configure around this issue. For now, this was a straightforward solution to implement.

``` shell
$ cat web.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysite-html
  namespace: default
data:
  index.html: |
    <html>
    <head><title>K3S!</title>
      <style>
        html {
          font-size: 62.5%;
        }
        body {
          font-family: sans-serif;
          background-color: midnightblue;
          color: white;
          display: flex;
          flex-direction: column;
          justify-content: center;
          height: 100vh;
        }
        div {
          text-align: center;
          font-size: 8rem;
          text-shadow: 3px 3px 4px dimgrey;
        }
      </style>
    </head>
    <body>
      <div>Hello from K3S!</div>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-kubernetes
  labels:
    app: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
      - name: hello
        image: nginxinc/nginx-unprivileged
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: mysite-html
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
    - protocol: TCP
      port: 8080
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: hello-service
          servicePort: 8080
---

$ kubectl create -f web.yaml
configmap/mysite-html created
deployment.apps/hello-kubernetes created
service/hello-service created
ingress.networking.k8s.io/web-ingress created
```

### Test
Here is a picture of the web-site.  It works from any of the public IP addresses of any node.

![test web workload](/images/web-traefik.png)

## EKS and ARM
EKS also supports running on the a1 instance class. It is currently in Preview/Beta, so there are more steps than normal. The directions are [here](https://docs.aws.amazon.com/eks/latest/userguide/arm-support.html).

1. Launch a cluster *without* a nodegroup.
2. Upgrade all relevant components to ARM variants:
    - DNS
    - Kube-Proxy
    - CNI
3. Launch an ARM nodegroup, and attach it to the control plane.

The process is too long for this blog post, but straightforward. Please give it a try.

## Downsides?
The obvious downside is that all compiled programs will not work until they have been re-compiled for an ARM processor. Since `Go`, which is a compiled language, is very strong in the Kubernetes ecosystem (as K8s is written in Go), you will find that many pods or public containers simply do not work.

Fortunately, most interpreted languages work without a problem (i.e. node, python, ruby, etc...). This assumes that the base image for the interpreter has been properly compiled.  Also, most of the official images on Docker Hub supports a variety of architectures. For example, the busybox image supports amd64, arm32v5, arm32v6, arm32v7, arm64v8, i386, ppc64le, and s390x. When running this image on an x86_64 / amd64 machine, the x86_64 variant will be pulled and run. This is why nginx (which is written in C), works fine.

At present, [ECR](https://aws.amazon.com/ecr/) does not support multi-architecture [builds](https://github.com/aws/containers-roadmap/issues/505).  This is a minor issues, but one you should be aware of.

### Summary
This post goes over how easy it is to run Kubernetes on ARM processors in the cloud. I demonstrated an impressive lightweight Kubernetes distribution (K3s), which supports ARM out of the box.  I also gave links to how to run EKS on ARM processors.  When the [Graviton2](https://aws.amazon.com/ec2/graviton/) processors hit, it may be worth while to investigate these new chips as a platform for a cost-effective but powerful Kubernetes cluster.

In the meantime, if you need to save money on your cluster, it is easier to use Spot, or Reserved Instances. Also, if you run your virtual machines with a lot of spare capacity, [Fargate](https://aws.amazon.com/fargate/) may turn out to be more cost-effective than you realized.

![ARM chip image](/images/Arm-Neoverse.jpg)

### References

* [AWS Graviton2](https://aws.amazon.com/blogs/aws/coming-soon-graviton2-powered-general-purpose-compute-optimized-memory-optimized-ec2-instances/)
* [k3s](https://k3s.io)
* [Zero to k3s Kubeconfig in seconds with k3sup](https://rancher.com/blog/2019/k3s-kubeconfig-in-seconds/)
* [k3sup](https://github.com/alexellis/k3sup)
* [EKS and ARM](https://aws.amazon.com/about-aws/whats-new/2019/04/-amazon-eks-supports-ec2-a1-instances-as-a-public-preview-/)
* [Building multi-architecture Docker images](https://collabnix.com/building-arm-based-docker-images-on-docker-desktop-made-possible-using-buildx/)
* [video of K3s and Traefik](https://www.youtube.com/watch?v=QcC-5fRhsM8)

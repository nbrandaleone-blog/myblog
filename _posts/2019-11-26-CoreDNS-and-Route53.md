---
layout: post
title:  "CoreDNS and Route53"
date:   2019-11-26 08:00:00 -0400
categories: aws eks dns
---
![coreDNS logo](/images/coredns.png)

A Kubernetes cluster is isolated by default.  The pods may (or may not) be using an overlay network IP scheme, the Cluster IPs may be using yet another CIDR block, and DNS is entirely local to the cluster.  So, how pray tell, do you interact with such an isolated beast?

Well, for _services_ you use Cloud load-balancers which direct traffic to the pods indirectly via NodePorts.  For DNS, one typically uses the [External DNS](https://github.com/kubernetes-sigs/external-dns) tool to export local Kubernetes names to global DNS names.

![external DNS logo](/images/external-dns.png)

Still, this does not solve all your DNS issues. External DNS only **exports** you name information.  What if you want to **import** information, say from a private zone (thinking AWS here)?!

Well, fortunately - there is a solution!  [CoreDNS](https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/), the default DNS provider for Kubernetes (it replaced KubeDNS in version 1.11) has a route53 plug-in. This plug-in will periodically poll your local Route53 private or public zones, and make all entries locally available to your Kubernetes/EKS cluster. This is very useful if your cluster needs to access resources within your VPC, which are private.

The other option is to use AWS [CloudMap](https://aws.amazon.com/cloud-map/). Cloud Map provides service discovery for compute resources within your VPC. Check out this [blog](https://aws.amazon.com/blogs/containers/cross-amazon-eks-cluster-app-mesh-using-aws-cloud-map/) post, which leveraged it for EKS to EKS communication using App Mesh (a type of servive mesh).  Pretty cool stuff!  :-)

## Configuring CoreDNS
The settings for CoreDNS are handled via a configmap, which lives in the kube-system namespace.  We must edit this CM in order to add the route53 plugin.0:w


### Original ConfigMap
``` yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"Corefile":".:53 {\n    errors\n    health\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      pods insecure\n      upstream\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"eks.amazonaws.com/component":"coredns","k8s-app":"kube-dns"},"name":"coredns","namespace":"kube-system"}}
  creationTimestamp: "2019-11-19T22:11:56Z"
  labels:
    eks.amazonaws.com/component: coredns
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
  resourceVersion: "140889"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: 997034b1-0b19-11ea-89c2-06bc2ec33374
```

### New ConfigMap
``` yaml
apiVersion: v1
data:
  Corefile: |
    nick.test:53 {
      route53 nick.test.:Z0699699UVAMCRGMLWSE {
      aws_access_key <access_key> <secret accress key>
      }
    }
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"Corefile":".:53 {\n    errors\n    health\n    kubernetes cluster.local in-addr.arpa ip6.arpa {\n      pods insecure\n      upstream\n      fallthrough in-addr.arpa ip6.arpa\n    }\n    prometheus :9153\n    forward . /etc/resolv.conf\n    cache 30\n    loop\n    reload\n    loadbalance\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"eks.amazonaws.com/component":"coredns","k8s-app":"kube-dns"},"name":"coredns","namespace":"kube-system"}}
  creationTimestamp: "2019-11-19T22:11:56Z"
  labels:
    eks.amazonaws.com/component: coredns
    k8s-app: kube-dns
  name: coredns
  namespace: kube-system
  resourceVersion: "153512"
  selfLink: /api/v1/namespaces/kube-system/configmaps/coredns
  uid: 997034b1-0b19-11ea-89c2-06bc2ec33374
  ```

You may have noticed that I had to put an access key/secret access key into the configuration.  The manuals states that you should be able to use the default credential provider available to the node (or perhaps IRSA), but I was not able to get it working in the small amount of time I dedicated to this project.

## Create a fargate task in ECS
```
$ fargate task run nginx-dns-test -i nginx:latest --security-group-id sg-097c56c9dfd62802a --region us-west-2

$ fargate task ps nginx-dns-test --region us-west-2
ID					IMAGE		STATUS	RUNNING	IP	        CPU	MEMORY	
fb5f875a91904a3a88be2d24ed04b37f	nginx:latest	running	2m25s	172.31.52.184	256	512
```
_Note: I changed the public IP to the private one for clarity_

![Zone details](/images/route532.png)
k
## Create a Route53 private domain
I created a private domain for _nick.test_. I then created an **A** record for the fargate task I just created, calling the record _nginx.nick.test_.

![Route53 zone](/images/zone1.png)

## Test if the EKS cluster can resolve the private Route53 zone

```bash
$ kubectl run dnstools --generator=run-pod/v1 --image=infoblox/dnstools -it

$ dig -t a nginx.nick.test. 
; <<>> DiG 9.8.3-P1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 60796
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
...
;nginx.nick.test.            IN    A
;; ANSWER SECTION:
nginx.nick.test.        3600    IN    A    172.31.52.184
...
```

Succcess!!  I have shown that it is possible to import Route53 zones into your Kubernetes cluster. This will allow for your cluster to access other compute resources within your VPC by name.  This gives you yet another option for service discovery using DNS.  Remember, you can also use Cloud Map as well.

While I did not show direct connectivity, I tested it and it worked perfectly. One of the advantages of using EKS, is that all the pods use VPC IP addresses by default.  This makes IP connectivity within the VPC trivial.

***

### References

* [CoreDNS](https://coredns.io/)
* [CoreDNS and Kubernetes](https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/)
* [CoreDNS and Route53 plugin](https://github.com/coredns/coredns/tree/master/plugin/route53)
* [CoreDNS book](https://www.oreilly.com/library/view/learning-coredns/9781492047957/)

---
layout: post
title:  "EKS and Horizontal Pod Autoscaler"
date:   2018-07-03 09:00:00 -0400
categories: aws
---
<details>
<summary><strong>HPA is now working with EKS!</strong></summary>
On August 31, 2018 - according to this <a href="https://aws.amazon.com/blogs/opensource/horizontal-pod-autoscaling-eks/">blog</a> HPA is now working with EKS. I have tested it, and it is indeed true.  However, you must manually install the metrics-server, which is a bit of a bummer.  However, just follow these <a href="https://github.com/kubernetes-incubator/metrics-server">instructions</a>, and you will be able to auto-scale with the best of them.  

Follow the <a href="https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/">kubernetes tutorial</a> to test out HPA.

<img src="/images/hpa-diagram-2.jpg" alt="diagram">

Go AWS!
</details>

![Horizontal Pod Autoscaler](/images/autoscaler_kubernetes.jpg)
The Horizontal Pod Autoscaler automatically scales the number of pods in a deployment or replica set. HPA typically works off of CPU utilization, but it is now possible to utilize custom metrics for scaling.

The Horizontal Pod Autoscaler feature was first introduced in Kubernetes v1.1 and has evolved a lot since then. Version 1 of the HPA scaled pods based on observed CPU utilization and later on based on memory usage. In Kubernetes 1.6 a new API Custom Metrics API was introduced that enables HPA access to arbitrary metrics. And Kubernetes 1.7 introduced the aggregation layer that allows 3rd party applications to extend the Kubernetes API by registering themselves as API add-ons. 

The Horizontal Pod Autoscaler is implemented as a control loop that periodically queries the Resource Metrics API for core metrics like CPU/memory and the Custom Metrics API for application-specific metrics.

![HPA control loop](/images/k8s-hpa-ms.png)

### HPA on EKS
Unfortuntely, the *Metrics Server* is not an approved add-on for EKS at this time. So, it is **NOT** possible to run HPA on an EKS cluster out of the box.  It may be possible to *hack* it, but I have not had success.  Since heapster is supported, it also may be possible to use an older version of HPA instead of the more recent releaseses. Since AWS typically moves quickly on features for new products, I would expect and hope to see metrics server available soon...

___
**References:**

[HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

[Metrics Server](https://blog.freshtracks.io/what-is-the-the-new-kubernetes-metrics-server-849c16aa01f4)

[Metrics API](https://blog.jetstack.io/blog/resource-and-custom-metrics-hpa-v2/)

[HPA with Prometheus](https://github.com/stefanprodan/k8s-prom-hpa)

[EKS documentation - html](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) or
 [EKS documentation - PDF](https://docs.aws.amazon.com/eks/latest/userguide/eks-ug.pdf)

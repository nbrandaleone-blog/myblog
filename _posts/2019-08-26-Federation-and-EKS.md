---
layout: post
title:  "Federation v2 and EKS"
date:   2019-08-26 08:00:00 -0400
categories: aws elixir
---
![Scheduler Diagram](/images/k8-scheduler.png)

The default Kubernetes scheduler is quite sophisticated. Under most circumstances, you should **not** muck with it. It is intelligent, performant and battle-tested.

In terms of K8 deployments on Cloud Providers, the scheduler is also Availability Zone aware (or simply "zones" in GCE parlance). This can be easily tested by creating a deployment with two or more pod replicas.  If you examine the placement, you will see that the pods are spread over the "zones". Kubernetes uses labels in order to understand the region and zone the worker nodes are located in.

## Kubernetes default scheduler:
```bash
$ kubectl apply -f https://k8s.io/examples/application/deployment.yaml

$ kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP
nginx-deployment-76bf4969df-m4szq   1/1     Running   0          73s   172.31.31.49
nginx-deployment-76bf4969df-zfsjn   1/1     Running   0          73s   172.31.13.159
```

## Now let's look at the node labels

```bash
NAME                                          STATUS   ROLES    AGE    VERSION              LABELS
ip-172-31-29-212.us-west-2.compute.internal   Ready    <none>   109m   v1.13.7-eks-c57ff8   alpha.eksctl.io/cluster-name=cluster0,alpha.eksctl.io/instance-id=i-008eeb529825fa431,alpha.eksctl.io/nodegroup-name=ng-1,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-west-2,failure-domain.beta.kubernetes.io/zone=us-west-2b,kubernetes.io/hostname=ip-172-31-29-212.us-west-2.compute.internal
ip-172-31-6-222.us-west-2.compute.internal    Ready    <none>   109m   v1.13.7-eks-c57ff8   alpha.eksctl.io/cluster-name=cluster0,alpha.eksctl.io/instance-id=i-02ae95c403058e42f,alpha.eksctl.io/nodegroup-name=ng-1,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=m5.xlarge,beta.kubernetes.io/os=linux,failure-domain.beta.kubernetes.io/region=us-west-2,failure-domain.beta.kubernetes.io/zone=us-west-2c,kubernetes.io/hostname=ip-172-31-6-222.us-west-2.compute.internal
```

If you look carefully, you will see the following labels associated with the nodes:

---

| **LABEL**                                    | **region** or **zone** |
| failure-domain.beta.kubernetes.io/**region** | *us-west-2* |
| failure-domain.beta.kubernetes.io/**zone** | *us-west-2c* |

---

## Using a custom scheduler
Now, even though I explained how good the default scheduler is, and even demonstrated that it is zone aware, it is not perfect.  For example. let's say I have two node-groups.  One group is "on demand" instances, and another node-group consists of Spot instances.  Perhaps I prefer for the scheduler to use the "Spot" instances first.  If they are filled, only then use the "on demand" instances. This is actually quite tricky to do using the default scheduler. One way is to use an open-source project called [k8-spot-scheduler](https://github.com/pusher/k8s-spot-rescheduler).

Another way to accomplish this goal is to create your own custom scheduler, with the logic tuned to your requirements. We will create a crude scheduler in order to demonstrate how easy this really is.


### Let me see the code!

**Elixir code**
{% highlight elixir %}
def poll_all_instances() do
    instance_data =
      pmap(get_good_regions(), fn (region) ->
        EC2.describe_instances() |> ExAws.request(region: region) |> parse_instance_info() end)
      |> List.flatten

# .. Print out data
  end
{% endhighlight %}
___

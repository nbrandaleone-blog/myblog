---
layout: post
title:  "Custom Kubernetes Schedulers"
date:   2019-08-21 08:00:00 -0400
categories: aws elixir kubernetes
---
![Scheduler Diagram](/images/k8-scheduler.png)

The default Kubernetes scheduler is quite sophisticated. Under most circumstances, you should **not** muck with it. It is intelligent, performant and battle-tested.

In terms of K8 deployments on Cloud Providers, the scheduler is also Availability Zone aware (or simply "zones" in GCE parlance). This can be easily tested by creating a pod with an PVC. Since the Volume cannot cross AZs, if you create a second pod that requires the same PVC, the pod will be created in the original AZ.  Kubernetes uses labels in order to understand the region and zone the worker nodes are located in.

Normally, when you create two or more pods in a deployment, the scheduler will do its best to spread the pods over multiple worker nodes.  Let's do a quick test to confirm this.

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
Now, even though I explained how good the default scheduler is, and even demonstrated that it is zone aware, it is not perfect.  For example. let's say I have two node-groups.  One group is on-demand instances, and another node-group consists of Spot instances.  Perhaps I prefer for the scheduler to use the Spot instances first.  If they are filled, only then use the on-demand instances. This is actually quite tricky to do using the default scheduler. One way is to use an open-source project called [k8-spot-scheduler](https://github.com/pusher/k8s-spot-rescheduler). Another project worth mentioning is a [_descheduler_](https://github.com/kubernetes-incubator/descheduler). It allows for pods to be moved around if certain rules are violated (i.e. too many pods on the same host).

Another way to accomplish this goal is to create your own custom scheduler, with the logic tuned to your requirements. We will create a crude scheduler in order to demonstrate how easy this really is. The rest of this code is based upon a talk by Kelsey Hightower and his [talk](https://skillsmatter.com/skillscasts/7897-keynote-get-your-ship-together-containers-are-here-to-stay), from 2016.

We are going to write our new scheduleder in Elixir, based upon this [blog](http://agonzalezro.github.io/scheduling-your-kubernetes-pods-with-elixir.html) and this [code](https://github.com/agonzalezro/escheduler). I have updated the [code](https://github.com/nbrandaleone/escheduler), and posted it on Github. Why [elixir](https://elixir-lang.org/)? I like the power of functional programming, the sytax of Ruby, and the ability to create 100,000 of processes easily. It would not be a bad choice for a real scheduler, if you were so inclined. Of course, you can peruse the actual [Go](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/scheduler.go) code of the default scheduler as well.

## Find all pods that are unscheduled, and assigned to our scheduler

**Elixir code**
{% highlight elixir %}
def unscheduled_pods() do
    is_managed_by_us = &(get_in(&1, ["spec", "schedulerName"]) == @name)

    resp = HTTPoison.get! "http://127.0.0.1:8001/api/v1/pods?fieldSelector=spec.nodeName="
    resp.body
    |> Poison.decode!
    |> get_in(["items"])
    |> Enum.filter(is_managed_by_us)
    |> Enum.map(&(get_in(&1, ["metadata", "name"])))
end
{% endhighlight %}
As you can see, we are using `127.0.0.1:8001` to query our API. This is possible thanks to the `kubectl proxy` command which easily allows us to test out our new scheduler without first converting it to a pod, and worrying about Service Accounts and other details.

## Get a list of available nodes
This function simply returns a list of available nodes. We are not checking to see if the nodes have any spare capacity - just that they are available.

{% highlight elixir %}
def nodes() do
    resp = HTTPoison.get! "http://127.0.0.1:8001/api/v1/nodes"
    resp.body
    |> Poison.decode!
    |> get_in(["items"])
    |> Enum.map(&(get_in(&1, ["metadata", "name"])))
end
{% endhighlight %}

## The Bind Function
Once we have a list of unscheduled pods, and potential nodes to run them on - we must bind the pods to a node. We call the bind endpoint, like this:

{% highlight elixir %}
def bind(pod_name, node_name) do
    url = "http://127.0.0.1:8001/api/v1/namespaces/default/pods/#{pod_name}/binding"
    body = Poison.encode!(%{
      apiVersion: "v1",
      kind: "Binding",
      metadata: %{
        name: pod_name
      },
      target: %{
        apiVersion: "v1",
        kind: "Node",
        name: node_name
      }
    })
    headers = [{"Content-Type", "application/json"}]
    options = [follow_redirect: true]

    HTTPoison.post!(url, body, headers, options)
    IO.puts "#{pod_name} pod scheduled in #{node_name}"
end
{% endhighlight %}

## Scheduling Strategy
For our simple scheduler, we will go with a random scheduling strategy. However, this is where one could provide significant business value by altering the default strategy, in order to take into account Spot prices, for example.

{% highlight elixir %}
def schedule(pods) do
    pods
    |> Enum.each(&(bind(&1, Enum.random(nodes()))))
end
{% endhighlight %}

## Test it out
1. Create some pods or a Deployment with the spec having the following addition: `schedulerName: escheduler`.
2. Start up the _proxy_.
3. Run your custom scheduler: `$ ./escheduler`

### Before the scheduler is run, the pods are *Pending*
``` bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7d96855ffb-947n8   0/1     Pending   0          11s
nginx-7d96855ffb-qnhgh   0/1     Pending   0          11s
nginx-7d96855ffb-sbltl   0/1     Pending   0          11s
```

### Afterwards...
``` bash
NAME                     READY   STATUS    RESTARTS   AGE
nginx-7d96855ffb-947n8   1/1     Running   0          31s
nginx-7d96855ffb-qnhgh   1/1     Running   0          31s
nginx-7d96855ffb-sbltl   1/1     Running   0          31s
```

## The future of custom schedulers
As cool as it is to create your own scheduler, it is unlikely you could create a production quality one without a great deal of effort. Therefore, it is exciting news that in Kubernetes 1.15+ (alpha) that it will be possible to add extensions into the default Kubernetes scheduler.  This will truely be the best of all worlds, where we can rely upon a rock-solid scheduler, but extend it to meet our business needs.  Please read about it [here](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/)

![Scheduler Extension](/images/scheduling-framework-extensions.png)

***

### References
* [Kubernetes Scheduler](https://kubernetes.io/docs/concepts/scheduling/kube-scheduler/)
* [Scheduler Performance Tuning](https://kubernetes.io/docs/concepts/scheduling/scheduler-perf-tuning/)
* [Julia Evans - How the Kubernetes scheduler work?](https://jvns.ca/blog/2017/07/27/how-does-the-kubernetes-scheduler-work/)
* [Writing custom Kubernetes schedulers](https://banzaicloud.com/blog/k8s-custom-scheduler/)
* [Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/)
* [Writing a custom Kubernetes scheduler using monitoring metrics](https://sysdig.com/blog/kubernetes-scheduler/)
* [Implementing Advanced Scheduling Techniques with Kubernetes](https://thenewstack.io/implementing-advanced-scheduling-techniques-with-kubernetes/)
* [Scheduling Framework](https://kubernetes.io/docs/concepts/configuration/scheduling-framework/)
* [Kelsey's toy scheduler](https://github.com/kelseyhightower/scheduler/blob/master/kubernetes.go)

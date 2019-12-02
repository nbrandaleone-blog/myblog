---
layout: post
title:  "Federation v2 and EKS"
date:   2019-09-02 08:00:00 -0400
categories: aws kubernetes
---
![Scheduler Diagram](/images/fedv2.png)

There is a great deal of buzz around Kubernetes Federation v2.  While it is still an _alpha_ project, it shows great promise. It allows one to control multiple K8 clusters from a centralized master or **host** cluster. These **member** clusters can be in different regions, or even different cloud providers, or on-premise for that matter. Unlike v1 of Federation, the current release uses CRD's extensively, which has turned out to be the most popular and stable way of extending the K8 API.

### Now, why would you want to use Federation?!
1. High Availability / Disaster Recovery
2. Hybrid on-premise / cloud architecture
3. Avoid vendor lock-in
4. Isolating sensitive workloads
5. Centralized control

To be honest, none of the reasons above are entirely compelling for the majority of Kubernetes users.  Kubernetes is open-sourced and already vendor neutral. The Kubernetes control-plan is self-healing, and if run in a multi-AZ/zone as dictated by best-practices, you are already highly available.  However, I do recognize that some enterprises will want a multi-region or multi-cloud designs for both technical and _political_ reasons. For those customers - **good** news! Federation v2 certainly works (albeit still in alpha), and there are numerous other options as well.

## Federation Concepts
The recommended way of working with Federated resources or objects is to work within a scoped namespace. It is also possible to work with a cluster-wide scoping. In either case, you create a namespace within your **host** cluster, and a set of federated resources. Any federated resource will be replicated out to your member clusters, usually within a few seconds (no more than 20 s). The replicated federated resource is converted into a standard resource. For example, a federated deployment is converted into a deployment.  A federated namespace is converted into a regular namespace, and so on...

The type of Federated resource that can be mirrored depends upon if there is CRD for that resource.  If so, and it has been enabled on the host cluster, then it will automatically watch for the corresponding type. [**KubeFed**](https://github.com/kubernetes-sigs/kubefed) (short for Kubernetes Cluster Federation v2) is configured with two types of information:
* **Type configuration** declares which API types KubeFed should handle
* **Cluster configuration** declares which clusters KubeFed should target

Type configuration has three fundamental concepts:
* **Teplates** define the representation of a resource common across clusters
* **Placement** defines which clustesr the resources is intended to appear in
* **Overrides** define per-cluster field-level variation to apply to the template

![KubeFed concepts](/images/fed-concepts.png)

## Federated Deployment Type Example
```bash
piVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test-deployment
  namespace: test
spec:
  template:
    << DEPLOYMENT SPEC >>
  placement:
    clusters:
    - name: cluster2
    - name: cluster1
    clusterSelector:
      matchLabels:
        region: eu-west-1
  overrides:
  - clusterName: cluster2
    clusterOverrides:
    - path: spec.replicas
      value: 2
```

## Running Federation v2 on EKS
Even though Federation is an alpha feature, it can be run in EKS today.  This is because it does not rely on the master control-plane for anything special. All components can be run on the worker nodes as standard pods/deployments/CRD, etc...

## Installation Process
The installation process is fairly well [documented](https://github.com/kubernetes-sigs/kubefed/blob/master/docs/userguide.md), but there are a few wrinkles specific to EKS.   The KubeFed control plane can run on any v1.13 or greater Kubernetes clusters. Let's go over the details. The following steps should occur only on the master or host cluster.  This cluster will control all the others. There is no configuration that has to be done on the member clusters - all setup is driven by the host.

I created three clusters, using [eksctl.io](https://eksctl.io/). I named them cluster0, cluster1 and cluster2.  They live in 3 different regions, and cluster0 is acting as the host. The architecture will look similar to this following diagram.

![KubeFed network diagram](/images/fed-demo.png)

### Helm Chart Deployment
Since EKS has RBAC enabled, let [create](https://github.com/kubernetes-sigs/kubefed/blob/master/charts/kubefed/README.md) a service account with the permissions necessart to deploy KubeFed

```bash
$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF

$ helm init --service-account tiller
```

### Installing the Chart
First, add the KubeFed chart repo to your local repository. Then, install the chart.

```bash
$ helm repo add kubefed-charts https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts

$ helm repo list
NAME            URL
kubefed-charts   https://raw.githubusercontent.com/kubernetes-sigs/kubefed/master/charts

$ helm install kubefed-charts/kubefed --name kubefed --version=0.1.0-rc6 --namespace kube-federation-system
```

There are instructions on how to delete the chart when you are [done](https://github.com/kubernetes-sigs/kubefed/blob/master/charts/kubefed/README.md).  I must warn you that all CR and CRD must be deleted, or else you will create problems for yourself.

### Prepare to Join clusters into the Federation
A separate tool must be downloaded and installed, in order to join the clusters together. The tool is called _kubefedctl_. It can be obtained from [github](https://github.com/kubernetes-sigs/kubefed/releases), and should match the version of the helm chart (at this time v0.1.0-rc6).

Now, this is where I ran into a little bit of trouble.  This tool, uses your _KUBECTL_ config file in order to be properly authenticated into the other member clusters. Apparently it parses the file, and the format that EKS uses to generate the YAML did not parse properly.  So, I manually adjusted my KUBECTL file to eliminate all offending characters, shift uppercase to lowercase letters and so on.  Once my file was converted into an acceptable format, there were **NO** other changes I had to make in order to get KubeFed working on EKS.

### Original Kubectl file, generated by EKS (for style comparison)
``` bash
apiVersion: v1
clusters:
- cluster:
certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSU <...> LS0tLQo=
    server: https://CB5FB82AD8B3FA05F1305283A2D84ECE.sk1.us-west-2.eks.amazonaws.com
  name: basic.us-west-2.eksctl.io
contexts:
- context:
    cluster: basic.us-west-2.eksctl.io
    user: nick-aws@basic.us-west-2.eksctl.io
  name: nick-aws@basic.us-west-2.eksctl.io
current-context: nick-aws@basic.us-west-2.eksctl.io
```

### Format needed by _kubefedctl_ tool
``` bash
apiVersion: v1
clusters:
- cluster:
certificate-authority-data: LS0tLS1CRUdJTiBDRVNBVE <...> LS0tLQo=
    server: https://aa9c46c6e79551ffb7ced7260168ffb4.sk1.us-west-2.eks.amazonaws.com
  name: cluster0
contexts:
- context:
    cluster: cluster0
    user: nick0
  name: nick0-cluster0
- context:
    cluster: cluster1
    user: nick1
  name: nick1-cluster1
- context:
    cluster: cluster2
    user: nick2
  name: nick2-cluster2
current-context: nick0-cluster0
```

### Join clusters

``` bash
$ kubefedctl join cluster0 --cluster-context nick0-cluster0 \
  --host-cluster-context nick0-cluster0 --v=2
$ kubefedctl join cluster1 --cluster-context nick1-cluster1 \
  --host-cluster-context nick0-cluster0 --v=2
$ kubefedctl join cluster2 --cluster-context nick2-cluster2 \
  --host-cluster-context nick0-cluster0 --v=2

$ kubectl -n kube-federation-system get kubefedclusters
NAME       READY   AGE
cluster0   True    10d
cluster1   True    10d
cluster2   True    10d
```

Now, let us examine all the Custom Resource Defintions running on the host cluster:

``` bash
$ kubectl get crd | grep kubefed.io | awk '{ print $1 }'
clusterpropagatedversions.core.kubefed.io
dnsendpoints.multiclusterdns.kubefed.io
domains.multiclusterdns.kubefed.io
federatedclusterroles.types.kubefed.io
federatedconfigmaps.types.kubefed.io
federateddeployments.types.kubefed.io
federatedingresses.types.kubefed.io
federatedjobs.types.kubefed.io
federatednamespaces.types.kubefed.io
federatedreplicasets.types.kubefed.io
federatedsecrets.types.kubefed.io
federatedserviceaccounts.types.kubefed.io
federatedservices.types.kubefed.io
federatedservicestatuses.core.kubefed.io
federatedtypeconfigs.core.kubefed.io
ingressdnsrecords.multiclusterdns.kubefed.io
kubefedclusters.core.kubefed.io
kubefedconfigs.core.kubefed.io
propagatedversions.core.kubefed.io
replicaschedulingpreferences.scheduling.kubefed.io
servicednsrecords.multiclusterdns.kubefed.io
```

### Create a local namespace on the host cluster
``` bash
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: test
EOF

namespace/test created
```

### Create a federated namespace
The local _ns_, and the federated _ns_ should match in name

``` bash
$ cat <<EOF | kubectl apply -f -
apiVersion: types.kubefed.io/v1beta1
kind: FederatedNamespace
metadata:
  name: test
  namespace: test
spec:
  placement:
    clusters:
    - name: cluster1
    - name: cluster2
EOF

federatednamespace.types.kubefed.io/test created
```

### Create a federated deployment
``` bash
$ cat <<EOF | kubectl apply -f - 
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test
  namespace: test
spec:
  template:
    metadata:
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
          - image: nginx
            name: nginx
  placement:
    clusters:
    - name: cluster2
    - name: cluster1
  overrides:
  - clusterName: cluster2
    clusterOverrides:
    - path: "/spec/replicas"
      value: 5
    - path: "/spec/template/spec/containers/0/image"
      value: "nginx:1.17.0-alpine"
    - path: "/metadata/annotations"
      op: "add"
      value:
        foo: bar
EOF

federateddeployment.types.kubefed.io/test created
```

## Verify federation worked
Notice that the number of replicas is different in each cluster. This is because an override was part of the template. The template also had overrides for the container image and added an annotation as well.  This ability to finely control what objects get placed where in your federation is powerful.

``` bash
$ kubectl --context nick1-cluster1 -n test get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   3/3     3            3           5m

$ kubectl --context nick2-cluster2 -n test get deployments
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   5/5     5            5           5m11s
```

### Cleaning up
The safest way to clean up all resources is to simply delete the namespace on the host cluster. This will trigger the deletion process on the member clusters of the namespace, and all objects living within it. This process takes longer than creation, and this is where I have run into issues with KubeFed, and I recognize that it is still an _alpha_ feature set.  I have had the deletion process hang, especially when I tried to delete a large number of objects.

``` bash
$ kubectl delete ns/test
namespace "test" deleted

$ kubectl --context nick2-cluster2 -n test get deployments
No resources found.
```
## DNS
I will not discuss DNS in this post, but be aware that KubeFed supports both ingress and service objects which can dynamically be added into your DNS records. For example, in a High Availability/DR scenario where you have the same application running on several clusters scattered around the world.  Each cluster would add a DNS entry into the top-level CNAME/alias for the app.  If you use Route53, you could also leverage latency-based routing.  If a cluster fails, or a new one is created with the same type of federated resources, the appropriate ingress/service records will be dynamically updated into the master DNS name.

This feature relies heavily on [External DNS](https://github.com/kubernetes-incubator/external-dns), an excellent kubernetes project, which integrates K8 DNS into a global provider.

---

### Other options
The idea of federation has been growing for some time within the Kubernetes community. It used to be that an enterprise would have a single large K8 cluster, or perhaps a small number of them at best.  Now, it is common for enterprises to have dozens, if not hundreds of clusters. Federation allows for greater control at the very least, if not improved independence and flexibility of remote resources.  Since KubeFed is still _alpha_, there are numerous open-source and commercial offerings that allow for the same flexibility and level of control.  Here are a few of the more well-known alternatives.

1. [SAP Gardener](https://kubernetes.io/blog/2018/05/17/gardener/)
2. [Rancher Submariner](https://rancher.com/blog/2019/announcing-submariner-multi-cluster-kubernetes-networking/)
3. [Banzai Pipeline](https://github.com/banzaicloud/pipeline)
4. [Google Anthos](https://cloud.google.com/anthos/)

Finally, if you do not need improved control, but simply want the ability to link your clusters together, one might want to consider a _service mesh_ (i.e. Istio or AWS App Mesh).  A _mesh_ will allow you to leverage pods running on remote clusters, like they were local.  The configuration overhead is currently high, but some of the products just mentioned can help with the setup.

![service mesh](/images/istio-mesh.png)

## Conclusion
The time for Federation seems to be upon us.  There are now numerous open-source and commercial offerings which allow for multi-cluster/multi-cloud architectures to easily be created.  Whether you need such features is still an open question, but the ability to do so robustly and in a scalable manner is rapidly becoming possible.

I will follow up this post with more details and information regarding DNS and other aspects of Federation.  Please contact me at _nbrand@mac.com_ if you have suggestions for additional topics to be addressed in this blog.

---

### References

1. [Banzai blog](https://banzaicloud.com/blog/multi-cloud-fedv2/">Federation v2 article on BanaziCloud)
2. [Evolution of Federation](https://kubernetes.io/blog/2018/12/12/kubernetes-federation-evolution/)
3. [Demo guide](https://github.com/kairen/aws-k8s-federation)
4. [KubeFed github](https://github.com/kubernetes-sigs/kubefed)
5. [KubeFed Helm Charts](https://github.com/kubernetes-sigs/kubefed/blob/master/charts/kubefed/README.md)
6. [KubeFed Cluster Registration](https://github.com/kubernetes-sigs/kubefed/blob/master/docs/cluster-registration.md)

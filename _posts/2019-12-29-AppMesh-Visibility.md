---
layout: post
title:  "App Mesh visibility"
date:   2019-12-29 08:00:00 -0400
categories: aws service_mesh
---

# **DRAFT** ~~Increased visibilty via App Mesh~~ **DRAFT**
![envoy stats](/images/envoy-stats.png)
<!--
    this is a an html comment. It works for Jekyl, but not for other tools, such as MacDown or Pandoc.
	See: https://www.bytedude.com/jekyll-syntax-highlighting-and-line-numbers/
-->
There are many advantages of using Service Meshes. One of the greatest is the increased visibility they can provide. AWS App Mesh leverages Envoy for its data plane. Each envoy proxy generates local [statistics](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/statistics#arch-overview-statistics) describing the network environment it is embedded into. Envoy originally only supported the TCP and UDP [statsD](https://github.com/b/statsd_spec) protocol for exporting its statistics; currently, Envoy also supports Prometheus endpoints as well. statsD is an incredibly simple but very widely supported transport format. 

One of the advantage of statsD, is that this format is easily consumed by a [CloudWatch agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html). This agent can then forward these stats to CloudWatch, allowing for dashboards, alarms and so on. This approach is demonstrated by the [2019 ReInvent talk - CON328, Improving observability of your containers](https://www.youtube.com/embed/O1NQrIm_4cg).

This blog post will demonstrate how to enable statsD for your App Mesh data plane.  This data will consumed by a CloudWatch agent, which will aggregate them into metrics and forward them (once per minute by default) into CloudWatch for further processing. App Mesh can be a critical component for any telemetry or observability based monitoring system, which ultimately allows for increased application visibility.

![pillars of telemetry](/images/pillars-of-telemetry1.png)

In order to quickly show this approach, I have created a Cloud Development Kit or [CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) template of a working containerized, micro-serviced based application running on ECS, along with a CloudWatch dashboard. I will assume you have CDK installed, and some familiarity with the tooling.  Under the hood, CDK generates CloudFormation, but is much more powerful and terse. It allow one to use a general-purpose programming language to create your CloudFormation templates (i.e. JavaScript, TypeScript, Python, Java, and C#)

## Let CDK do all the work
Clone the demo git repository. This repo has been tested in `us-west-2`, but should work in any region with only minor changes.

``` bash
$ git clone https://github.com/nbrandaleone/appmesh-visibility-cdk.git
$ cd appmesh-visibility-cdk
$ npm install

$ npx cdk@1.19.0 deploy --require-approval never

$ # wait about 10 minutes...
```

### The architecture
The application is based upon three micro-services (two tasks each).  The *greeter* must get a *greeting* and a *name*.
* [nathanpeck/greeter](https://hub.docker.com/r/nathanpeck/greeter/) - Constructs a random greeting phrase from a greeting and a name.
* [nathanpeck/greeting](https://hub.docker.com/r/nathanpeck/greeting/) - Returns a random greeting
* [nathanpeck/name](https://hub.docker.com/r/nathanpeck/name/) - Returns a random name

The microservices are connected like this:

![application images](/images/greeter-architecture.png)

Each containerized application has two additional side-cars. One, for the envoy proxy, and the second side-car is the CloudWatch agent.
A rough schematic of the Task archiecture is below.

![3 sidecars](/images/3-containers.png)

From the ECS Console, one can see all three containers working together.

![application images](/images/greeter-sidecar.png)

The CDK template also creates an Application Load Balancer, which exposes the *greeter* micro-services to the world. If you look at the CDK output, you will see the name of the load-balancer, and quickly test out the application from the command line.

![CDK output from the command line](/images/cdk-output.png)

#### Testing
``` bash
$ curl http://Appme-exter-Q7XDLJTVUQED-270012836.us-west-2.elb.amazonaws.com
From ip-10-0-201-189.us-west-2.compute.internal: Greetings (ip-10-0-213-9.us-west-2.compute.internal) Art (ip-10-0-143-116.us-west-2.compute.internal)

$ curl http://Appme-exter-Q7XDLJTVUQED-270012836.us-west-2.elb.amazonaws.com
From ip-10-0-249-232.us-west-2.compute.internal: Hi (ip-10-0-248-152.us-west-2.compute.internal) Courtney (ip-10-0-175-239.us-west-2.compute.internal)
```

Let's generate a little traffic to make our statistics more interesting.
``` bash
$ while true; do curl -s http://Appme-exter-Q7XDLJTVUQED-270012836.us-west-2.elb.amazonaws.com; sleep 0.5; done
```

After a few minutes, go to your CloudWatch / Metrics console.  You will see a new section called *CWAgent*. Once you click on it, you will see over **600** new metrics available to CloudWatch.  Thank you Envoy!!!

![envoy stats in CW](/images/600-metrics.png)

### Configuration and Settings
There are several configuration settings that must be accomplished in order to get the telemetry data flowing into CloudWatch. They are:

1. Configure your [Task definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#proxyConfiguration) to use a special setting called *proxyConfiguration*. This *proxyConfiguration* sets up the network routing between the containers, so that all traffic is routed throught the envoy proxy on its way way in and out of the Task. For CDK, the [code](https://docs.aws.amazon.com/cdk/api/latest/docs/@aws-cdk_aws-ecs.AppMeshProxyConfiguration.html) for the *proxyConfiguration* looks like this:
{% highlight typescript %}
this.taskDefinition = new ecs.Ec2TaskDefinition(this, 'my-task-definition', {
    taskRole: taskIAMRole,
    networkMode: ecs.NetworkMode.AWS_VPC,
    proxyConfiguration: new ecs.AppMeshProxyConfiguration({
        containerName: 'envoy',
        properties: {
            appPorts: [this.portNumber],
            proxyEgressPort: 15001,
            proxyIngressPort: 15000,
            ignoredUID: 1337,
            egressIgnoredIPs: [
            '169.254.170.2',
            '169.254.169.254'
		    ]
        }
    }) 
})  
{% endhighlight %}
2. Add two sidecar containers into your Task, along with your application container. The first sidecar is Envoy. The second sidecar will be the `CloudWatch agent`, which does not need anything special, so I will omit the config. App Mesh manages Envoy configuration to provide service mesh capabilities. App Mesh exports metrics, logs, and traces to the endpoints specified in the Envoy bootstrap configuration provided. Notice the `environmental` variables which enable `statsD` export. We also enable thhe [*datadog*](https://www.datadoghq.com/) version of `statsD` because it allows for richer tagging of metrics which are supported by CloudWatch, even if we are obviously not using *datadog*.
``` typescript
this.envoyContainer = this.taskDefinition.addContainer('envoy', {
    image: ecs.ContainerImage.fromEcrRepository(appMeshRepository, 'v1.12.1.1-prod'),
    essential: true,
    environment: {
        APPMESH_VIRTUAL_NODE_NAME: 'mesh/meshName/virtualNode/myServiceName',
        AWS_REGION: cdk.Stack.of(this).region,
	    ENABLE_ENVOY_STATS_TAGS: '1',
	    ENABLE_ENVOY_DOG_STATSD: '1',
	    ENVOY_LOG_LEVEL: 'debug'
    },
    healthCheck: {
        command: [
            'CMD-SHELL',
            'curl -s http://localhost:9901/server_info | grep state | grep -q LIVE'
        ],
        startPeriod: cdk.Duration.seconds(10),
        interval: cdk.Duration.seconds(5),
        timeout: cdk.Duration.seconds(2),
        retries: 3
    },
    memoryLimitMiB: 128,
    user: '1337',
    logging: new ecs.AwsLogDriver({
        streamPrefix: 'myService-envoy'
    })
})
```
3. Configure container dependencies for proper start-up.  We do not want our application container to start until Envoy is functioning properly. See the [docs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#container_definition_dependson) for the Task definition commands or review the CDK statement below.
``` typescript
// Set start-up order of containers
this.applicationContainer.addContainerDependencies(
	{
           container: this.envoyContainer,
	       condition: ecs.ContainerDependencyCondition.HEALTHY,
	},
	{
  	container: this.cwAgentContainer,
  	condition: ecs.ContainerDependencyCondition.START,
	} 
)
```
4. Configure your second sidecar (i.e. the CloudWatch agent) to accept statsD metrics. There are many ways of configuring the agent, but easiest is again using an `environmental` variable which contains the configuration information. While we will hard-code this variable, it is frequently stored in SSM Parameter Store, where the same config can be leveraged by many agents. The agent will grab most of its relevant information from the EC2 or Task meta-data server, making this configuration minimal.
``` typescript
environment: { 
    CW_CONFIG_CONTENT: '{ "logs": { "metrics_collected": {"emf": {} }}, "metrics": { "metrics_collected": { "statsd": {}}}}'
```

#### DNS
App Mesh relies heavily on DNS or [Cloud Map](https://aws.amazon.com/cloud-map/) to resolve virtual node endpoints.
Fortunately, ECS integrates nicely with Cloud Map, and all our services have private DNS names.  Let's verify this:

``` bash
$ aws servicediscovery list-services --output table
--------------------------------------------------------------------------------------------------
|                                          ListServices                                          |
+------------------------------------------------------------------------------------------------+
||                                           Services                                           ||
|+------------+---------------------------------------------------------------------------------+|
||  Arn       |  arn:aws:servicediscovery:us-west-2:<account ID>:service/srv-24raqotjvd34dfxj   ||
||  CreateDate|  1577741692.171                                                                 ||
||  Id        |  srv-24raqotjvd34dfxj                                                           ||
||  Name      |  name                                                                           ||
|+------------+---------------------------------------------------------------------------------+|
|||                                          DnsConfig                                         |||
||+--------------------------------------------------+-----------------------------------------+||
|||  RoutingPolicy                                   |  MULTIVALUE                             |||
||+--------------------------------------------------+-----------------------------------------+||
||||                                        DnsRecords                                        ||||
|||+---------------------------------------------------+--------------------------------------+|||
||||  TTL                                              |  60                                  ||||
||||  Type                                             |  A                                   ||||
|||+---------------------------------------------------+--------------------------------------+|||
|||                                   HealthCheckCustomConfig                                  |||
||+-------------------------------------------------------------------------+------------------+||
|||  FailureThreshold                                                       |  2               |||
||+-------------------------------------------------------------------------+------------------+||
||                                           Services                                           ||
|+------------+---------------------------------------------------------------------------------+|
...
||+-------------------------------------------------------------------------+------------------+||
```

## Using CloudWatch
Now that all the metrics are streaming into CloudWatch, we can view the logs and visualize the data.
I have built a sample dashboard, which graphs various metrics from Envoy.  [Envoy](https://blog.envoyproxy.io/envoy-stats-b65c7f363342) generates a lot of [statistics](https://www.envoyproxy.io/docs/envoy/v1.5.0/configuration/http_conn_man/stats),
so I focused only on the following most important fields. First, a little clarification on some [vocabulary](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/terminology) that Envoy uses.

> Downstream: A downstream host connects to Envoy, sends requests, and receives responses.

> Upstream: An upstream host receives connections and requests from Envoy and returns responses.

> Listener: A listener is a named network location (e.g., port, unix domain socket, etc.) that can be connected to by downstream clients. Envoy exposes one or more listeners that downstream hosts connect to.

> Cluster: A cluster is a group of logically similar upstream hosts that Envoy connects to. Envoy discovers the members of a cluster via service discovery. It optionally determines the health of cluster members via active health checking.

> Counters: - Unsigned integers that only increase and never decrease. E.g., total requests.

> Gauges: Unsigned integers that both increase and decrease. E.g., currently active requests.

> Timers/histograms: Unsigned integers that ultimately will yield summarized percentile values. Envoy does not differentiate between timers, which are typically measured in milliseconds, and raw histograms, which may be any unit. E.g., upstream request time in milliseconds.

| Statistic | Type | Description |
| --------- | ---- | ----------- |
| downstream_cx_total | Counter | Total downstream connections |
| downstream_cx_active | Gauge | Total active connections |
| downstream_cx_http1_active | Gauge | Total active HTTP/1.1 connections |
| downstream_cx_http2_total | Counter | Total HTTP/2 requests |
| downstream_rq_2xx | Counter | Total 2xx responses |
| ... | ... | ... |
| upstream_cx_ | ... | ... |
| upstream_rq_ | ... | ... |

### The Dashboard
If you open up your CloudWatch tab of your AWS Console, you will find a new Dashboard.
The name is dynamic, but should start with "cloudwatchdashboardappmesh...".

This dashboard shows some of more useful metrics that is gathered from Envoy, along with information retrieved from your Application Load Balancer.
As mentioned, Envoy generates a tremendous amount of useful metrics on your application.  This dashboard is only an example, and it is quite likely that you would build out yours differently than mine. Here is what [Matt Klein](https://blog.envoyproxy.io/lyfts-envoy-dashboards-5c91738816b1) from Lyft [uses](https://github.com/mattklein123/lyft_envoy_dashboards/blob/master/envoy_stats.sls).

![CloudWatch Dashboard](/images/cw-dashboard.png)

*NOTE: [**Metric Math**](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html#metric-math-syntax) is not yet [available](https://github.com/aws/aws-cdk/issues/1077) for CDK.  Once it is, these envoy statistics will become even more powerful. Until them, if you wish to use **Metric Math**, one can build out Dashboards using CloudFormation or manually.*

### CloudWatch Logs Insights
CloudWatch [Logs Insight](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) is an amazing tool.  It allows you to parse through a mountain of logs using a powerful SQL-like syntax. What is also impressive, is that it will create graphs of the search terms you are looking for. Also, since *container insights* embeds performance data into logs, you have yet another way to analyze your containers health. See this helpful [blog](https://aws.amazon.com/blogs/mt/introducing-container-insights-for-amazon-ecs/) post to get you going.

We can use Logs Insights to parse through our Envoy logs as well. Every envoy proxy can log data into Cloudwatch Logs. To export only the Envoy access logs (and ignore the other Envoy container logs), you can set the ENVOY_LOG_LEVEL to [off](https://docs.aws.amazon.com/app-mesh/latest/userguide/envoy.html).

For example, I did a quick review of all my envoy logs to see if there were any HTTP 503 errors.  I only found a handful, but I could imagine how useful this could be in investigating networking issues.

![cloudwatch logs insights](/images/./cw-loginsights-503errors.png)

### Clean up your ECS cluster
Don't forget to tear down your infrastructure to save money.
``` bash
$ npx cdk@1.19.0 destroy
```

## Kubernetes and EKS
We are going to deploy a different micro-service based application on an EKS cluster.
In terms of gathering statistics from our App Mesh based envoy proxies, we have more options using Kubernetes.

There are [several](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-StatsD.html) different methods of forwarding the envoy statistics to CloudWatch.
1. We can duplicate the side-car pattern that we demonstrated for ECS (using statsD).
2. We can implement a daemon set, or a single cloudwatch-agent collector for the Kubernetes/EKS clusters (using statsD).
3. We can use Prometheus, which scapes metrics from the envoy proxies.  We can then configure Prometheus to [export](https://github.com/prometheus/cloudwatch_exporter) the gathered stats to CloudWatch.

In the spirit of replicating the ECS set-up as close as possible, we are going to use option #1. Also, we will use [Helm](https://helm.sh/) almost exclusivey for our Kubernetes package installation and management.

*Add diagram*

### Create a cluster
Let's create an EKS cluster to work with. I am using `us-east-2` this time, to spread out my resources. 

*Would it be easier to stick with us-west-2?*
``` bash
$ # You can skip this part if you already cloned the repository.
$ git clone https://github.com/nbrandaleone/appmesh-visibility-cdk.git
$ cd kubernetes
$ eksctl create cluster -f cluster.yaml
```
I use a `YAML` file so that I can better control the specific configuration of the cluster.  In this case, I want to ensure that the appmesh IAM role is added to the worker nodes.
``` yaml
...
nodeGroups:
  - name: ng-1
    instanceType: t3.xlarge
    desiredCapacity: 2
    volumeSize: 120
    volumeType: gp2
    volumeEncrypted: false
    iam:
      withAddonPolicies:
        appMesh: true
        autoScaler: true
        externalDNS: true
        xRay: true
        cloudwatch: true
        imageBuilder: false
        albIngress: true
...
```
### Install Helm charts
We will use helm 3.x, since this simplifies the configuration as there is no longer a *tiller* component to install on the cluster. We will also add the EKS repo, so we can easily install App Mesh and Prometheus using the appropriate charts.

Of course, if you want to add these component manually, see [here](https://docs.aws.amazon.com/eks/latest/userguide/mesh-k8s-integration.html) and [here](https://docs.aws.amazon.com/eks/latest/userguide/prometheus.html).

``` bash
$ helm repo add eks https://aws.github.io/eks-charts
```

### Install App Mesh
The next series of commands are documented in the Github [repo](https://github.com/aws/eks-charts) for the EKS helm charts.  There are additional commands and steps available that one might want to use. For example, there are examples on how to enable Jaeger tracing, Datadog tracing and AWS X-Ray. Check it out!

Create the appmesh-system namespace:
``` bash
$ kubectl create ns appmesh-system
```

### Install the App Mesh CRD's
```
$ kubectl apply -f https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/crds/crds.yaml
```

### Install the App Mesh CRD controller
``` bash
$ helm upgrade -i appmesh-controller eks/appmesh-controller \
--namespace appmesh-system
```

###  Install the App Mesh admission controller
We are **NOT** going to use the admissions controller/injector for this part of the tutorial. Why you ask? Well, the injector will add the appropriate
*App Mesh* envoy container to your application container automatically.  While this is a great help to the average developer working on Kubernetes, it will create issues when we go to add yet another sidecar. In an effort to better control all these sidecars, their dependencies, start-up order and so on, I will create the full pod/deployment templates without any auto-injector magic.

We will still install the controller, since it does have a knob which will create the App Mesh for us.  Also, you may wish to experiment with it on another namespace.
```
$ helm upgrade -i appmesh-inject eks/appmesh-inject \
--namespace appmesh-system \
--set mesh.create=true \
--set mesh.name=global
```

### Install Prometheus and Grafana
We are going to install both Prometheus and Grafana, which are the most common open-source monitoring tools for Kubernetes clusters. We will only breifly touch on these tools, since the focus of this post in on CloudWatch.  However, it is still important to understand how to install and use them. The AWS provided helm charts has the added benefit of creating a default Grafana dashboard, which displays various App Mesh statistics automatically.

``` bash
$ helm upgrade -i appmesh-prometheus eks/appmesh-prometheus \
--namespace appmesh-system

$ # Install App Mesh Grafana:
$ helm upgrade -i appmesh-prometheus eks/appmesh-grafana \
--namespace appmesh-system 
```

### Install the sample application
Also, review components...

### Review CloudWatch Dashboard
*Should be similar results as ECS*

### Clean up EKS cluster

## Summary
<img src="/images/appmesh-logo.svg" align="right" width="250" height="250">
Just to recap, we have seen how to leverage App Mesh in order to get increased visibility into
all the networking stats produced by the envoy proxies.  We have forwarded these metrics to CloudWatch, where we can easily build a Dashboard for increased awareness regarding our application. While this does require some work to set-up and configure, it is essentially free data that we can utilize simply by using App Mesh. CDK can also make the set-up considerably easier than in the past. 

Although we have just scratched the service in terms of Envoy statistics, hopefully, you have found this post useful and interesting.

Of course, it is always possible, and in many cases preferrable to use AWS X-Ray for increased visibility.  X-Ray provides a visual representation of traffic flowing between services which can invaluable during troubleshooting. This particular topic has been discussed multiple times. For example: see these blog posts and articles:
- [ReInvent 2018 slides on using X-ray with containers](https://www.slideshare.net/AmazonWebServices/instrumenting-kubernetes-for-observability-using-aws-xray-and-amazon-cloudwatch-dev303r2-aws-reinvent-2018)
- [EKS workshop](https://eksworkshop.com/intermediate/245_x-ray/)
- [App Mesh workshop](https://www.appmeshworkshop.com/x-ray/)

![xray images](/images/xray.png)

Also, a very easy way to get started with in-depth container and cluster statistics for either ECS or EKS is [Container Insights from CloudWatch](https://eksworkshop.com/intermediate/250_cloudwatch_container_insights/).

![Container Insights picture](/images/insights.png)

Finally, you should definitely check out this new CloudWatch service, [**ServiceLens**](Appme-exter-Q7XDLJTVUQED-270012836.us-west-2.elb.amazonaws.com). It brings together numerous monitoring tools (i.e. CloudWatch logs and X-Ray) into a single location, while focusing on micro-services based application.  There is already a great [demo](https://github.com/lavignes/aws-app-mesh-examples/tree/service-lens/walkthroughs/howto-service-lens) on Github.  Check it out!

![CloudWatch ServiceLens](/images/ServiceMap.png)

***

## References
* Using [CloudWatch Agent and StatsD](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-custom-metrics-statsd.html)
* Nathan Peck's [greeter application](https://github.com/nathanpeck/greeter-app-mesh-cdk/blob/master/README.md) written in CDK and leveraging App Mesh
* Tony Pujals ColorTeller [app](https://github.com/subfuzion/enable-appmesh) in Console and CDK
* ColorTeller with a [Vue](https://github.com/enghwa/cdkcolorteller) front-end. Also implemented in CDK.

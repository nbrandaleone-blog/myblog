---
layout: post
title:  "How to configure your EKS cluster"
date:   2020-03-06 08:00:00 -0400
categories: kubernetes
---
# Four ways to configure an EKS cluster

It is becoming more and more common for companies to have numerous kubernetes clusters.  [EKS](https://aws.amazon.com/eks/) is quickly becoming the standard for managed kubernetes clusters.  However, an EKS cluster is created bare bones, with no **add-ons**.  While this is something that other kubernetes distros support, it is not difficult to automate the process of installing common add-ons onto your cluster.

![CNCF poll](/images/CNCF-poll.png)

## What are add-ons?

Before your kubernetes cluster can be productive, it needs customization and additional components.  The most common software packages I install are:

- CloudWatch Container Insights for collecting performance metrics and logs
- ALB Ingress Controller
- Calico nework policy engine
- Prometheus to scrape pod and kubernetes metrics
- Grafana. To display the aforementioned metrics
- Metric Server. Required for Horizontal Autoscaler to work
- Cluster Autoscaler
- etc...

Now, the question is: **How** do we quickly and easily install these, and other packages, into every new EKS cluster in an automated manner?  Well, I propose four options. Each choice works well, but comes with its own pluses and minuses. You will have to judge what is best for you.

## Option 1 - BASH scripts

Let's not fool ourselves, we can go a long way using BASH scripts. Fortunately, there is a [project](https://github.com/mreferre/ekstender) called **ekstender** which uses Docker to install some tooling, and a BASH script to install or uninstall various packages onto your cluster.  You can customize the install process by toggling environmental variables, or of course by modifying the installation script.

### ekstender example

Customize the install process using environmental variables.

``` shell
export MINNODES=2
export MAXNODES=6
export EXTERNALDASHBOARD=yes
export EXTERNALPROMETHEUS=yes
export DEMOAPP=yes
export KUBEFLOW=no
export NAMESPACE="kube-system"
export MESH_NAME="ekstender-mesh"
```
This is an example of a typical setup

``` shell
sh-4.2# ./ekstender.sh eks2
 2019-09-13 13:39:16 -- *************************************************
 2019-09-13 13:39:16 -- ***  Do not run this on a production cluster  ***
 2019-09-13 13:39:16 -- *** This is solely for demo/learning purposes ***
 2019-09-13 13:39:16 -- *************************************************
 2019-09-13 13:39:16 -- You are about to EKStend your EKS cluster
 2019-09-13 13:39:16 -- These are the environment settings that are going to be used:
 2019-09-13 13:39:16 -- Cluster Name          : eks1
 2019-09-13 13:39:16 -- AWS Region            : us-west-2
 2019-09-13 13:39:16 -- Node Instance Role    : eksctl-eks2-nodegroup-ng-9a16907b-NodeInstanceRole-1AS92OJAG6GPG
 2019-09-13 13:39:16 -- Kubernetes Namespace  : kube-system
 2019-09-13 13:39:16 -- ASG Name              : eksctl-eks2-nodegroup-ng-9a16907b-NodeGroup-173MMIPFJ79XK
 2019-09-13 13:39:16 -- Min Number of Nodes   : 2
 2019-09-13 13:39:16 -- Max Number of Nodes   : 6
 2019-09-13 13:39:16 -- External Dashboard    : yes
 2019-09-13 13:39:16 -- External Prometheus   : yes
 2019-09-13 13:39:16 -- Mesh Name             : ekstender-mesh
 2019-09-13 13:39:16 -- Demo application      : yes
 2019-09-13 13:39:16 -- Kubeflow              : yes
 2019-09-13 13:39:16 -- Press [Enter] to continue or CTRL-C to abort...

 2019-09-13 13:39:20 -- Namespace exists. Skipping...
 2019-09-13 13:39:20 -- Creation of the generic eks-admin service account is starting...
 2019-09-13 13:39:21 -- Creation of the generic eks-admin service account has completed...
 2019-09-13 13:39:21 -- Calico setup is starting...
 2019-09-13 13:39:21 -- Waiting for Calico pods to come up...
 2019-09-13 13:39:31 -- Calico has been installed properly!
 2019-09-13 13:39:31 -- Helm setup is starting...
 2019-09-13 13:39:33 -- Waiting for the tiller pod to come up...
 2019-09-13 13:39:48 -- Tiller has been installed properly!
 2019-09-13 13:39:48 -- Metric server deployment is starting...
 2019-09-13 13:39:50 -- Metric server has been installed properly!
 2019-09-13 13:39:50 -- Dashboard setup is starting...
 2019-09-13 13:39:52 -- Warning: I am exposing the Kubernetes dashboard to the Internet...
 2019-09-13 13:39:52 -- Dashboard has been installed properly!
 2019-09-13 13:39:52 -- ALB Ingress controller setup is starting...
 2019-09-13 13:40:09 -- ALB Ingress controller has been installed properly!
 2019-09-13 13:40:09 -- Prometheus setup is starting...
 2019-09-13 13:40:11 -- Prometheus is being exposed to the Internet......
 2019-09-13 13:40:11 -- Prometheus has been installed properly!
 2019-09-13 13:40:11 -- Grafana setup is starting...
 2019-09-13 13:40:14 -- Grafana has been installed properly!
 2019-09-13 13:40:14 -- CloudWatch Containers Insights setup is starting...
 2019-09-13 13:40:18 -- CloudWatch Containers Insights has been installed properly!
 2019-09-13 13:40:18 -- Cluster Autoscaler deployment is starting...
 2019-09-13 13:40:19 -- Cluster Autoscaler has been installed properly!
 2019-09-13 13:40:19 -- Appmesh components setup is starting...
 2019-09-13 13:40:34 -- Appmesh components have been installed properly
 2019-09-13 13:40:34 -- Demo application setup is starting...
 2019-09-13 13:42:35 -- Demo application has been installed properly!
 2019-09-13 13:42:35 -- Kubeflow setup is starting...
 2019-09-13 13:45:22 -- Kubeflow has been installed properly!
 2019-09-13 13:45:22 -- Almost there...
 2019-09-13 13:46:03 -- Congratulations! You made it!
 2019-09-13 13:46:03 -- Your EKStended kubernetes environment is ready to be used
 2019-09-13 13:46:03 -- ------
 2019-09-13 13:46:03 -- Grafana UI           : http://a03b6ee5fd62c11e9bdfd067eea79c3b-534376637.us-west-2.elb.amazonaws.com
 2019-09-13 13:46:03 -- Prometheus UI        : http://a022e2c58d62c11e9bdfd067eea79c3b-1184130494.us-west-2.elb.amazonaws.com
 2019-09-13 13:46:03 -- Kubernetes Dashboard : https://af7088762d62b11e9a77b0ea9c3f9cb7-1239877256.us-west-2.elb.amazonaws.com:8443
 2019-09-13 13:46:03 -- Demo application     : http://a5fb8759-kubesystem-yelbui-bbd8-1694418741.us-west-2.elb.amazonaws.com
 2019-09-13 13:46:03 -- Kubeflow             : http://null
 2019-09-13 13:46:03 -- ------
 2019-09-13 13:46:03 -- Note that it may take several minutes for these end-points to be fully operational
 2019-09-13 13:46:03 -- If you see a <null> or no value you specifically opted out for that particular feature or the LB isn't ready yet (check with kubectl)
 2019-09-13 13:46:03 -- Enjoy!
sh-4.2#
```

## Option 2 - Helmfile

Helm is the de-facto package manager for kubernetes. With the recent release of version 3, this tool has become even easier to use. It works by combining several manifests into a single package that is called a chart. Helm also has amazing templating features, based upon Go templates.

However, helm typically manages the lifecycle of a single software package.  Is there a way to have it handle multiple packages at once?

Fortunately, there is a way using a project simply called [helmfile](https://github.com/roboll/helmfile). Let's create a simple **helmfile.yaml**, with a few packages as a demonstration.

Obviously, the one downside of this solution is that your software packages must be availale as helm chart.  This is most likely the case, but may cause issues if you are using something a little more unique.

### Helmfile example

```shell
$ brew install helmfile
...

$ cat helmfile.yaml
repositories:
- name: incubator
  url: "http://storage.googleapis.com/kubernetes-charts-incubator"

releases:
- name: prom-norbac-ubuntu
  namespace: prometheus
  chart: stable/prometheus
  set:
  - name: rbac.create
    value: true
  - name: alertmanager.persistentVolume.storageClass
    value: gp2
  - name: server.persistentVolume.storageClass
    value: gp2
  hooks:
  - events: ["presync"]
    showlogs: true
    command: "/bin/sh"
    args:
    - "-c"
    - >-
      kubectl get namespace "{{`{{ .Release.Namespace }}`}}" >/dev/null 2>&1 || kubectl create namespace "{{`{{ .Release.Namespace }}`}}";
- name: alb-ingress-controller
  namespace: kube-system
  chart: incubator/aws-alb-ingress-controller
  set:
  - name: clusterName
    value: MyCluster
  - name: autoDiscoverAwsRegion
    value: true
  - name: autoDiscoverAwsVpcID
    value: true
```

Now, let's execute the helmfile, which will apply all the helm packages to the cluster.  In this case, there are only 2 releases, but it could be as many as you like.

``` shell
$ helmfile sync
...
$ helmfile list
NAME                      	NAMESPACE  	INSTALLED	LABELS
prom-norbac-ubuntu        	prometheus 	true
aws-alb-ingress-controller	kube-system	true
```

## Option 3 - CDK and Cloudformation

Cloud Delivery Kit or [CDK](https://aws.amazon.com/cdk/) is an interesting project from AWS, which takes much of the pain out of CloudFormation. It allows you to to use the language of your choice (JavaScript, TypeScript, Python, Java, and C#), which then generates and deploys/destroys/updates the resulting CF templates.

There is a new CDK library focused on Kuberetes. It not only allows you to create EKS clusters, but also deploy helm charts or other native Kubernetes objects like **pods** or **deployments**, **services**, etc...

If you are using CDK/CloudFormation as your *infrastructure as code* tool, this new library may be very useful for you.  Then again, if you are not, there are other tools that may be easier and more performant to use.

One last interesting item.  This CDK library is now being spun off as its own project called [**cdk8s**](https://github.com/awslabs/cdk8s).  Now, you will be able to generate kubernetes templates, and not be reliant upon CloudFormation or EKS. So, you can use a high level programming language for template generation.  Not a bad option!

> cdk8s is a software development framework for defining Kubernetes applications and reusable abstractions using familiar programming languages and rich object-oriented APIs. cdk8s generates pure Kubernetes YAML - you can use cdk8s to define applications for any Kubernetes cluster running anywhere.

### CDK example
Here is a [snippet](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-eks-readme.html) of the CDK typescript file, describing a kubernetes deployment and service. I will deploy the CDK application, and it will create a new EKS cluster, and install the kubernetes deployment and service at the same time.

```
const appLabel = { app: "hello-kubernetes" };

const deployment = {
  apiVersion: "apps/v1",
  kind: "Deployment",
  metadata: { name: "hello-kubernetes" },
  spec: {
    replicas: 3,
    selector: { matchLabels: appLabel },
    template: {
      metadata: { labels: appLabel },
      spec: {
        containers: [
          {
            name: "hello-kubernetes",
            image: "paulbouwer/hello-kubernetes:1.5",
            ports: [ { containerPort: 8080 } ]
          }
        ]
      }
    }
  }
};

const service = {
  apiVersion: "v1",
  kind: "Service",
  metadata: { name: "hello-kubernetes" },
  spec: {
    type: "LoadBalancer",
    ports: [ { port: 80, targetPort: 8080 } ],
    selector: appLabel
  }
};

// option 1: use a construct
new KubernetesResource(this, 'hello-kub', {
  cluster,
  manifest: [ deployment, service ]
});

// or, option2: use `addResource`
cluster.addResource('hello-kub', service, deployment);
```

Now, lets deploy the CDK application.

```shell
$ npx cdk version
1.27.0 (build a98c0b3)
✔ ~/src/cdk/cdk-eks [master L|✚ 4…1]
12:29 $ npx cdk deploy
This deployment will make potentially sensitive changes according to your current security approval level (--require-approval broadening).
Please confirm you intend to make the following modifications:

IAM Statement Changes
┌───┬─────────────────┬────────┬─────────────────┬───────────────────┬───────────┐
│   │ Resource        │ Effect │ Action          │ Principal         │ Condition │
├───┼─────────────────┼────────┼─────────────────┼───────────────────┼───────────┤
│ + │ ${AdminRole.Arn │ Allow  │ sts:AssumeRole  │ AWS:arn:${AWS::Pa │           │
│   │ }               │        │                 │ rtition}:iam::${A │           │
│   │                 │        │                 │ WS::AccountId}:ro │           │
│   │                 │        │                 │ ot                │           │
├───┼─────────────────┼────────┼─────────────────┼───────────────────┼───────────┤
│ + │ ${hello-eks/Res │ Allow  │ sts:AssumeRole  │ AWS:${@aws-cdk--a │           │
│   │ ource/CreationR │        │                 │ ws-eks.ClusterRes │           │
│   │ ole.Arn}        │        │                 │ ourceProvider.Nes │           │
│   │                 │        │                 │ tedStack/@aws-cdk │           │
│   │                 │        │                 │ --aws-eks.Cluster │           │
│   │                 │        │                 │ ResourceProvider. │           │
│   │                 │        │                 │ NestedStackResour │           │
│   │                 │        │                 │ ce.Outputs.CdkEks │           │
│   │                 │        │                 │ Stackawscdkawseks │           │
│   │                 │        │                 │ ClusterResourcePr │           │
│   │                 │        │                 │ oviderIsCompleteH │           │
│   │                 │        │                 │ andlerServiceRole │           │
│   │                 │        │                 │ 5C844CA3Arn}      │           │
│   │                 │        │                 │ AWS:${@aws-cdk--a │           │
│   │                 │        │                 │ ws-eks.ClusterRes │           │
│   │                 │        │                 │ ourceProvider.Nes │           │
│   │                 │        │                 │ tedStack/@aws-cdk │           │
│   │                 │        │                 │ --aws-eks.Cluster │           │
│   │                 │        │                 │ ResourceProvider. │           │
│   │                 │        │                 │ NestedStackResour │           │
│   │                 │        │                 │ ce.Outputs.CdkEks │           │
│   │                 │        │                 │ Stackawscdkawseks │           │
│   │                 │        │                 │ ClusterResourcePr │           │
│   │                 │        │                 │ oviderOnEventHand │           │
│   │                 │        │                 │ lerServiceRoleC66 │           │
│   │                 │        │                 │ 3A22DArn}         │           │
│ + │ ${hello-eks/Res │ Allow  │ sts:AssumeRole  │ AWS:${@aws-cdk--a │           │
│   │ ource/CreationR │        │                 │ ws-eks.KubectlPro │           │
│   │ ole.Arn}        │        │                 │ vider.NestedStack │           │
│   │                 │        │                 │ /@aws-cdk--aws-ek │           │
│   │                 │        │                 │ s.KubectlProvider │           │
│   │                 │        │                 │ .NestedStackResou │           │
│   │                 │        │                 │ rce.Outputs.CdkEk │           │
│   │                 │        │                 │ sStackawscdkawsek │           │
│   │                 │        │                 │ sKubectlProviderH │           │
│   │                 │        │                 │ andlerServiceRole │           │
│   │                 │        │                 │ D02136FCArn}      │           │
├───┼─────────────────┼────────┼─────────────────┼───────────────────┼───────────┤
│ + │ ${hello-eks/Def │ Allow  │ sts:AssumeRole  │ Service:ec2.amazo │           │
│   │ aultCapacity/In │        │                 │ naws.com          │           │
│   │ stanceRole.Arn} │        │                 │                   │           │
├───┼─────────────────┼────────┼─────────────────┼───────────────────┼───────────┤
│ + │ ${hello-eks/Rol │ Allow  │ sts:AssumeRole  │ Service:eks.amazo │           │
│   │ e.Arn}          │        │                 │ naws.com          │           │
│ + │ ${hello-eks/Rol │ Allow  │ iam:PassRole    │ AWS:${hello-eks/R │           │
│   │ e.Arn}          │        │                 │ esource/CreationR │           │
│   │                 │        │                 │ ole}              │           │
├───┼─────────────────┼────────┼─────────────────┼───────────────────┼───────────┤
│ + │ *               │ Allow  │ ec2:DescribeSub │ AWS:${hello-eks/R │           │
│   │                 │        │ nets            │ esource/CreationR │           │
│   │                 │        │                 │ ole}              │           │
│ + │ *               │ Allow  │ eks:CreateClust │ AWS:${hello-eks/R │           │
│   │                 │        │ er              │ esource/CreationR │           │
│   │                 │        │ eks:CreateFarga │ ole}              │           │
│   │                 │        │ teProfile       │                   │           │
│   │                 │        │ eks:DeleteClust │                   │           │
│   │                 │        │ er              │                   │           │
│   │                 │        │ eks:DescribeClu │                   │           │
│   │                 │        │ ster            │                   │           │
│   │                 │        │ eks:UpdateClust │                   │           │
│   │                 │        │ erConfig        │                   │           │
│   │                 │        │ eks:UpdateClust │                   │           │
│   │                 │        │ erVersion       │                   │           │
│ + │ *               │ Allow  │ eks:DeleteFarga │ AWS:${hello-eks/R │           │
│   │                 │        │ teProfile       │ esource/CreationR │           │
│   │                 │        │ eks:DescribeFar │ ole}              │           │
│   │                 │        │ gateProfile     │                   │           │
│ + │ *               │ Allow  │ iam:GetRole     │ AWS:${hello-eks/R │           │
│   │                 │        │                 │ esource/CreationR │           │
│   │                 │        │                 │ ole}              │           │
│ + │ *               │ Allow  │ iam:CreateServi │ AWS:${hello-eks/R │           │
│   │                 │        │ ceLinkedRole    │ esource/CreationR │           │
│   │                 │        │                 │ ole}              │           │
└───┴─────────────────┴────────┴─────────────────┴───────────────────┴───────────┘
IAM Policy Changes
┌───┬─────────────────────────────────────┬──────────────────────────────────────┐
│   │ Resource                            │ Managed Policy ARN                   │
├───┼─────────────────────────────────────┼──────────────────────────────────────┤
│ + │ ${hello-eks/DefaultCapacity/Instanc │ arn:${AWS::Partition}:iam::aws:polic │
│   │ eRole}                              │ y/AmazonEKSWorkerNodePolicy          │
│ + │ ${hello-eks/DefaultCapacity/Instanc │ arn:${AWS::Partition}:iam::aws:polic │
│   │ eRole}                              │ y/AmazonEKS_CNI_Policy               │
│ + │ ${hello-eks/DefaultCapacity/Instanc │ arn:${AWS::Partition}:iam::aws:polic │
│   │ eRole}                              │ y/AmazonEC2ContainerRegistryReadOnly │
├───┼─────────────────────────────────────┼──────────────────────────────────────┤
│ + │ ${hello-eks/Role}                   │ arn:${AWS::Partition}:iam::aws:polic │
│   │                                     │ y/AmazonEKSClusterPolicy             │
│ + │ ${hello-eks/Role}                   │ arn:${AWS::Partition}:iam::aws:polic │
│   │                                     │ y/AmazonEKSServicePolicy             │
└───┴─────────────────────────────────────┴──────────────────────────────────────┘
Security Group Changes
┌───┬──────────────────────┬─────┬──────────────────────┬────────────────────────┐
│   │ Group                │ Dir │ Protocol             │ Peer                   │
├───┼──────────────────────┼─────┼──────────────────────┼────────────────────────┤
│ + │ ${hello-eks/ControlP │ In  │ TCP 443              │ ${hello-eks/DefaultCap │
│   │ laneSecurityGroup.Gr │     │                      │ acity/InstanceSecurity │
│   │ oupId}               │     │                      │ Group.GroupId}         │
│ + │ ${hello-eks/ControlP │ Out │ Everything           │ Everyone (IPv4)        │
│   │ laneSecurityGroup.Gr │     │                      │                        │
│   │ oupId}               │     │                      │                        │
├───┼──────────────────────┼─────┼──────────────────────┼────────────────────────┤
│ + │ ${hello-eks/DefaultC │ In  │ Everything           │ ${hello-eks/DefaultCap │
│   │ apacity/InstanceSecu │     │                      │ acity/InstanceSecurity │
│   │ rityGroup.GroupId}   │     │                      │ Group.GroupId}         │
│ + │ ${hello-eks/DefaultC │ In  │ TCP 443              │ ${hello-eks/ControlPla │
│   │ apacity/InstanceSecu │     │                      │ neSecurityGroup.GroupI │
│   │ rityGroup.GroupId}   │     │                      │ d}                     │
│ + │ ${hello-eks/DefaultC │ In  │ TCP 1025-65535       │ ${hello-eks/ControlPla │
│   │ apacity/InstanceSecu │     │                      │ neSecurityGroup.GroupI │
│   │ rityGroup.GroupId}   │     │                      │ d}                     │
│ + │ ${hello-eks/DefaultC │ Out │ Everything           │ Everyone (IPv4)        │
│   │ apacity/InstanceSecu │     │                      │                        │
│   │ rityGroup.GroupId}   │     │                      │                        │
└───┴──────────────────────┴─────┴──────────────────────┴────────────────────────┘
(NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)

Do you wish to deploy these changes (y/n)? y
CdkEksStack: deploying...
Updated: asset.f8180bff2a23be3f5a1fee126adab8881214d211868a8b0adc8ba767dd9d430c (zip)
Updated: asset.f587c683163dea7b70b883fe8f803ffe0622a40e05b3766e08ffa9ed25caabc9 (zip)
Updated: asset.7885982e43353796b16117389f19d46f133d2b866759b8b87c714aa408d6a25a (zip)
Updated: CdkEksStackawscdkawseksKubectlProviderD47BE170.nested.template.json (file)
Updated: CdkEksStackawscdkawseksClusterResourceProviderD28F6BA5.nested.template.json (file)
CdkEksStack: creating CloudFormation changeset...
  0/45 | 12:30:22 PM | CREATE_IN_PROGRESS   | AWS::IAM::User                        | nick-aws (nickawsE311DE7F)
  0/45 | 12:30:23 PM | CREATE_IN_PROGRESS   | AWS::IAM::User                        | nick-aws (nickawsE311DE7F) Resource creation Initiated
  0/45 | 12:30:23 PM | CREATE_IN_PROGRESS   | AWS::CDK::Metadata                    | CDKMetadata
  0/45 | 12:30:23 PM | CREATE_IN_PROGRESS   | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet1/EIP (helloeksDefaultVpcPublicSubnet1EIPCE084C76)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/Role (helloeksRole59C1FE10)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.ClusterResourceProvider.NestedStack/@aws-cdk--aws-eks.ClusterResourceProvider.NestedStackResource (awscdkawseksClusterResourceProviderNestedStackawscdkawseksClusterResourceProviderNestedStackResource9827C454)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet1/EIP (helloeksDefaultVpcPublicSubnet1EIPCE084C76) Resource creation Initiated
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/Role (helloeksRole59C1FE10) Resource creation Initiated
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::EC2::InternetGateway             | hello-eks/DefaultVpc/IGW (helloeksDefaultVpcIGWC3E1888E)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet2/EIP (helloeksDefaultVpcPublicSubnet2EIPE99C1EF7)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::EC2::VPC                         | hello-eks/DefaultVpc (helloeksDefaultVpcB529665A)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | AdminRole (AdminRole38563C57)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.KubectlProvider.NestedStack/@aws-cdk--aws-eks.KubectlProvider.NestedStackResource (awscdkawseksKubectlProviderNestedStackawscdkawseksKubectlProviderNestedStackResourceA7AEBA6B)
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::CDK::Metadata                    | CDKMetadata Resource creation Initiated
  0/45 | 12:30:24 PM | CREATE_IN_PROGRESS   | AWS::EC2::InternetGateway             | hello-eks/DefaultVpc/IGW (helloeksDefaultVpcIGWC3E1888E) Resource creation Initiated
  0/45 | 12:30:25 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | AdminRole (AdminRole38563C57) Resource creation Initiated
  0/45 | 12:30:25 PM | CREATE_IN_PROGRESS   | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet2/EIP (helloeksDefaultVpcPublicSubnet2EIPE99C1EF7) Resource creation Initiated
  1/45 | 12:30:25 PM | CREATE_COMPLETE      | AWS::CDK::Metadata                    | CDKMetadata
  1/45 | 12:30:25 PM | CREATE_IN_PROGRESS   | AWS::EC2::VPC                         | hello-eks/DefaultVpc (helloeksDefaultVpcB529665A) Resource creation Initiated
  1/45 | 12:30:25 PM | CREATE_IN_PROGRESS   | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.ClusterResourceProvider.NestedStack/@aws-cdk--aws-eks.ClusterResourceProvider.NestedStackResource (awscdkawseksClusterResourceProviderNestedStackawscdkawseksClusterResourceProviderNestedStackResource9827C454) Resource creation Initiated
  1/45 | 12:30:25 PM | CREATE_IN_PROGRESS   | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.KubectlProvider.NestedStack/@aws-cdk--aws-eks.KubectlProvider.NestedStackResource (awscdkawseksKubectlProviderNestedStackawscdkawseksKubectlProviderNestedStackResourceA7AEBA6B) Resource creation Initiated
  2/45 | 12:30:38 PM | CREATE_COMPLETE      | AWS::IAM::Role                        | hello-eks/Role (helloeksRole59C1FE10)
  3/45 | 12:30:38 PM | CREATE_COMPLETE      | AWS::IAM::Role                        | AdminRole (AdminRole38563C57)
  4/45 | 12:30:40 PM | CREATE_COMPLETE      | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet1/EIP (helloeksDefaultVpcPublicSubnet1EIPCE084C76)
  5/45 | 12:30:41 PM | CREATE_COMPLETE      | AWS::EC2::EIP                         | hello-eks/DefaultVpc/PublicSubnet2/EIP (helloeksDefaultVpcPublicSubnet2EIPE99C1EF7)
  6/45 | 12:30:41 PM | CREATE_COMPLETE      | AWS::EC2::InternetGateway             | hello-eks/DefaultVpc/IGW (helloeksDefaultVpcIGWC3E1888E)
  7/45 | 12:30:42 PM | CREATE_COMPLETE      | AWS::EC2::VPC                         | hello-eks/DefaultVpc (helloeksDefaultVpcB529665A)
  7/45 | 12:30:46 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet2/RouteTable (helloeksDefaultVpcPublicSubnet2RouteTableB32C98F7)
  7/45 | 12:30:46 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet2/RouteTable (helloeksDefaultVpcPrivateSubnet2RouteTable4EE1E9EA)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroup               | hello-eks/ControlPlaneSecurityGroup (helloeksControlPlaneSecurityGroup4C2AFE28)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet2/RouteTable (helloeksDefaultVpcPublicSubnet2RouteTableB32C98F7) Resource creation Initiated
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet1/Subnet (helloeksDefaultVpcPrivateSubnet1Subnet640168A4)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet2/RouteTable (helloeksDefaultVpcPrivateSubnet2RouteTable4EE1E9EA) Resource creation Initiated
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::VPCGatewayAttachment        | hello-eks/DefaultVpc/VPCGW (helloeksDefaultVpcVPCGWC6655FB8)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet1/RouteTable (helloeksDefaultVpcPublicSubnet1RouteTable3B0A76CD)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet2/Subnet (helloeksDefaultVpcPrivateSubnet2Subnet61370A07)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet1/Subnet (helloeksDefaultVpcPrivateSubnet1Subnet640168A4) Resource creation Initiated
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet1/Subnet (helloeksDefaultVpcPublicSubnet1Subnet00837EBB)
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::VPCGatewayAttachment        | hello-eks/DefaultVpc/VPCGW (helloeksDefaultVpcVPCGWC6655FB8) Resource creation Initiated
  7/45 | 12:30:47 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet1/RouteTable (helloeksDefaultVpcPublicSubnet1RouteTable3B0A76CD) Resource creation Initiated
  7/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet2/Subnet (helloeksDefaultVpcPrivateSubnet2Subnet61370A07) Resource creation Initiated
  7/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet1/RouteTable (helloeksDefaultVpcPrivateSubnet1RouteTable5FE24FD2)
  8/45 | 12:30:48 PM | CREATE_COMPLETE      | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet2/RouteTable (helloeksDefaultVpcPublicSubnet2RouteTableB32C98F7)
  9/45 | 12:30:48 PM | CREATE_COMPLETE      | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet2/RouteTable (helloeksDefaultVpcPrivateSubnet2RouteTable4EE1E9EA)
  9/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet1/Subnet (helloeksDefaultVpcPublicSubnet1Subnet00837EBB) Resource creation Initiated
  9/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet2/Subnet (helloeksDefaultVpcPublicSubnet2Subnet8FD7C575)
  9/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet1/RouteTable (helloeksDefaultVpcPrivateSubnet1RouteTable5FE24FD2) Resource creation Initiated
  9/45 | 12:30:48 PM | CREATE_IN_PROGRESS   | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet2/Subnet (helloeksDefaultVpcPublicSubnet2Subnet8FD7C575) Resource creation Initiated
 10/45 | 12:30:49 PM | CREATE_COMPLETE      | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PublicSubnet1/RouteTable (helloeksDefaultVpcPublicSubnet1RouteTable3B0A76CD)
 11/45 | 12:30:49 PM | CREATE_COMPLETE      | AWS::EC2::RouteTable                  | hello-eks/DefaultVpc/PrivateSubnet1/RouteTable (helloeksDefaultVpcPrivateSubnet1RouteTable5FE24FD2)
 11/45 | 12:30:51 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroup               | hello-eks/ControlPlaneSecurityGroup (helloeksControlPlaneSecurityGroup4C2AFE28) Resource creation Initiated
 12/45 | 12:30:53 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroup               | hello-eks/ControlPlaneSecurityGroup (helloeksControlPlaneSecurityGroup4C2AFE28)
 13/45 | 12:30:58 PM | CREATE_COMPLETE      | AWS::IAM::User                        | nick-aws (nickawsE311DE7F)
 14/45 | 12:31:03 PM | CREATE_COMPLETE      | AWS::EC2::VPCGatewayAttachment        | hello-eks/DefaultVpc/VPCGW (helloeksDefaultVpcVPCGWC6655FB8)
 15/45 | 12:31:04 PM | CREATE_COMPLETE      | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet1/Subnet (helloeksDefaultVpcPrivateSubnet1Subnet640168A4)
 16/45 | 12:31:04 PM | CREATE_COMPLETE      | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PrivateSubnet2/Subnet (helloeksDefaultVpcPrivateSubnet2Subnet61370A07)
 17/45 | 12:31:04 PM | CREATE_COMPLETE      | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet1/Subnet (helloeksDefaultVpcPublicSubnet1Subnet00837EBB)
 18/45 | 12:31:05 PM | CREATE_COMPLETE      | AWS::EC2::Subnet                      | hello-eks/DefaultVpc/PublicSubnet2/Subnet (helloeksDefaultVpcPublicSubnet2Subnet8FD7C575)
 18/45 | 12:31:08 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet1/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet1RouteTableAssociation63177EF1)
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet2/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet2RouteTableAssociationD7E96AD6)
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet2/DefaultRoute (helloeksDefaultVpcPublicSubnet2DefaultRouteC8106461)
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet1/DefaultRoute (helloeksDefaultVpcPublicSubnet1DefaultRoute01D8F765)
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet2/DefaultRoute (helloeksDefaultVpcPublicSubnet2DefaultRouteC8106461) Resource creation Initiated
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet2/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet2RouteTableAssociationD7E96AD6) Resource creation Initiated
 18/45 | 12:31:09 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet1/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet1RouteTableAssociation63177EF1) Resource creation Initiated
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet1/RouteTableAssociation (helloeksDefaultVpcPublicSubnet1RouteTableAssociation817AEFF1)
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet1/DefaultRoute (helloeksDefaultVpcPublicSubnet1DefaultRoute01D8F765) Resource creation Initiated
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet2/RouteTableAssociation (helloeksDefaultVpcPublicSubnet2RouteTableAssociation1728C296)
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet2/NATGateway (helloeksDefaultVpcPublicSubnet2NATGatewayD1BD3D33)
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet1/NATGateway (helloeksDefaultVpcPublicSubnet1NATGateway67696503)
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet2/NATGateway (helloeksDefaultVpcPublicSubnet2NATGatewayD1BD3D33) Resource creation Initiated
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet1/RouteTableAssociation (helloeksDefaultVpcPublicSubnet1RouteTableAssociation817AEFF1) Resource creation Initiated
 18/45 | 12:31:10 PM | CREATE_IN_PROGRESS   | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet2/RouteTableAssociation (helloeksDefaultVpcPublicSubnet2RouteTableAssociation1728C296) Resource creation Initiated
 18/45 | 12:31:11 PM | CREATE_IN_PROGRESS   | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet1/NATGateway (helloeksDefaultVpcPublicSubnet1NATGateway67696503) Resource creation Initiated
 19/45 | 12:31:24 PM | CREATE_COMPLETE      | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet2/DefaultRoute (helloeksDefaultVpcPublicSubnet2DefaultRouteC8106461)
 20/45 | 12:31:25 PM | CREATE_COMPLETE      | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet1/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet1RouteTableAssociation63177EF1)
 21/45 | 12:31:25 PM | CREATE_COMPLETE      | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PrivateSubnet2/RouteTableAssociation (helloeksDefaultVpcPrivateSubnet2RouteTableAssociationD7E96AD6)
 22/45 | 12:31:25 PM | CREATE_COMPLETE      | AWS::EC2::Route                       | hello-eks/DefaultVpc/PublicSubnet1/DefaultRoute (helloeksDefaultVpcPublicSubnet1DefaultRoute01D8F765)
 23/45 | 12:31:26 PM | CREATE_COMPLETE      | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet1/RouteTableAssociation (helloeksDefaultVpcPublicSubnet1RouteTableAssociation817AEFF1)
 24/45 | 12:31:26 PM | CREATE_COMPLETE      | AWS::EC2::SubnetRouteTableAssociation | hello-eks/DefaultVpc/PublicSubnet2/RouteTableAssociation (helloeksDefaultVpcPublicSubnet2RouteTableAssociation1728C296)
 25/45 | 12:32:01 PM | CREATE_COMPLETE      | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.KubectlProvider.NestedStack/@aws-cdk--aws-eks.KubectlProvider.NestedStackResource (awscdkawseksKubectlProviderNestedStackawscdkawseksKubectlProviderNestedStackResourceA7AEBA6B)
 26/45 | 12:32:20 PM | CREATE_COMPLETE      | AWS::CloudFormation::Stack            | @aws-cdk--aws-eks.ClusterResourceProvider.NestedStack/@aws-cdk--aws-eks.ClusterResourceProvider.NestedStackResource (awscdkawseksClusterResourceProviderNestedStackawscdkawseksClusterResourceProviderNestedStackResource9827C454)
 26/45 | 12:32:25 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/Resource/CreationRole (helloeksCreationRole0B71AB61)
 26/45 | 12:32:26 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/Resource/CreationRole (helloeksCreationRole0B71AB61) Resource creation Initiated
 27/45 | 12:32:38 PM | CREATE_COMPLETE      | AWS::IAM::Role                        | hello-eks/Resource/CreationRole (helloeksCreationRole0B71AB61)
 27/45 | 12:32:43 PM | CREATE_IN_PROGRESS   | AWS::IAM::Policy                      | hello-eks/Resource/CreationRole/DefaultPolicy (helloeksCreationRoleDefaultPolicy74D6F223)
 27/45 | 12:32:44 PM | CREATE_IN_PROGRESS   | AWS::IAM::Policy                      | hello-eks/Resource/CreationRole/DefaultPolicy (helloeksCreationRoleDefaultPolicy74D6F223) Resource creation Initiated
 28/45 | 12:32:55 PM | CREATE_COMPLETE      | AWS::IAM::Policy                      | hello-eks/Resource/CreationRole/DefaultPolicy (helloeksCreationRoleDefaultPolicy74D6F223)
 29/45 | 12:32:59 PM | CREATE_COMPLETE      | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet2/NATGateway (helloeksDefaultVpcPublicSubnet2NATGatewayD1BD3D33)
 29/45 | 12:33:00 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-Cluster            | hello-eks/Resource/Resource/Default (helloeks5A23CE00)
 29/45 | 12:33:05 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet2/DefaultRoute (helloeksDefaultVpcPrivateSubnet2DefaultRouteF71A61E9)
 29/45 | 12:33:05 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet2/DefaultRoute (helloeksDefaultVpcPrivateSubnet2DefaultRouteF71A61E9) Resource creation Initiated
 30/45 | 12:33:15 PM | CREATE_COMPLETE      | AWS::EC2::NatGateway                  | hello-eks/DefaultVpc/PublicSubnet1/NATGateway (helloeksDefaultVpcPublicSubnet1NATGateway67696503)
 30/45 | 12:33:20 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet1/DefaultRoute (helloeksDefaultVpcPrivateSubnet1DefaultRouteAD89C6CD)
 30/45 | 12:33:21 PM | CREATE_IN_PROGRESS   | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet1/DefaultRoute (helloeksDefaultVpcPrivateSubnet1DefaultRouteAD89C6CD) Resource creation Initiated
 31/45 | 12:33:21 PM | CREATE_COMPLETE      | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet2/DefaultRoute (helloeksDefaultVpcPrivateSubnet2DefaultRouteF71A61E9)
 32/45 | 12:33:36 PM | CREATE_COMPLETE      | AWS::EC2::Route                       | hello-eks/DefaultVpc/PrivateSubnet1/DefaultRoute (helloeksDefaultVpcPrivateSubnet1DefaultRouteAD89C6CD)
32/45 Currently in progress: helloeks5A23CE00
 32/45 | 12:43:18 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-Cluster            | hello-eks/Resource/Resource/Default (helloeks5A23CE00) Resource creation Initiated
 33/45 | 12:43:19 PM | CREATE_COMPLETE      | Custom::AWSCDK-EKS-Cluster            | hello-eks/Resource/Resource/Default (helloeks5A23CE00)
 33/45 | 12:43:25 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/manifest-hello-kub/Resource/Default (helloeksmanifesthellokubC0EBC737)
 33/45 | 12:43:25 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroup               | hello-eks/DefaultCapacity/InstanceSecurityGroup (helloeksDefaultCapacityInstanceSecurityGroup8E2BB97C)
 33/45 | 12:43:25 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/DefaultCapacity/InstanceRole (helloeksDefaultCapacityInstanceRole4DA288E1)
 33/45 | 12:43:26 PM | CREATE_IN_PROGRESS   | AWS::IAM::Role                        | hello-eks/DefaultCapacity/InstanceRole (helloeksDefaultCapacityInstanceRole4DA288E1) Resource creation Initiated
 33/45 | 12:43:30 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroup               | hello-eks/DefaultCapacity/InstanceSecurityGroup (helloeksDefaultCapacityInstanceSecurityGroup8E2BB97C) Resource creation Initiated
 34/45 | 12:43:32 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroup               | hello-eks/DefaultCapacity/InstanceSecurityGroup (helloeksDefaultCapacityInstanceSecurityGroup8E2BB97C)
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:443 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA443B9F73FA5)
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:443 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA443B9F73FA5) Resource creation Initiated
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:ALL TRAFFIC (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BBALLTRAFFIC8C9AE92C)
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:ALL TRAFFIC (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BBALLTRAFFIC8C9AE92C) Resource creation Initiated
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/ControlPlaneSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:443 (helloeksControlPlaneSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB44308D25812)
 34/45 | 12:43:37 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/ControlPlaneSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:443 (helloeksControlPlaneSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB44308D25812) Resource creation Initiated
 35/45 | 12:43:38 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:443 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA443B9F73FA5)
 35/45 | 12:43:38 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:1025-65535 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA102565535EB22E3D3)
 35/45 | 12:43:38 PM | CREATE_IN_PROGRESS   | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:1025-65535 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA102565535EB22E3D3) Resource creation Initiated
 36/45 | 12:43:38 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:ALL TRAFFIC (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BBALLTRAFFIC8C9AE92C)
 37/45 | 12:43:38 PM | CREATE_COMPLETE      | AWS::IAM::Role                        | hello-eks/DefaultCapacity/InstanceRole (helloeksDefaultCapacityInstanceRole4DA288E1)
 38/45 | 12:43:38 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroupIngress        | hello-eks/ControlPlaneSecurityGroup/from CdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB:443 (helloeksControlPlaneSecurityGroupfromCdkEksStackhelloeksDefaultCapacityInstanceSecurityGroup2A4C16BB44308D25812)
 39/45 | 12:43:39 PM | CREATE_COMPLETE      | AWS::EC2::SecurityGroupIngress        | hello-eks/DefaultCapacity/InstanceSecurityGroup/from CdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA:1025-65535 (helloeksDefaultCapacityInstanceSecurityGroupfromCdkEksStackhelloeksControlPlaneSecurityGroupBF09ABDA102565535EB22E3D3)
 39/45 | 12:43:44 PM | CREATE_IN_PROGRESS   | AWS::IAM::InstanceProfile             | hello-eks/DefaultCapacity/InstanceProfile (helloeksDefaultCapacityInstanceProfileEAC17C4A)
 39/45 | 12:43:44 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/AwsAuth/manifest/Resource/Default (helloeksAwsAuthmanifest12726185)
 39/45 | 12:43:44 PM | CREATE_IN_PROGRESS   | AWS::IAM::InstanceProfile             | hello-eks/DefaultCapacity/InstanceProfile (helloeksDefaultCapacityInstanceProfileEAC17C4A) Resource creation Initiated
39/45 Currently in progress: helloeksmanifesthellokubC0EBC737, helloeksDefaultCapacityInstanceProfileEAC17C4A, helloeksAwsAuthmanifest12726185
 39/45 | 12:45:21 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/AwsAuth/manifest/Resource/Default (helloeksAwsAuthmanifest12726185) Resource creation Initiated
 40/45 | 12:45:21 PM | CREATE_COMPLETE      | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/AwsAuth/manifest/Resource/Default (helloeksAwsAuthmanifest12726185)
 40/45 | 12:45:41 PM | CREATE_IN_PROGRESS   | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/manifest-hello-kub/Resource/Default (helloeksmanifesthellokubC0EBC737) Resource creation Initiated
 41/45 | 12:45:42 PM | CREATE_COMPLETE      | Custom::AWSCDK-EKS-KubernetesResource | hello-eks/manifest-hello-kub/Resource/Default (helloeksmanifesthellokubC0EBC737)
 42/45 | 12:45:45 PM | CREATE_COMPLETE      | AWS::IAM::InstanceProfile             | hello-eks/DefaultCapacity/InstanceProfile (helloeksDefaultCapacityInstanceProfileEAC17C4A)
 42/45 | 12:45:50 PM | CREATE_IN_PROGRESS   | AWS::AutoScaling::LaunchConfiguration | hello-eks/DefaultCapacity/LaunchConfig (helloeksDefaultCapacityLaunchConfig590D7393)
 42/45 | 12:45:51 PM | CREATE_IN_PROGRESS   | AWS::AutoScaling::LaunchConfiguration | hello-eks/DefaultCapacity/LaunchConfig (helloeksDefaultCapacityLaunchConfig590D7393) Resource creation Initiated
 43/45 | 12:45:52 PM | CREATE_COMPLETE      | AWS::AutoScaling::LaunchConfiguration | hello-eks/DefaultCapacity/LaunchConfig (helloeksDefaultCapacityLaunchConfig590D7393)
 43/45 | 12:45:57 PM | CREATE_IN_PROGRESS   | AWS::AutoScaling::AutoScalingGroup    | hello-eks/DefaultCapacity/ASG (helloeksDefaultCapacityASG14F4C3D2)
 43/45 | 12:45:58 PM | CREATE_IN_PROGRESS   | AWS::AutoScaling::AutoScalingGroup    | hello-eks/DefaultCapacity/ASG (helloeksDefaultCapacityASG14F4C3D2) Resource creation Initiated
43/45 Currently in progress: helloeksDefaultCapacityASG14F4C3D2

 ✅  CdkEksStack

Outputs:
CdkEksStack.helloeksGetTokenCommandB7A8C164 = aws eks get-token --cluster-name helloeks5A23CE00-acf651404d6b4004b80b336e1f8d0c86 --region us-east-1 --role-arn arn:aws:iam::991225764181:role/CdkEksStack-AdminRole38563C57-XE9DS8R7U3HU
CdkEksStack.helloeksConfigCommand9213DBDB = aws eks update-kubeconfig --name helloeks5A23CE00-acf651404d6b4004b80b336e1f8d0c86 --region us-east-1 --role-arn arn:aws:iam::991225764181:role/CdkEksStack-AdminRole38563C57-XE9DS8R7U3HU

Stack ARN:
arn:aws:cloudformation:us-east-1:991225764181:stack/CdkEksStack/0b603cb0-5f20-11ea-af29-0e6b90c5010d
✔ ~/src/cdk/cdk-eks [master L|✚ 4…1]
12:46 $ aws eks get-token --cluster-name helloeks5A23CE00-acf651404d6b4004b80b336e1f8d0c86 --region us-east-1 --role-arn arn:aws:iam::991225764181:role/CdkEksStack-AdminRole38563C57-XE9DS8R7U3HU
{"kind": "ExecCredential", "apiVersion": "client.authentication.k8s.io/v1alpha1", "spec": {}, "status": {"expirationTimestamp": "2020-03-05T21:02:00Z", "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFTSUE2TlNOSTNGS1JGVDZGNFZXJTJGMjAyMDAzMDUlMkZ1cy1lYXN0LTElMkZzdHMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDIwMDMwNVQyMDQ4MDBaJlgtQW16LUV4cGlyZXM9NjAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JTNCeC1rOHMtYXdzLWlkJlgtQW16LVNlY3VyaXR5LVRva2VuPUZ3b0daWEl2WVhkekVMYiUyRiUyRiUyRiUyRiUyRiUyRiUyRiUyRiUyRiUyRndFYURFMmtQeENuYmZuSElzaEVLU0t6QWMyQmRDU3ZRdHNGNVU5MElZQ085NVNOM0o3Vk5ObGRSaXFaRzZXajNXWnBTelNnblFHcTVZc3JRTHhLY0MlMkJMY2UzWDNSTGdJSFBWOSUyRlZvWmJLRkFoMiUyQng0eU1yZm02byUyRndlUld0WnF2UkglMkZhY3lMVmZyTXdiYjJsOE1IbmFPNUdhQ3drRkJvJTJCNjNJeHU2akRzZEpIbFdXRThsTWxodW9HZndTd0hVeGdHbVolMkZvTGdFRE1aY2UlMkY4dCUyQkQlMkJ5eW1XN2pBbTZTciUyRkFlQmZlUnptUmdjc2h1bnh1WjFHNWs0b2RUbFpadnlVSHRxSTBJSEtJRExoZk1GTWkwSkNzWnl0dGNmT3ZDd1ZSWlljQUl4Q2JENGJ1UkhxVm14Q1pjVXA2UWtITGNFeWhpcTBBcUhGYkRyMUZvJTNEJlgtQW16LVNpZ25hdHVyZT05Y2ZhYTA0OTFmNGY2NmY0NDBmNjA2NDRhYWEzZGQxM2I3MDFlYTcwYjQwY2ZiN2E4YTdmZDU1ZWJkMjk1YzFh"}}
✔ ~/src/cdk/cdk-eks [master L|✚ 4…1]
12:48 $ aws eks update-kubeconfig --name helloeks5A23CE00-acf651404d6b4004b80b336e1f8d0c86 --region us-east-1 --role-arn arn:aws:iam::991225764181:role/CdkEksStack-AdminRole38563C57-XE9DS8R7U3HU
Added new context arn:aws:eks:us-east-1:991225764181:cluster/helloeks5A23CE00-acf651404d6b4004b80b336e1f8d0c86 to /Users/nbrand/.kube/config
```
## Option 4 - GitOps
This last option is the most in-line with the Kubernetes philosophy of a self-correcting control loop.  It is called *GitOps*, and it is starting to gain traction for synchronizing configuration files, and deploying CI/CD pipelines.  The basic idea is to use a git repository as a centralized **source of truth**. Any changes will be driven though the Git repository. Therefore, your team will get all the benefits of Git (logging, rollback, commit messages, PR, etc...) for your configuration management.

The two leading companies in this space are [Weaveworks](https://www.weave.works/technologies/gitops/), with their **Flux** tool and Intuit's **Argo** [project](https://argoproj.github.io/argo-cd/). Also, Google just jumped on the bandwagon with a new GitOps based application [manager](https://www.theregister.co.uk/2020/02/20/google_cloud_embraces_gitops_with_new_application_manager_for_kubernetes/). Actually, Weavworks and Argo are combining forces to create a new GitOps engine, called [Argo Flux](https://www.weave.works/blog/argo-flux-join-forces). I look forward to seeing what comes out of this enterprise.

Weaveworks is the same company that created **eksctl**, the preferred tooling to create/manage EKS clusters.  They have built an experimental GitOps feature into the `eksctl` CLI. The following example comes from their [documentation](https://eksctl.io/usage/experimental/gitops-flux/).

### eksctl and GitOps example using Flux

>    This is an experimental feature. To enable it, set the environment variable EKSCTL_EXPERIMENTAL=true. Experimental features are not stable and their command name and flags may change.

Installing Flux on the cluster is the first step towards a gitops workflow. To install it, you need a Git repository and an existing EKS cluster. Then run the following command:

`EKSCTL_EXPERIMENTAL=true eksctl enable repo --cluster=<cluster_name> --region=<region> --git-url=<git_repo> --git-email=<git_user_email>`

Or use a config file:

`EKSCTL_EXPERIMENTAL=true eksctl enable repo -f examples/01-simple-cluster.yaml --git-url=git@github.com:weaveworks/cluster-1-gitops.git --git-email=johndoe+flux@weave.works`

Full example:

``` shell
$ EKSCTL_EXPERIMENTAL=true ./eksctl enable repo --cluster=cluster-1 --region=eu-west-2  --git-url=git@github.com:weaveworks/cluster-1-gitops.git  --git-email=johndoe+flux@weave.works --namespace=flux
[ℹ]  Generating manifests
[ℹ]  Cloning git@github.com:weaveworks/cluster-1-gitops.git
Cloning into '/var/folders/zt/sh1tk7ts24sc6dybr5z9qtfh0000gn/T/eksctl-install-flux-clone-142184188'...
[ℹ]  Writing Flux manifests
[ℹ]  Applying manifests
[ℹ]  created "Namespace/flux"
[ℹ]  created "flux:Secret/flux-git-deploy"
[ℹ]  created "flux:Deployment.apps/memcached"
[ℹ]  created "flux:ServiceAccount/flux"
[ℹ]  created "ClusterRole.rbac.authorization.k8s.io/flux"
[ℹ]  created "ClusterRoleBinding.rbac.authorization.k8s.io/flux"
[ℹ]  created "CustomResourceDefinition.apiextensions.k8s.io/helmreleases.helm.fluxcd.io"
[ℹ]  created "flux:Deployment.apps/flux"
[ℹ]  created "flux:Service/memcached"
[ℹ]  created "flux:Deployment.apps/flux-helm-operator"
[ℹ]  created "flux:ServiceAccount/flux-helm-operator"
[ℹ]  created "ClusterRole.rbac.authorization.k8s.io/flux-helm-operator"
[ℹ]  created "ClusterRoleBinding.rbac.authorization.k8s.io/flux-helm-operator"
[ℹ]  Waiting for Helm Operator to start
[ℹ]  Helm Operator started successfully
[ℹ]  Waiting for Flux to start
[ℹ]  Flux started successfully
[ℹ]  Committing and pushing manifests to git@github.com:weaveworks/cluster-1-gitops.git
[master ec43024] Add Initial Flux configuration
 Author: Flux <johndoe+flux@weave.works>
14 files changed, 694 insertions(+)
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 4 threads
Compressing objects: 100% (6/6), done.
Writing objects: 100% (6/6), 2.09 KiB | 2.09 MiB/s, done.
Total 6 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3), completed with 3 local objects.
To github.com:weaveworks/cluster-1-gitops.git
   5fe1eb8..ec43024  master -> master
[ℹ]  Flux will only operate properly once it has write-access to the Git repository
[ℹ]  please configure git@github.com:weaveworks/cluster-1-gitops.git  so that the following Flux SSH public key has write access to it
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDYYsPuHzo1L29u3zhr4uAOF29HNyMcS8zJmOTDNZC4EiIwa5BXgg/IBDKudxQ+NBJ7mknPlNv17cqo4ncEq1xiQidfaUawwx3xxtDkZWam5nCBMXEJwkr4VXx/6QQ9Z1QGXpaFwdoVRcY/kM4NaxM54pEh5m43yeqkcpRMKraE0EgbdqFNNARN8rIEHY/giDorCrXp7e6AbzBgZSvc/in7Ul9FQhJ6K4+7QuMFpJt3O/N8KDumoTG0e5ssJGp5L1ugIqhzqvbHdmHVfnXsEvq6cR1SJtYKi2GLCscypoF3XahfjK+xGV/92a1E7X+6fHXSq+bdOKfBc4Z3f9NBwz0v
```

At this point Flux and the Helm Operator should be installed in the specified cluster. The only thing left to do is to give Flux write access to the repository. Configure your repository to allow write access to that ssh key, for example, through the Deploy keys if it lives in GitHub.

``` shell
$ kubectl get pods --namespace flux
NAME
flux-699cc7f4cb-9qc45
memcached-958f745c-qdfgz
flux-helm-operator-6bc7c85bb5-l2nzn
```

To deploy a new workload on the cluster using gitops just add a kubernetes manifest to the repository. After a few minutes you should see the resources appearing in the cluster.

### Installing components from a Quick Start profile

`eksctl` provides an application development Quick Star profile which can install the following components in your cluster: - Metrics Server - Prometheus - Grafana - Kubernetes Dashboard - FluentD with connection to CloudWatch logs - CNI, present by default in EKS clusters - Cluster Autoscaler - ALB ingress controller - Podinfo as a demo application

To install those components the command `generate profile` can be used:

`EKSCTL_EXPERIMENTAL=true eksctl generate profile --config-file=<cluster_config_file> --git-url git@github.com:weaveworks/eks-quickstart-app-dev.git --profile-path <output_directory>`

For example:

``` shell
$ EKSCTL_EXPERIMENTAL=true eksctl generate profile  --config-file 01-simple-cluster.yaml --git-url git@github.com:weaveworks/eks-quickstart-app-dev.git --profile-path my-gitops-repo/base/
[ℹ]  cloning repository "git@github.com:weaveworks/eks-quickstart-app-dev.git":master
Cloning into '/tmp/quickstart-224631067'...
warning: templates not found /home/.../.git_template
remote: Enumerating objects: 75, done.
remote: Counting objects: 100% (75/75), done.
remote: Compressing objects: 100% (63/63), done.
remote: Total 75 (delta 25), reused 49 (delta 11), pack-reused 0
Receiving objects: 100% (75/75), 19.02 KiB | 1.19 MiB/s, done.
Resolving deltas: 100% (25/25), done.
[ℹ]  processing template files in repository
[ℹ]  writing new manifests to "base/"

$ tree my-gitops-repo/base
$ tree base/
base/
├── amazon-cloudwatch
│   ├── cloudwatch-agent-configmap.yaml
│   ├── cloudwatch-agent-daemonset.yaml
│   ├── cloudwatch-agent-rbac.yaml
│   ├── fluentd-configmap-cluster-info.yaml
│   ├── fluentd-configmap-fluentd-config.yaml
│   ├── fluentd-daemonset.yaml
│   └── fluentd-rbac.yaml
├── demo
│   └── helm-release.yaml
├── kubernetes-dashboard
│   ├── dashboard-metrics-scraper-deployment.yaml
│   ├── dashboard-metrics-scraper-service.yaml
│   ├── kubernetes-dashboard-configmap.yaml
│   ├── kubernetes-dashboard-deployment.yaml
│   ├── kubernetes-dashboard-rbac.yaml
│   ├── kubernetes-dashboard-secrets.yaml
│   └── kubernetes-dashboard-service.yaml
├── kube-system
│   ├── alb-ingress-controller-deployment.yaml
│   ├── alb-ingress-controller-rbac.yaml
│   ├── cluster-autoscaler-deployment.yaml
│   └── cluster-autoscaler-rbac.yaml
├── LICENSE
├── monitoring
│   ├── metrics-server.yaml
│   └── prometheus-operator.yaml
├── namespaces
│   ├── amazon-cloudwatch.yaml
│   ├── demo.yaml
│   ├── kubernetes-dashboard.yaml
│   └── monitoring.yaml
└── README.md
```

After running the command, add, commit and push the files:

``` shell
cd my-gitops-repo/
git add .
git commit -m "Add application development quick start components"
git push origin master
```

After a few minutes, Flux and Helm should have installed all the components in your cluster.

## Summary
I was able to give four options on how to quickly and easily get a new EKS cluster configured with a variety of useful add-ons and software packages.

1. BASH scripts
2. Helmfile
3. Cloud Development Kit
4. GitOps

I personally use option #2 the most, since most of my clusters are disposable. However, I have used all the options at one time or another. Regardless of your choice, any sort of automation is better, quicker and more reliable than installing and managing software manually.

Of course, this is just a baseline to get ones cluster up and productive. For ongoing Kubernetes deployments and management, I suspect that GitOps will become more common once the tooling matures.  It offers the greatest promise of power and control, and the one most in line with Kubernetes philosophy of declarative infrastructure combined with a self-correcting control loop.

***

### References

* [ekstender](https://github.com/mreferre/ekstender)
* [Helmfile](https://github.com/roboll/helmfile)
* [Helmfile Best Practices](https://github.com/roboll/helmfile/blob/master/docs/writing-helmfile.md)
* [Helmfile - its like helm for helm](https://medium.com/@naseem_60378/helmfile-its-like-a-helm-for-your-helm-74a908581599)
* [Setting  up your Kubernetes cluster with Helmfile](https://itnext.io/setup-your-kubernetes-cluster-with-helmfile-809828bc0a9f)
* [CDK and EKS module](https://docs.aws.amazon.com/cdk/api/latest/docs/aws-eks-readme.html)
* [CDK8s - Creating K8 templates using higher level languages](https://github.com/awslabs/cdk8s)
* [Using CloudFormation and Helm Charts](https://aws.amazon.com/blogs/infrastructure-and-automation/using-aws-cloudformation-to-deploy-software-into-amazon-eks-clusters/?nc1=b_rp)
* [Weaveworks gitops with eksctl](https://eksctl.io/usage/experimental/gitops-flux/)
* [GitOps Helm workshop](https://helm.workshop.flagger.dev/intro/)
* [EKS and GitOps workshop](https://eks.handson.flagger.dev/)
* [Google launches gitops tool](https://www.theregister.co.uk/2020/02/20/google_cloud_embraces_gitops_with_new_application_manager_for_kubernetes/)

---
layout: post
title:  "How to easily create an EKS cluster"
date:   2018-07-01 09:00:00 -0400
categories: aws kubernetes
---
![EKS cluster creation](/images/what-is-eks.png)

Launching an EKS (Elastic Kubernetes Service) cluster is **NOT** as easy as it should be. If you follow the documentation, these are the steps to take:
1. Create an IAM Service Role for EKS.  This only has to be done once.
2. You launch a CloudFormation script to create a dedicated VPC (optional). One needs three subnets, one for each AZ that the masters will be deployed into.
3. You create your EKS management plane.  This just takes a few clicks in the Console, or a line or two in the CLI.
4. Edit the KUBECONFIG file with proper data for server endpoint, certificate authority and IAM info for the heptio authenticator.
5. You launch another CloudFormation script, in order to launch your worker nodes.
6. You copy the worker node *NodeInstanceRole* from the CF output, and then use it as input for an authorization config-map.  This allows worker nodes to dynamically join the cluster in a secure manner.

If you do this a few times, you will find it not too onerous. Also, you can use [Terraform](https://www.terraform.io/docs/providers/aws/guides/eks-getting-started.html) scripting or other tools to glue everthing together. Still, it is **NOT** easy.  If you have worked on GCP or Azure, it is a single line of commands that launches both your masters and workers.  This is 2018 AWS - let's polish that API!

Fortunately, the Kubernetes open-source community being what it is, a partner has already solved this problem. [Weaveworks](https://www.weave.works) has created a github repo caleld [*eksctl*](https://github.com/weaveworks/eksctl), and they describe it as "a CLI for Amazon EKS".  Well, they are right.  In a single line of code, they automate the entire process described above, and brings the ability to launch both masters **and** workers into the realm of us mortals.
![eksctl cluster creation](/images/eksctl.png)

#### Comparison of CLI commands to launch Kubernetes clusters

Cloud Provider | Command | Notes 
-------------- | ------- | -----
eksctl (AWS) | eksctl create cluster | 3 masters, 2 workers
AWS | aws eks create-cluster --name devel --role-arn arn:aws:iam::111122223333:role/eks-service-role-AWSServiceRoleForAmazonEKS-EXAMPLEBKZRQR --resources-vpc-config subnetIds=subnet-a9189fe2,subnet-50432629,securityGroupIds=sg-f5c54184 | 3 masters only
GCP | gcloud container clusters create CLUSTER_NAME --zone COMPUTE_ZONE | 1 master, 3 workers
Azure | az acs create --orchestrator-type kubernetes --resource-group myResourceGroup --name myK8sCluster --generate-ssh-keys | 1 master, 3 workers

___
**References:**

[eksctl repo](http://localhost:4000/aws/2018/07/01/Easily-create-an-EKS-cluster.html)

[EKS documentation - html](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) or
 [EKS documentation - PDF](https://docs.aws.amazon.com/eks/latest/userguide/eks-ug.pdf)

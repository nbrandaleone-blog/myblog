---
layout: post
title:  "Elastic Kubernetes Service and roles on AWS"
date:   2018-06-24 21:09:01 -0500
categories: aws kubernetes
---
![IAM authentication](/images/authenticator.png)
A few weeks ago AWS released EKS or their managed Elastic Kubernetes Service. What is amazing, is that it **IS** Kubernetes (version 1.10).  Of course, there are some AWS subleties. The most significant are:
1. **VPC Support.** A CNI networking plug-in which allows the containers to run within the VPC CIDR address space.
2. **IAM Authentication**. All *kubectl* calls will be authenticated using an IAM user/role. This requires a separate executable to be installed on both the client and K8 masters.

In terms of the IAM roles, I found the documentation good, but scattered, so I will describe how I created an EKS cluster, using an IAM role. The advantage of this method is that the *kubectl* configuration file can be shared freely among DevOps staff.  If they have access to the proper role, then they will be able to hit the Kubernetes cluster securely.  If not, the EKS masters will drop the calls.  Once authenticated, the calls will be passed on to K8 RBAC for normal authorization and processing. The new IAM process is meant to handle the authentication process of the standard AAA security model.

I have found a common misconception. By default, EKS maps the IAM user who created the EKS cluster into the *system:masters* group.  This group is linked to the *cluster-admin* cluster role.  This kubernetes role allows super-user access to perform any action on any resource in the cluster. While this *AWS* user (or IAM group/role) has automatic access into the kubernetes cluster - **no one else does!**  How do we give cluster access to others?

First of all, follow the EKS [instructions](https://docs.aws.amazon.com/eks/latest/userguide/eks-ug.pdf).  You will need to install the heptio authenticaor client on your laptop/workstation.  The server component will be automatically installed on your EKS cluster. The most important sections are: *Managing Cluster Authentication* through *Managing Users or IAM Roles for your Cluster*

There are 2 common methods:
1. Pass around the user credentials who created the cluster. This is a **bad** idea, but certainly possible for a small DevOps team. You can either then assume that role (using AWS profiles), or have *kubectl* reference that role directly.
2. Create a new IAM role (I use *KubernetesAdmin*), and bind that role into the *system:masters* group manually.  This is the **better** idea.

Also, you must update the KUBECONFIG file you are using to reference this new role.

### Create an AWS IAM Role
{% highlight bash %}
# get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --output text --query 'Account')

# define a role trust policy that opens the role to users in your account (limited by IAM policy)
POLICY=$(echo -n '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":"arn:aws:iam::'; echo -n "$ACCOUNT_ID"; echo -n ':root"},"Action":"sts:AssumeRole","Condition":{}}]}')

# create a role named KubernetesAdmin (will print the new role's ARN)
aws iam create-role \
  --role-name KubernetesAdmin \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "$POLICY" \
  --output text \
  --query 'Role.Arn'
{% endhighlight %}

The above IAM role can be assumed by anyone in the users AWS account.  Of course, this can be limited to appropriate users/group according to the principle of *least priviledge*. Also, this role has **NO** policy.  It does absolutely *nothing*, except to act as a authentication token. It would be best-practices to create an IAM group, with an in-line policy only allowing this IAM group to assume the KubernetesAdmin role.

{% highlight bash %}
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::ACCOUNT-ID-WITHOUT-HYPHENS:role/KubernetesAdmin"
  }
}
{% endhighlight %}

### Bind IAM role to Kubernetes *system:masters* group
AWS has created a [config map](https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-06-05/aws-auth-
cm.yaml) for you called *aws-auth-cm.yaml*. You must edit/update it, to use the newly created IAM role, and bind it to the *system:masters* group. My cluster's name is *EKS-attractive-gopher-1529340392*.

{% highlight bash %}
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
    namespace: kube-system
    data:
      mapRoles: |
        - rolearn: arn:aws:iam::<account ID>:role/EKS-attractive-gopher-1529340392-NodeInstanceRole-9EQ7OZE3PQ34
	  username: system:node:{{EC2PrivateDNSName}}
	  groups:
            - system:bootstrappers
	    - system:nodes
	- rolearn: arn:aws:iam::<account ID>:role/KubernetesAdmin
	  username: kubernetes-admin
	  groups:
	    - system:masters
{% endhighlight %}

### Update the KUBECONFIG file to reference the new IAM role
{% highlight bash %}
# boilerplate yaml...
- name: kubernetes-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: heptio-authenticator-aws
      args:
      - "token"
      - "-i"
      - "attractive-gopher-1529340392"
      - "-r"
      - "arn:aws:iam::<account ID>:role/KubernetesAdmin"
      env: null
{% endhighlight %}

You are done.  You can now pass this KUBECONFIG file to any DevOps engineer or Developer who needs access to the Kubernetes cluster (i.e. Save the config file to a private S3 bucket). Even if the config file becomes lost, only users who have access into the AWS account *AND* have access to the role will be able to get into your K8 cluster. I think that this is a **HUGE** win over traditional kubernetes, and makes EKS very much in-line with other AWS services.

___
**References:**

[EKS documentation - html](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) or
[EKS documentation - PDF](https://docs.aws.amazon.com/eks/latest/userguide/eks-ug.pdf)

[heptio authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator)

[Deploying the Heptio Authenticator](https://aws.amazon.com/blogs/opensource/deploying-heptio-authenticator-kops/)

[5 things I wish I'd known before setting up heptio authenticator](https://blog.lola.com/5-things-i-wish-id-known-before-setting-up-heptio-authenticator)

---
layout: post
title:  "Interacting with EKS via Lambda"
date:   2018-09-03 09:00:00 -0400
categories: aws
---
![EKS authentications](/images/authenticator.png)

<details>
<summary><strong>Blog and Code update</strong></summary>
<p>
Due to feedback from colleagues (thank you Chris Hein and Paul Maddox!) I have significantly cleaned up the code needed to interact with EKS using the kubernetes go client. This blog post has been edited since it was originally released on August 26, 2018.
</p>
<hr>
</details>

One typically interacts with a Kubernetes cluster through _kubectl_. However, that only really works for interactive commands. When you want to automate something, you need to script it. Fortunately, there are several excellent kubernetes client [libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/). The officially supported one is written in [Go](https://github.com/kubernetes/client-go/), simply because kubernetes is written in Go.

Now, there is a hurdle that must be overcome when scripting for an EKS cluster. EKS, like all AWS services, uses _IAM_ for authentication. This means, that any scripts also need to somehow use IAM. The official Go client library supports external authenticators (now called credential [plugins](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)), as defined in your local KUBECONFIG file. Unfortunately, this support for external authenticators is not yet supported in other libraries (who doesn't love [Python](https://github.com/kubernetes-client/python) these days?!). So, we will use the Go client library to demonstrate how to script interactions with your EKS cluster.

## Using the Go client

Grab the sample [code](https://github.com/kubernetes/client-go/tree/master/examples/out-of-cluster-client-configuration) from the repository, for __out-of-cluster-client-configuration__.

### Running this example

Make sure your `kubectl` is configured and pointed to a cluster. You also need your [authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator) plugin installed on your system, and your _kubectl_ config file referencing it. Notice how the arguments being passed to the authenticator references a __role__. That is an __IAM__ role that the user must be able to execute for authentication to succeed. The [authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator) repo and AWS [docs](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html) describes how you must create a role to be used for authentication, and then use a __ConfigMap__ to update your cluster to recognized that particular role.

**Kubectl Configuration file**
```yaml
# [...]
users:
- name: kubernetes-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "CLUSTER_ID"
        - "-r"
        - "ROLE_ARN"
  # no client certificate/key needed here!
```
**Kubernetes ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    # statically map arn:aws:iam::000000000000:role/KubernetesAdmin to cluster admin
    - roleARN: arn:aws:iam::000000000000:role/KubernetesAdmin
      username: kubernetes-admin
      groups:
      - system:masters
# [... for worker node authentication]
```
I created a _kubernetesAdmin_ role for all my trusted colleagues who need access to the cluster by making us all part of an IAM group, which has the following role access.
```yaml
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
```

Run `kubectl get nodes` to confirm that everything is working.

Run this application with:

{% highlight bash %}
$ cd out-of-cluster-configuration
$ go build -o app main.go
$ ./app
There are 12 pods in the cluster
Pod example-xxxxx in namespace default not found
{% endhighlight %}

---

## Creating a Lambda function
In order to run a Lambda function that can interact with our cluster, there is some preparation work required:
1. Pass the appropriate EKS cluster name, AWS region and IAM authentication role required for EKS access via environmental variables into the lambda function 
2. Import the [heptio](https://github.com/kubernetes-sigs/aws-iam-authenticator/tree/master/pkg/token) authenticator - now a [Kubernetes SIG](https://github.com/kubernetes-sigs) - package into the lambda function, so the go client can generate an authentication token
3. Create a RESTful call into your Kubernets API server, inserting the authentication Bearer Token 
4. Ensure the lambda function has appropriate access to assume the K8 necessary authentication role needed for token generation

### Lambda role
When you create a lambda function, you must assign it an IAM role. The role I created to accompany
my lambda function has 3 associated policies:
- A CloudWatch policy. This is standard for all lambda functions, so they log output and errors
- An EKS service policy, so the function can call into the EKS service (i.e. to list clusters, etc...)
- An assume policy, which allows the lambda function to assume the authentication role required by the EKS cluster
![EKS authentications](/images/lambda-trust.png)

### Bearer Token
It is possible to directly access the API server, if the RESTful call is in the right format. This is described in the kubernetes [documentation](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/). Here is an example using _curl_. Notice the Bearer Token.  We use the same element to pass the encoded IAM role which is generated by the external/plug-in authenticator.  

We create a similar RESTful call using the go client.

```bash
$ APISERVER=$(kubectl config view | grep server | cut -f 2- -d ":" | tr -d " ")
$ TOKEN=$(kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t')
$ curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

The API server decodes the bearer token, and then uses AWS STS, to verify that the user/lambda function has legitimate access to the given role.  This API server decoding is roughly equivalent to the following:

```bash
curl -X GET \
     -H "accept: application/json" \
     -H "x-k8s-aws-id: $CLUSTERNAME" \
     $(aws-iam-authenticator token -i $CLUSTERNAME | \
        jq -r ".status.token" | \
        sed 's/k8s-aws-v1\.//' | \
        base64 -D)
```

The role must be properly mapped via a ConfigMap (as previously shown above) to a kubernetes cluster role, before the command can be accepted.

## Summary
Once everything is in place, and your function is uploaded - invoke it.

```bash
$ aws lambda invoke --function-name eksClient \
		--region us-east-1 \
		--log-type Tail - \
		| jq '.LogResult' -r | base64 -D

    START RequestId: b4e7770a-a961-11e8-9d7a-e18d853267af Version: $LATEST
    There are 31 pods in the cluster
    Pod example-xxxxx in namespace default not found
    END RequestId: b4e7770a-a961-11e8-9d7a-e18d853267af
    REPORT RequestId: b4e7770a-a961-11e8-9d7a-e18d853267af	Duration: 4859.56 ms	Billed Duration: 4900 ms 	Memory Size: 256 MB	Max Memory Used: 145 MB
```

So, we have demonstrated how it is possible to use a lambda function to interact with your EKS cluster.
I would recommend additional error checking be added before you use this lambda for production use.

The working code is located in the following [repository](https://github.com/nbrandaleone/eksClient). 

___

#### **References:**

[Go client package](https://github.com/kubernetes/client-go)

[AWS Go SDK version 1](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/welcome.html)

[AWS Go SDK version 2](https://aws.amazon.com/blogs/developer/aws-sdk-for-go-2-0-developer-preview/)

[AWS Lambda using Go](https://aws.amazon.com/blogs/compute/announcing-go-support-for-aws-lambda/)

[Repo for blog](https://github.com/nbrandaleone/eksClient)

[Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)

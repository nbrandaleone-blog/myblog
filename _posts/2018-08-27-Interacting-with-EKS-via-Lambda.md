---
layout: post
title:  "Interacting with EKS via Lambda"
date:   2018-08-26 09:00:00 -0400
categories: aws
---
![EKS authentications](/images/authenticator.png)

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
1. Download the binary authenticator plugin, so the go client can call it as part of the authentication process
2. Create a local _kubectl_ configuration file referencing the plugin, and the appropriate role
3. Ensure the lambda function has appropriate access to assume the K8 authentication role

### Download the authenticator
Once the lambda function starts, I download a local copy of the authenticator program into "/tmp".
This allows for the go client to execute the binary, and generate the proper authentication token.

```go
func getAuthenticator() {
// This function gets the "aws-iam-authenticator" binary, and installs it in /tmp.

  const authURL = "https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator"
  var netClient = &http.Client{
    Timeout: time.Second * 10,
  }  

  resp, err := netClient.Get(authURL)
  check(err)

  defer resp.Body.Close()
  body, err := ioutil.ReadAll(resp.Body)
  check(err)

  // write binary to /tmp
  err = ioutil.WriteFile("/tmp/aws-iam-authenticator", body, 0755)
  check(err)
}
```

### Create a _kubectl_ configuration file
In order to create a _kubectl_ configuration file, I leverage Go templates, which allow for easy substitution of values.
Although this file could be hard-coded, by using a template I increase the flexibility of this function by allowing it to be reused for other clusters.

**NOTE:** The below template should have a second pair of braces around each variable. I dropped the second pair for increased visibility, as the blog was munging them.

```go
func buildConfig(){
// This function creates a KUBECONFIG file, using a template structure.

  t := template.New("KUBECONFIG")
  text := `
apiVersion: v1
clusters:
- cluster:
    server: {.Server}
    certificate-authority-data: {.CertificateAuthority}
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: aws
  name: aws
current-context: aws
kind: Config
preferences: {}
users:
- name: aws
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      command: /tmp/aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "{.Name}"
        - "-r"
        - "{.Role}"
`
// function continues ...
```

### Lambda role
When you create a lambda function, you must assign it an IAM role. The role I created to accompany
my lambda function has 3 associated policies:
- A CloudWatch policy. This is standard for all lambda functions, so they log output and errors
- An EKS service policy, so the function can call into the EKS service (i.e. to list clusters, etc...)
- An assume policy, which allows the lambda function to assume the authentication role required by the EKS cluster
![EKS authentications](/images/lambda-trust.png)

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
While the lambda function is not recommended for production use as is, it shows how
such a task can be accomplished.

The working code is located in the following [repository](https://github.com/nbrandaleone/eksClient).  Before you use it yourself, please review and improve where necessary. Also, there are several spots where values are hard-coded and should take environmental variables for greater flexibility. Also, the function should support Lambda _context_, but since it is not the target of an event, I did not import or use that package..

___

#### **References:**

[Go client program](https://github.com/kubernetes/client-go)

[AWS Go SDK version 1](https://docs.aws.amazon.com/sdk-for-go/v1/developer-guide/welcome.html)

[AWS Go SDK version 2](https://aws.amazon.com/blogs/developer/aws-sdk-for-go-2-0-developer-preview/)

[AWS Lambda using Go](https://aws.amazon.com/blogs/compute/announcing-go-support-for-aws-lambda/)

[Repo for blog](https://github.com/nbrandaleone/eksClient)

[Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)

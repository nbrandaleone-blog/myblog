---
layout: post
title:  "Don't Do This!"
date:   2018-10-09 09:00:00 -0400
categories: aws kubernetes
---
![EKS authentications](/images/authenticator.png)

In this post, I am going to describe how you can turn off RBAC controls on your EKS cluster.  This also circumvents IAM authentication. Now, this essentially opens up your cluster to attack, since all EKS clusters have pubic API endpoints. The cluster is still reasonably secure, since an attacker would have to know the DNS name/IP address of the cluster, and the Service Account token. Still, I would **NOT** recommend it except for testing, or for using functionality that is not yet compatible with IAM credentials. 

One day PrivateLink will become available for EKS.  When that happens, turning off RBAC will be reasonable, since the cluster will be isolated from the Internet. Not today though...

To effectively disable RBAC, global permissions can be applied granting full access:
```bash
kubectl create clusterrolebinding permissive-binding \
  --clusterrole=cluster-admin \
  --user=admin \
  --user=kubelet \
  --group=system:serviceaccounts
```

## Testing using a new KUBECONFIG

Now, in order to test this out, I create a new KUBECONFIG, but I removed the IAM credentials and added a serviceaccount token.

  - List the secrets with `kubectl get secrets`, and one should see one named similar to `default-token-xxxxx`. Copy that token name for use below
  - Get the certificate with `kubectl get secret <secret name> -o jsonpath='{.data.ca\.crt}'`
  - Retrieve the token with `kubectl get secret <secret name> -o jsonpath='{.data.token}' | base64 --decode`

Here is a sample config, with some details redacted:

**Kubectl Configuration file**
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRU...
    server: https://3D5...397D66C9.yl4.us-east-1.eks.amazonaws.com
  name: development
contexts:
- context:
    cluster: development
    user: aws
  name: aws
current-context: aws
kind: Config
preferences: {}
users:
- name: aws
  user:
    token: eyJhbGci...
```

## Execute commands without the IAM authenticator

```bash
kubectl --kubeconfig ./eks-test get nodes

NAME                              STATUS   ROLES    AGE   VERSION
ip-192-168-100-225.ec2.internal   Ready    <none>   37d   v1.10.3
ip-192-168-167-0.ec2.internal     Ready    <none>   37d   v1.10.3
ip-192-168-233-207.ec2.internal   Ready    <none>   37d   v1.10.3
```

## Summary

I have shown how it is possible to turn off perhaps the greatest security feature AWS has added to its managed Kubernetes service (EKS). Clearly, this is **not** recommended.  However, once PrivateLink becomes available it may be something worth considering.

---


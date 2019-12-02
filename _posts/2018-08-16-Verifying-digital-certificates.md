---
layout: post
title:  "Verifying EKS digital certificates"
date:   2018-08-16 09:00:00 -0400
categories: aws kubernetes
---
![TLS flow](/images/tls-flow.png)

When one creates a new EKS cluster, the management plane returns the following bits of information to you:

1. The public URL of the Load-Balancer which talks to your 3 deployed masters (i.e. API server endpoint)
2. The Certificate Authority data key
3. Some other odds and ends, like the Cluster ARN, name, security groups, version and status

![AWS EKS dashboard](/images/EKS-management.png)

You can get #1 and #2 from the AWS CLI using the following commands:

{% highlight bash %}
$ aws eks describe-cluster --name eks --query cluster.endpoint --region us-west-2
"https://715065595AECC5FB4F2F3811FBA82A45.yl4.us-west-2.eks.amazonaws.com"

$ aws eks describe-cluster --name eks --query cluster.certificateAuthority.data --region us-west-2
"LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWT
VJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRFNE1EWXhNekl4TlRZeU5Wb1hEVEk0TURZeE1ESXhOVFl5TlZvd0Z
URVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBT
29wCmllRWpnZlpReU01dDN4aW8rT1Bici9obWRJRjNza1gvNlZZVGZxYWJUNlJyb05WRnhVWlRJVWpIOGRKOTErQVYKdmlOamF
VbEJRbTZnZVJEUnRYWUxWcjU5cWpDZ0NIN21FbU1wN3pZLzBTajYxd0xHaXVsQmFleURSZVdGaUdPQwoxTzU2TmtqRTJrWkhRU
StjNWluMFZZK1UwVlNmUVRZNEVLS2dpQ0czNVhqUERNaEttMFZmb1BtbXhldnNZVlpyCm1DU2pFR1crb28yR0pTSm1GRmVCekp
sOWlRWkRsbUEzdW1KeWtWSFZTcE5ZM3dlTng0SDJLY3JkREpXdGpxbzQKVWd1RjZyWVp2K01nRDZ6MFRLbDVIdTZXRzdCS2w0a
0k2RkF1QnVLNTFUSTV2NmRuVkRNVGZ3eUpZTEdBNURIVgpTd1RlV1puUjFvMGJVeEVLMThjQ0F3RUFBYU1qTUNFd0RnWURWUjB
QQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFLbVA2SHdQSkd3U
StGSGxBM2pmKys4ejFRTS8KNG0vMUZoanY2Y3QrWUpYSitxZk42ZmxXeXZVRlJMM3Vsd1l5R0lPUEw3WVBvK2l5bnJyUlkzU1l
mcUZSeElCdQpoTTk5Z1VVL3lDK1ZleGFMNmswZGlQSTd0M1Q5bVhLay9NWk5nbHB2UWJ4cjRGTmR2NUhQTmVZR2pHWEtEVWk0C
nAxYkxUZktyZWFDYmp5REFxWTFCcFlCM2krcDNCOFFSejVDc3p2QXZFMjFMSEs5dXZ4cDhzL2lPUWduaDJVYXcKT09oK0gzQW9
0d1pZMXJ6YWJnZlFwdkIzY2hvekowMDZxbm9LUC9NVFZOalZGa3FMRHV5UWViTTZUZlMrNE84NwoxTCtER1ZZY25wZUR1cjdNN
3pRTGZVMExSckYxeURxQ2RMbmRIdG8xcG0wZWhGOVl4NmRmYVZoNUlCMD0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo="
{% endhighlight %}

### What is the CA data key useful for?
It is common practices for Kubernetes clusters to self-signed their digital certificates.
I often get from Security Practictioners the hairy eyeball when this fact is discussed. Why not use "real" certificates, that are signed by trusted CAs.  Well, some people do that, but for the rest of us, you are simply not gaining any signficant security benefits, and you are creating more work for yourself.  You see, Kubernetes clusters use many digital certificates in all aspects of managaging a cluster. For example, each node has its own digital certificate to verify its authenticity.  If you have a 100 nodes, are you going to register 100 "real" certificates - **no**, I don't think so. Still, it would not shock me if in some future Kubernets releases the masters automatically generate a "real" digital certificate by leveraging [Let's Encrypt](https://letsencrypt.org), for example. This may increase security at least a little bit, because let's be honest... we all click through the browser warnings, don't we?!

For the uber-paranoid, please see this [blog](https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/) post by Julia Evans, where she outlines at least 5 different ways of setting up a cluster using different CA strategies. These strategies won't work for EKS or other managed Kubernetes clusters, but if you want to roll your own using [KOPS](https://github.com/kubernetes/kops), please check out her comments.

So, we ARE going to use a self-signed digitial certificate for the master nodes - get with the program.  While all the tooling will automatically confirm that the certificates are valid, it would be nice if we could manually confirm it as well.  That is what we are going to do in the rest of this blog post.

#### Get the certificates in the proper format
The public key provided by the AWS management plane is the _signing_ key.  It is *NOT* the key used in TLS certificates or anywhere else. Since it signs other keys, it is the **root** CA for the cluster. In order to use it, we must be aware it has been _base64_ formatted.  Let's undo that...

Also, if you examine the output below carefully, you will see the following line: **CA:TRUE**

{% highlight bash %}
$ echo <LS0tLS...tLS0tLQo=> | base64 -D > ca_cert.pem

$ cat ca_cert.pem
-----BEGIN CERTIFICATE-----
MIICyDCCAbCgAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTE4MDYxMzIxNTYyNVoXDTI4MDYxMDIxNTYyNVowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAOop
<...>
1L+DGVYcnpeDur7M7zQLfU0LRrF1yDqCdLndHto1pm0ehF9Yx6dfaVh5IB0=
-----END CERTIFICATE-----

$ openssl x509 -in ca_cert -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 0 (0x0)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jun 13 21:56:25 2018 GMT
            Not After : Jun 10 21:56:25 2028 GMT
        Subject: CN=kubernetes
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:ea:29:89:e1:23:81:f6:50:c8:ce:6d:df:18:a8:
                    f8:e3:db:af:f8:66:74:81:77:b2:45:ff:e9:56:13:
                    7e:a6:9b:4f:a4:6b:a0:d5:45:c5:46:53:21:48:c7:
                    f1:d2:7d:d7:e0:15:be:23:63:69:49:41:42:6e:a0:
                    79:10:d1:b5:76:0b:56:be:7d:aa:30:a0:08:7e:e6:
                    12:63:29:ef:36:3f:d1:28:fa:d7:02:c6:8a:e9:41:
                    69:ec:83:45:e5:85:88:63:82:d4:ee:7a:36:48:c4:
                    da:46:47:41:0f:9c:e6:29:f4:55:8f:94:d1:54:9f:
                    41:36:38:10:a2:a0:88:21:b7:e5:78:cf:0c:c8:4a:
                    9b:45:5f:a0:f9:a6:c5:eb:ec:61:56:6b:98:24:a3:
                    10:65:be:a2:8d:86:25:22:66:14:57:81:cc:99:7d:
                    89:06:43:96:60:37:ba:62:72:91:51:d5:4a:93:58:
                    df:07:8d:c7:81:f6:29:ca:dd:0c:95:ad:8e:aa:38:
                    52:0b:85:ea:b6:19:bf:e3:20:0f:ac:f4:4c:a9:79:
                    1e:ee:96:1b:b0:4a:97:89:08:e8:50:2e:06:e2:b9:
                    d5:32:39:bf:a7:67:54:33:13:7f:0c:89:60:b1:80:
                    e4:31:d5:4b:04:de:59:99:d1:d6:8d:1b:53:11:0a:
                    d7:c7
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment, Certificate Sign
            X509v3 Basic Constraints: critical
                CA:TRUE
    Signature Algorithm: sha256WithRSAEncryption
         a9:8f:e8:7c:0f:24:6c:10:f8:51:e5:03:78:df:fb:ef:33:d5:
         03:3f:e2:6f:f5:16:18:ef:e9:cb:7e:60:95:c9:fa:a7:cd:e9:
         f9:56:ca:f5:05:44:bd:ee:97:06:32:18:83:8f:2f:b6:0f:a3:
         e8:b2:9e:ba:d1:63:74:98:7e:a1:51:c4:80:6e:84:cf:7d:81:
         45:3f:c8:2f:95:7b:16:8b:ea:4d:1d:88:f2:3b:b7:74:fd:99:
         72:a4:fc:c6:4d:82:5a:6f:41:bc:6b:e0:53:5d:bf:91:cf:35:
         e6:06:8c:65:ca:0d:48:b8:a7:56:cb:4d:f2:ab:79:a0:9b:8f:
         20:c0:a9:8d:41:a5:80:77:8b:ea:77:07:c4:11:cf:90:ac:ce:
         f0:2f:13:6d:4b:1c:af:6e:bf:1a:7c:b3:f8:8e:42:09:e1:d9:
         46:b0:38:e8:7e:1f:70:28:b7:06:58:d6:bc:da:6e:07:d0:a6:
         f0:77:72:1a:33:27:4d:3a:aa:7a:0a:3f:f3:13:54:d8:d5:16:
         4a:8b:0e:ec:90:79:b3:3a:4d:f4:be:e0:ef:3b:d4:bf:83:19:
         56:1c:9e:97:83:ba:be:cc:ef:34:0b:7d:4d:0b:46:b1:75:c8:
         3a:82:74:b9:dd:1e:da:35:a6:6d:1e:84:5f:58:c7:a7:5f:69:
         58:79:20:1d

{% endhighlight %}

Now, let's grab the TLS certificate, used by the web server on the master nodes. I would simply export it from your browser, after you have gone to the API Server's endpoint.  You can also use _openssl_, like this:

{% highlight bash %}
$ openssl s_client -connect 715065595AECC5FB4F2F3811FBA82A45.yl4.us-west-2.eks.amazonaws.com:443 \
  </dev/null 2>/dev/null | openssl x509 -text

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4999861380098072088 (0x4563136f4ef09e18)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jun 13 21:56:25 2018 GMT
            Not After : Jun 13 21:56:25 2019 GMT
        Subject: CN=kube-apiserver
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:ba:06:3c:5a:cf:ab:65:37:fc:bf:f2:fe:66:8b:
                    cf:c2:30:9e:2d:aa:1a:11:e2:f3:27:0c:9b:5f:72:
                    fa:ca:83:71:f9:d8:fa:dd:43:01:a5:3f:d1:7b:8c:
                    bf:58:df:df:7f:71:98:6d:81:37:9b:e3:b6:4a:36:
                    10:3e:6f:98:92:80:9e:61:ac:97:9d:0e:d0:b7:ea:
                    bf:d8:d0:94:e8:3a:54:5b:22:f6:4a:de:fb:3f:4f:
                    5d:38:c9:05:b5:e2:07:6a:7e:5e:f8:f7:1f:9c:37:
                    1f:3b:b9:c9:bf:26:ff:bf:c8:22:25:17:fe:99:28:
                    94:0e:0f:ae:3f:5e:72:10:59:f5:91:33:48:72:e7:
                    15:b8:49:f2:01:30:66:37:da:19:ab:bb:70:5b:15:
                    b7:12:c7:64:57:bc:76:50:99:96:26:12:17:63:4c:
                    b1:5b:8c:0b:19:3b:37:4c:b3:69:fe:4f:59:71:33:
                    cd:c3:f6:29:fd:ba:26:ff:7b:75:80:09:ea:0d:ae:
                    ef:de:df:4d:b3:5a:e1:5b:c7:b7:99:a2:cb:38:46:
                    7f:47:e6:96:75:74:7c:fd:0b:91:31:e3:7f:1b:c9:
                    ed:42:42:fc:dc:85:3e:f0:4e:d2:f0:78:14:76:5a:
                    c2:f4:c7:4a:19:62:e9:93:af:b1:61:f0:1c:a5:d4:
                    f8:9f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication
            X509v3 Subject Alternative Name: 
                DNS:ip-10-0-54-116.us-west-2.compute.internal, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, DNS:715065595aecc5fb4f2f3811fba82a45.yl4.us-west-2.eks.amazonaws.com, IP Address:10.100.0.1, IP Address:192.168.139.208
    Signature Algorithm: sha256WithRSAEncryption
         4a:cc:1f:ce:d0:67:cf:bd:d7:c9:b4:92:81:00:0c:fc:3b:bb:
         31:d7:dd:c7:ef:77:59:36:0b:93:0b:2d:ac:ce:42:2b:67:2f:
         48:c7:2a:20:a0:fc:87:97:bd:05:3a:e6:50:74:07:5b:5d:be:
         94:48:4e:dc:8f:99:41:b6:ec:f5:81:79:8a:32:fb:77:ac:ee:
         84:97:7a:91:0b:8b:ff:d5:98:51:03:fa:0f:83:5b:c6:c5:b8:
         2a:35:3d:72:33:6f:f2:30:55:18:e6:b1:c6:cb:49:0f:9e:77:
         cb:31:de:6c:17:74:9d:45:b7:5d:6c:39:b9:f2:44:c5:2e:8f:
         1f:61:a1:33:6e:d4:1d:4c:2d:05:6d:fb:b5:ae:bf:e6:f8:de:
         45:37:94:a8:c4:a3:b0:5f:53:eb:16:bb:ab:ba:1f:fd:21:08:
         c0:ff:bc:91:2c:fc:bf:8d:c5:40:d7:ff:63:b1:e6:f2:72:11:
         2d:3e:82:f8:de:f7:d5:98:5b:10:41:4e:7d:af:64:47:6f:98:
         b0:74:ab:72:f0:57:b1:42:22:d1:94:fb:bd:34:bd:0d:43:40:
         a6:87:fb:9b:2a:f1:96:ea:15:90:31:2d:33:7c:b8:05:ca:41:
         c9:86:a7:bd:28:4c:d3:e1:79:7f:b9:18:79:12:3b:d6:63:f2:
         79:ea:53:91
-----BEGIN CERTIFICATE-----
MIIDwDCCAqigAwIBAgIIRWMTb07wnhgwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
AxMKa3ViZXJuZXRlczAeFw0xODA2MTMyMTU2MjVaFw0xOTA2MTMyMTU2MjVaMBkx
FzAVBgNVBAMTDmt1YmUtYXBpc2VydmVyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
MIIBCgKCAQEAugY8Ws+rZTf8v/L+ZovPwjCeLaoaEeLzJwybX3L6yoNx+dj63UMB
pT/Re4y/WN/ff3GYbYE3m+O2SjYQPm+YkoCeYayXnQ7Qt+q/2NCU6DpUWyL2St77
P09dOMkFteIHan5e+PcfnDcfO7nJvyb/v8giJRf+mSiUDg+uP15yEFn1kTNIcucV
uEnyATBmN9oZq7twWxW3EsdkV7x2UJmWJhIXY0yxW4wLGTs3TLNp/k9ZcTPNw/Yp
/bom/3t1gAnqDa7v3t9Ns1rhW8e3maLLOEZ/R+aWdXR8/QuRMeN/G8ntQkL83IU+
8E7S8HgUdlrC9MdKGWLpk6+xYfAcpdT4nwIDAQABo4IBDjCCAQowDgYDVR0PAQH/
BAQDAgWgMBMGA1UdJQQMMAoGCCsGAQUFBwMBMIHiBgNVHREEgdowgdeCKWlwLTEw
LTAtNTQtMTE2LnVzLXdlc3QtMi5jb21wdXRlLmludGVybmFsggprdWJlcm5ldGVz
ghJrdWJlcm5ldGVzLmRlZmF1bHSCFmt1YmVybmV0ZXMuZGVmYXVsdC5zdmOCJGt1
YmVybmV0ZXMuZGVmYXVsdC5zdmMuY2x1c3Rlci5sb2NhbIJANzE1MDY1NTk1YWVj
YzVmYjRmMmYzODExZmJhODJhNDUueWw0LnVzLXdlc3QtMi5la3MuYW1hem9uYXdz
LmNvbYcECmQAAYcEwKiL0DANBgkqhkiG9w0BAQsFAAOCAQEASswfztBnz73XybSS
gQAM/Du7Mdfdx+93WTYLkwstrM5CK2cvSMcqIKD8h5e9BTrmUHQHW12+lEhO3I+Z
Qbbs9YF5ijL7d6zuhJd6kQuL/9WYUQP6D4NbxsW4KjU9cjNv8jBVGOaxxstJD553
yzHebBd0nUW3XWw5ufJExS6PH2GhM27UHUwtBW37ta6/5vjeRTeUqMSjsF9T6xa7
q7of/SEIwP+8kSz8v43FQNf/Y7Hm8nIRLT6C+N731ZhbEEFOfa9kR2+YsHSrcvBX
sUIi0ZT7vTS9DUNApof7myrxluoVkDEtM3y4BcpByYanvShM0+F5f7kYeRI71mPy
eepTkQ==
-----END CERTIFICATE-----
{% endhighlight %}

#### Verify the certificate chain using _openssl_
Grab the certificate at the end of the output, and place it into a file.
I called mine _kube-apiserver.pem_.  Now, let's ask openssl to verify the TLS cert used by the API server has been signed by the CA given to us by the AWS management plane.

{% highlight bash %}
$ openssl verify -verbose -CAfile ca_cert.pem kube-apiserver.pem 
kube-apiserver.pem: OK
{% endhighlight %}

If you don't see the *OK*, then you have a security issue.  Otherwise, you are fine.

As mentioned, all the Kubernetes tooling does this for you automatically.  If you examine your _kubectl_ config file, you will notice that the CA certificate at the very top of the file. _kubectl_ checks the certificate chain for you ever time. Thank you _kubectl_!!! 

### Summary
By understanding the Certificate Authority data key, given to us by the AWS management plane, we now have a better understanding of how TLS security works for self-signed hosts, or EKS in particular.

Now, some of you may be concerned that I just publically shared my CA certificate.  While I do not recommend this as a best practice, my cluster is entirely safe.  *Why* you ask? All AWS EKS clusters use *IAM* credentials along with standard Kubernetes RBAC controls.  This feature, using external authorization plugins, is still a BETA feature in Kubernetes version 1.11, but has been fully adopted by AWS EKS.  I will talk about it more in my next blog post.

___
#### **References:**

[kubernetes and certificates](https://jvns.ca/blog/2017/08/05/how-kubernetes-certificates-work/)

[Kubernetes and TLS](http://www.iorchard.net/2017/01/01/k8s_tls_howto.html)

[KOPS](https://github.com/kubernetes/kops)

[Create a kubeconfig for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)

[Kubernetes Authorization](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#client-go-credential-plugins)

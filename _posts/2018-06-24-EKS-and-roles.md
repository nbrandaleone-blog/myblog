---
layout: post
title:  "Elastic Kubernetes Service and roles on AWS"
date:   2018-06-24 21:09:01 -0500
categories: aws
---
A few weeks ago AWS released EKS or their managed Elastic Kubernetes Service. What is amazing, is that it **is** Kubernetes (version 1.10).  Of course, there are some AWS subleties. The most significant are:
1. **VPC Support.** A CNI networking plug-in which allows the containers to run within the VPC CIDR address space.
2. **IAM Authentication**. All *kubectl* calls will be authenticated using an IAM user/role. This requires a separate executable to be installed on both the client and K8 masters.

In terms of the IAM roles, I found the documentation good, but scattered, so I will describe how I created an EKS cluster, using an IAM role. The advantage of this method is that *kubectl* configuration file can be shared freely among DevOps staff.  If they have access to the proper role, then they will be able to hit the Kubernetes cluster securely.  If not, the EKS masters will drop the calls.  Once authenticated, the calls will be passed on to K8 RBAC for normal authorization and processing.

{% highlight ruby %}
require 'aws-sdk-ec2'  # v3 of AWS Ruby SDK
require 'pp'

ec2 = Aws::EC2::Resource.new(region: 'us-east-1')

i = ec2.instances.reduce({}) do |m, i|
   m[i.id] = [i.state.name, i.instance_type, i.public_ip_address]
   m
end

pp(i)
#=> {"i-0571d202e78cf8fa0"=>["running", "m5.xlarge", "54.162.10.143"],
#=> "i-010d497776465a0aa"=>["running", "t2.small", "54.89.160.235"],
#=> "i-0125db430bf7092b7"=>["running", "t2.small", "34.239.126.165"],
#=> "i-0cf9dd2190de4c84a"=>["running", "p2.xlarge", "34.228.56.193"]}
{% endhighlight %}

Check out the [AWS Ruby SDK][ruby-docs] for more info on how to get the most out of the SDK.  The latest version (v3) is a very solid SDK, and I now think it is as good as Boto3, the popular Python SDK.

[ruby-docs]: https://aws.amazon.com/sdk-for-ruby/

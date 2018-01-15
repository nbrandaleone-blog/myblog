---
layout: post
title:  "Listing all instances on AWS"
date:   2018-01-14 12:09:01 -0500
categories: aws
---
It is common to ask which instances (i.e. virtual machines) are running in your AWS account. Instances are the most expensive part of almost any AWS bill. However, it can be tricky for new DevOps Engineers to find this information. Below is a simple Ruby script (the AWS gem is already installed), which prints out this information for a single region.

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

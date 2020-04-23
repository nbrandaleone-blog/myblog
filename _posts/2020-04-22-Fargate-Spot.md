---
layout: post
title:  "Run Fargate on Spot"
date:   2020-04-22 08:00:00 -0400
categories: ecs
---
![fargate logo](/images/fargate-logo.png)

## How do I run Fargate on Spot?

With all the economic slowdown in response to the COVID-19 outbreak, I thought I would do a short blog post on how to run containers as inexpensively as possible.  That is of course done using [Spot](https://aws.amazon.com/blogs/aws/aws-fargate-spot-now-generally-available/).

Spot instances are unused capacity that AWS offers at huge discounts from on-demand prices, typically in the 70-90% discount range.  Of course, this does come with a caveat - AWS will reclaim the instances if someone is willing to pay the on-demand price.  So, while I do know of many companies who run production work-loads **entirely** on Spot, I generally do not recommend such a strategy. While a huge cost-saver, it also implies you may lose your entire fleet at once. It is a wiser strategy to run some of your _base_ capacity using on-demand (or RI, now known as [Savings Plan](https://aws.amazon.com/savingsplans/)), and your _burst_ capacity on Spot.  This way you always have some compute available if the Spot 2-minute warning comes your way.

### Using the AWS CLI
I will show you how to launch Fargate tasks using the AWS CLIs.

### Show me the code!

Let's create a cluster. The cluster needs to have a *capacity provider* associated with it. One can also add capacity providers to existing clusters. There are two capacity providers already available for Fargate: FARGATE and FARGATE_SPOT.

``` sh
aws ecs create-cluster \
     --cluster-name FargateCluster \
     --capacity-providers FARGATE FARGATE_SPOT \
     --region us-west-2
```

Let's confirm that the capacity providers are properly associated with the cluster.

``` sh
$ aws ecs describe-clusters --cluster FargateCluster | jq -r '.clusters[0].capacityProviders'
[
  "FARGATE_SPOT",
  "FARGATE"
]
```

Now, let's launch a sample task.  I have already created a task definition, and a security group (which allows HTTP traffic). The same concept applies for launching an ECS service.

Notice that I am simply using one Capacity Provider (FARGATE_SPOT), and it has a weight of 1.  However, it is possible to utilize both Capacity Providers, and to mix the weights along with an optional *base* argument. This allow for great flexibility when it comes to mixing FARGATE and FARGATE_SPOT providers which allows for safety if our Spot fleet is reclaimed.

The syntax for the `capacityProvider` argument is: `capacityProvider=string,weight=integer,base=integer ...`.

``` sh
aws ecs run-task \
     --capacity-provider-strategy capacityProvider=FARGATE_SPOT,weight=1 \
     --cluster FargateCluster \
     --task-definition sample-fargate:8 \
     --count 1 \
     --network-configuration "awsvpcConfiguration={subnets=[subnet-b1f0b3d5,subnet-6b982f40],securityGroups=[sg-0eb694fc9c09f50bf],assignPublicIp=ENABLED}" \
     --region us-west-2
```

Now, lets check that the application is running properly. Let's first get the task-id, and then we can lookup the Public IP address of the task.

``` sh
$ aws ecs list-tasks --cluster FargateCluster
{
    "taskArns": [
        "arn:aws:ecs:us-west-2:991225764181:task/9db04bcd-89ec-4671-bd93-a10628a560e0"
    ]
}
```

``` sh
$ aws ecs describe-tasks --cluster FargateCluster --tasks 9db04bcd-89ec-4671-bd93-a10628a560e0 | jq -r '.tasks[0].attachments[0].details[] | select(.name=="networkInterfaceId") | .value'

eni-0cb9cee5bd332e6e1

# Track down the public IP address from the ENI.
$ aws ec2 describe-network-interfaces --network-interface-ids eni-0cb9cee5bd332e6e1 | jq -r '.NetworkInterfaces[0].Association.PublicIp'

34.222.210.104
```

Test the running container:

``` sh
$ curl 34.222.210.104

<html>
<head>
  <title>Amazon ECS Sample App</title> 
  <style>body {margin-top: 40px; background-color: #0066ff;} </style> 
</head>
  <body> 
    <div style=color:white;text-align:center> 
      <h1>Amazon ECS Sample App</h1> 
      <h2>Congratulations!</h2> 
      <hr>
      <p>Your application is now running on a container in Amazon ECS.</p>
    </div>
  </body>
</html>
```

Now, let's clean up our running task. Spot may be inexpensive - but it not free.

``` sh
aws ecs stop-task --cluster FargateCluster --task 9db04bcd-89ec-4671-bd93-a10628a560e0
```

### Running Fargate on Spot using CDK

Now, I would like to show you how to use CDK to launch Fargate tasks.
Unfortunately, it is still an open [issue](https://github.com/aws/aws-cdk/issues/5850), since Capacity Providers are not yet supported in CloudFormation.

### Using Fargate Spot with EC2 Auto Scaling Groups
While the focus of this post is on Fargate, many customers still use EC2 nodes to host their containers.  These nodes are often managed by an Auto Scaling Group. It is possilbe to use Capacity Providers with ASG.

In order to keep this post small, I will not demonstrate how to do this, but just point you to the [documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-auto-scaling.html#asg-capacity-providers). It is very similar to what we did above, except that one must create a Capacity Provider and associate it with an previously created Auto Scaling Group.

``` sh
aws ecs create-capacity-provider \
     --name CapacityProviderName \
     --auto-scaling-group-provider autoScalingGroupArn="AutoScalingGroupARN",managedScaling=\{status='ENABLED|DISABLED',targetCapacity=integer,minimumScalingStepSize=integer,maximumScalingStepSize=integer\},managedTerminationProtection="ENABLED|DISABLED" \
     --region us-east-2
```

### References

* [Capacity Providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-capacity-providers.html)
* [Using Fargate Capacity Providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-capacity-providers.html)
* [Fargate Spot Blog](https://aws.amazon.com/blogs/aws/aws-fargate-spot-now-generally-available/)
* [Auto Scaling with Capacity Providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cluster-auto-scaling.html)
* [Tutorial on ASG and Capacity Providers](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/tutorial-cluster-auto-scaling-cli.html)
* [AWS CLI docs for ECS](https://docs.aws.amazon.com/cli/latest/reference/ecs/index.html#cli-aws-ecs)

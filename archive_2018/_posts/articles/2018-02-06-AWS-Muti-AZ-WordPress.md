---
layout: post
comments: true
title: AWS WordPress with Multi-AZ and Self-Healing
excerpt: "Cloudformation Project"
categories: articles
comments: true
share: true
modified: 2017-11-12T14:11:53-04:00
---

# Deploying Multi-AZ and Self-Healing Wordpress with Backup to S3 sync

This reference architecture provides a set of YAML templates for deploying the following AWS services :
- Amazon IAM
- Amazon S3
- Amazon VPC
- Amazon NACL

## Prerequisites Notes
The Cloudformation Security Group IP address is open by default (testing purpose). You should update the Security Group Access with your own IP Address to ensure your instances security.

Before you can deploy this process, you need the following:
 - Your AWS account must have one VPC available to be created in the selected region
 - Amazon EC2 key pair

## Tested on the following Region:
 - US East (N. Virginia)

This template describes a VPC with two private and two public subnets.

![wordpress-infrastructure-overview]({{ site.url }}/assets/images/Multi-AZ_and_Self-Healing_Wordpress.png)

The repository consists of a set of nested templates that deploy the following:

 - A tiered [VPC](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html) with public and private subnets, spanning an AWS region.
 - A self-healing arc deployed across multi [Availability Zones](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) in an [Auto Scaling](https://aws.amazon.com/autoscaling/) group.
 - Two [NAT gateways](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/vpc-nat-gateway.html) to handle outbound traffic.

#### Infrastructure-as-Code

A template can be used repeatedly to create identical copies of the same stack (or to use as a foundation to start a new stack).  Templates are simple YAML- or JSON-formatted text files that can be placed under your normal source control mechanisms, stored in private or public locations such as Amazon S3, and exchanged via email. With CloudFormation, you can see exactly which AWS resources make up a stack. You retain full control and have the ability to modify any of the AWS resources created as part of a stack. 

#### Self-documenting 

Fed up with outdated documentation on your infrastructure or environments? Still keep manual documentation of IP ranges, security group rules, etc.?

With CloudFormation, your template becomes your documentation. Want to see exactly what you have deployed? Just look at your template. If you keep it in source control, then you can also look back at exactly which changes were made and by whom.

#### Intelligent updating & rollback

CloudFormation not only handles the initial deployment of your infrastructure and environments, but it can also manage the whole lifecycle, including future updates. During updates, you have fine-grained control and visibility over how changes are applied, using functionality such as [change sets](https://aws.amazon.com/blogs/aws/new-change-sets-for-aws-cloudformation/), [rolling update policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html) and [stack policies](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html).

## Template details

The templates below are included in this repository and reference architecture:

| Template | Description |
| --- | --- | 
| [master.yaml](master.yaml) | This is the master template - deploy it to CloudFormation and it includes all of the nested templates automatically. |
| [infrastructure/wp-iam.yaml](infrastructure/wp-iam.yaml) | This template deploys will create policy to allow EC2 instance full access to S3 & CloudWatch, and VPC Logs to CloudWatch. |
| [infrastructure/wp-s3bucket.yaml](infrastructure/wp-s3bucket.yaml) | This template deploys Backup Data Bucket with security data at rest and archive objects greater than 60 days, and ELB logging. |
| [infrastructure/wp-securitygroup.yaml](infrastructure/wp-securitygroup.yaml) | This template deploys a VPC with a pair of public and private subnets spread across two Availability Zones. It 

After the CloudFormation templates have been deployed, the [stack outputs](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/outputs-section-structure.html) contain a link to the site URLs.

![wordpress-stack-outputs]({{ site.url }}/assets/images/wordpress-stack-outputs.png)

![wordpress_bamboo_var1]({{ site.url }}/assets/images/wordpress_bamboo_var1.png)

![wordpress_bamboo_var2]({{ site.url }}/assets/images/wordpress_bamboo_var2.png)


## How do I...?

### Get started and deploy this into my AWS account

You can launch this CloudFormation stack in your account:

Example using AWS CLI Command :

1. First Download this code into your workstation, make your own changes and make the prerequisites updates.
 - Your AWS account must have one VPC available to be created in the selected region.
 - Create Amazon EC2 key pair
 - Install a domain in Route 53.
 - Install a certificate (in your selected region & also one in us-east-1) 

2. Next install [AWS CLI](aws.amazon.com/cli) in your workstation.

3. Upload the files in the "infrastructure" directory into to your own S3 bucket.
 - Eg. aws s3 cp --recursive infrastructure/ s3://cf-templates-19sg5y0d6d084-ap-southeast-1/

4. You can run the master.yaml file from your workstation.


```
Example Setup:

Stage1 (~ 10 - 15 minutes)
===========================
To create a environment :
aws cloudformation create-stack \
--stack-name <env> \
--capabilities=CAPABILITY_IAM \
--template-body file:////path_to_template//cloudformation-vpc//master.yaml

To update a environment :
aws cloudformation update-stack \
--stack-name <env> \
--capabilities=CAPABILITY_IAM \
--template-body file:////path_to_template//cloudformation-vpc//master.yaml

To delete a environment :
aws cloudformation delete-stack --stack-name <env>

<env> - Note :stack-name that can be used are (devVPC, stagingVPC, prodVPC)

Example :
aws cloudformation create-stack \
--stack-name devVPC \
--capabilities=CAPABILITY_IAM \
--parameters ParameterKey=PMOWNIP,ParameterValue=0.0.0.0/0 \
--parameters ParameterKey=PMPrivateSubnet1CIDR,ParameterValue=10.0.3.0/24 \
--parameters ParameterKey=PMPrivateSubnet2CIDR,ParameterValue=10.0.4.0/24 \
--parameters ParameterKey=PMPublicSubnet1CIDR,ParameterValue=10.0.1.0/24 \
--parameters ParameterKey=PMPublicSubnet2CIDR,ParameterValue=10.0.2.0/24 \
--parameters ParameterKey=PMTemplateURL,ParameterValue=https://s3.amazonaws.com/cf-templates-hyv79l0oex7c-us-east-1/cloudformation-vpc/infrastructure \
--parameters ParameterKey=PMVpcCIDR,ParameterValue=10.0.0.0/16 \
--template-body file:////path_to_template//cloudformation-vpc//master.yaml
	
```

### Deploy multiple environments (e.g., dev, staging, production)

Deploy another CloudFormation stack from the same set of templates to create a new environment. The stack name provided when deploying the stack is prefixed to all taggable resources (e.g., EC2 instances, VPCs, etc.) so you can distinguish the different environment resources in the AWS Management Console. 

### Change the VPC or subnet IP ranges

This set of templates deploys the following network design:

| Item | CIDR Range | Usable IPs | Description |
| --- | --- | --- | --- |
| VPC | 10.0.0.0/16 | 65,536 | The whole range used for the VPC and all subnets |
| Public Subnet 1 | 10.0.1.0/24 | 251 | The public subnet in the first Availability Zone |
| Public Subnet 2 | 10.0.2.0/24 | 251 | The public subnet in the second Availability Zone |
| Private Subnet 1 | 10.0.3.0/24 | 251 | The private subnet in the first Availability Zone |
| Private Subnet 2 | 10.0.4.0/24 | 251 | The private subnet in the second Availability Zone |

You can adjust the following section of the parameter template:

### Add a new item to this list

If you found yourself wishing this set of frequently asked questions had an answer for a particular problem, please [submit a pull request](https://help.github.com/articles/creating-a-pull-request-from-a-fork/). The chances are that others will also benefit from having the answer listed here.

## Contributing

Please [create a new GitHub issue](https://github.com/thinegan/cloudformation-jira/issues/new) for any feature requests, bugs, or documentation improvements. 

Where possible, please also [submit a pull request](https://help.github.com/articles/creating-a-pull-request-from-a-fork/) for the change. 

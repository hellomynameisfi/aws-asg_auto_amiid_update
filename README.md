# aws_imgbldr_asg_from_amiTags

## Table of contents
* [General info](#general-info)
* [Setup](#setup)

## General info
These are step-by-step instructions on how to automatically update the ami-ID inside an Auto Scalig Group with an Image Builder generated image ami-ID. The goal is to have one function responsible for updating all desired ASG's instead of a function separately for each ASG. This is done by reading a set amiTag value instead of a environment variable from within Lambda function.

This solution is building on "Sample EC2 Auto Scaling groups Instance Refresh solution" by aws-samples (https://github.com/aws-samples/ec2-auto-scaling-instance-refresh-sample).
	
## Setup
To run this project, install it locally using npm:




### Create a Lambda function
asdasdas

```
code
```

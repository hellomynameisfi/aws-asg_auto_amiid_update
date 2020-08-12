# EC2 Auto Scaling groups Instance Refresh (amiTag) solution

## Table of contents
* [General info](#general-info)
* [Step 1: Create an IAM role for EC2 Image Builder](#step-1-create-a-iam-role-for-ec2-image-builder)
* [Step 2: Create an IAM role for Lambda](#step-2-create-a-iam-role-for-lambda)
* [Step 3: Create a SNS topic](#step-3-create-an-sns-topic)
* [Step 4: Create your pipeline and run it]()


## General info
These are step-by-step instructions on how to automatically update the ami-ID inside an Auto Scalig Group with an EC2 Image Builder generated image ami-ID. The goal is to have one function responsible for updating all desired ASG's instead of a function for each ASG separately (and because I was unsuccesfull with the original solution provided by aws-samples). I use a amiTag set in the EC2 Image Builder pipeline instead of a environment variable from within Lambda function.

This solution is building on "Sample EC2 Auto Scaling groups Instance Refresh solution" by aws-samples (https://github.com/aws-samples/ec2-auto-scaling-instance-refresh-sample).


### Step 1: Create a IAM role for EC2 Image Builder
First of all we need to create a IAM role for our Image Builder pipeline. For that navigate to IAM in youe ARS Console (https://console.aws.amazon.com/iam/) and under Access management on the left panel select "Roles" (https://console.aws.amazon.com/iam/home?/roles#/) click "Create role" button.

After being taken to the next screen select:
* AWS service
* EC2

... and click the "Next: Permissions" button at the bottom of the page.

Here you will need to add the following policies: 
* AmazonSSMManagedInstanceCore 
* EC2InstanceProfileForImageBuilder 

Now click "Next: Tags" button on the bottom of the page (add Tags if needed by your setup) and again click "Next: Review" on the bottom of the page. Here you will have to provide a name for your role. Let's call it: *imagebuilder_pipeline_role*. Click the "Create role" button at the bottom of the screen.

Search for the role you just created (https://console.aws.amazon.com/iam/home?/roles#/) and click on it. Now you can add additional permissions that are still needed. To do that click on "Add inline policy" on the right of the "Permissions" tab. On the next page select JSON and paste the following:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::gitlab-ansible"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::gitlab-ansible/*"
        }
    ]
}
```
Now click the "Review policy" button at the bottom of the page. Give the policy a name: *image_builder_s3-readonly-access*, and create the policy by clicking "Create policy" button at the bottom of the page.

You have now succesfully created a role for your EC2 Image Builder pipeline. Let's continue.

### Step 2: Create a IAM role for Lambda
We also need to create the role for the Lambda function to be able to update the Auto Scaling Group. The process is quite similiar to what we did in when we created a role for EC2 Image Builder.

Navigate to IAM in youe ARS Console (https://console.aws.amazon.com/iam/) and under Access management on the left panel select "Roles" (https://console.aws.amazon.com/iam/home?/roles#/) click "Create role" button.

After being taken to the next screen select:
* AWS service
* Lambda

... and click the "Next: Permissions" button at the bottom of the page.

Here you will need to add the following policy: 
* AWSLambdaBasicExecutionRole

Now click "Next: Tags" button on the bottom of the page (add Tags if needed by your setup) and again click "Next: Review" on the bottom of the page. Here you will have to provide a name for your role. Let's call it: *lambda_function_refresh_ami*. Click the "Create role" button at the bottom of the screen.

Search for the role you just created (https://console.aws.amazon.com/iam/home?/roles#/) and click on it. Now you can add additional permissions that are still needed. To do that click on "Add inline policy" on the right of the "Permissions" tab. On the next page select JSON and paste the following:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "logs:PutLogEvents",
                "autoscaling:StartInstanceRefresh",
                "ec2:CreateLaunchTemplateVersion"
            ],
            "Resource": [
                "arn:aws:logs:*:*:log-group:*:log-stream:*",
                "arn:aws:autoscaling:*:*:autoScalingGroup:*:autoScalingGroupName/*",
                "arn:aws:ec2:*:*:launch-template/*"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingNotificationTypes",
                "autoscaling:DescribeLifecycleHookTypes",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeTerminationPolicyTypes",
                "autoscaling:DescribeScalingProcessTypes",
                "ec2:DescribeLaunchTemplates",
                "autoscaling:DescribePolicies",
                "autoscaling:DescribeTags",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeMetricCollectionTypes",
                "autoscaling:DescribeLoadBalancers",
                "autoscaling:DescribeLifecycleHooks",
                "autoscaling:DescribeAdjustmentTypes",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAccountLimits",
                "autoscaling:DescribeScheduledActions",
                "autoscaling:DescribeLoadBalancerTargetGroups",
                "autoscaling:DescribeNotificationConfigurations",
                "autoscaling:DescribeInstanceRefreshes"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": "logs:CreateLogStream",
            "Resource": "arn:aws:logs:*:*:log-group:*"
        }
    ]
}
```

Now click the "Review policy" button at the bottom of the page. Give the policy a name: *lambda_asg-refresh_ami*, and create the policy by clicking "Create policy" button at the bottom of the page.

You have now succesfully created a role for your Lambda function. Let's continue.

### Step 3: Create an SNS topic
In your AWS console navigate to SNS and choose "Topics" from the left panel (https://console.aws.amazon.com/sns/v3/home?/topics). Click the "Create topic" button. Give it a name: *image_builder-to-lambda*, and click "Create topic" button at the bottom of the page.

YYou have now succesfully created an SNS topic for your setup. Don't worry about a subscription, we'll do it later. Let's continue.

### Step 4: Create your pipeline and run it
Navigate to EC2 Image builder (https://console.aws.amazon.com/imagebuilder/). Here you can create your pipeline that will be creating your "golden image" for your Auto Scaling Group.




### Create a Lambda function
Now we need to create a Lambda function that will do all the job for us. To do that navigate to Lambda in your AWS Console (https://console.aws.amazon.com/lambda/) and click the "Create function" button. 


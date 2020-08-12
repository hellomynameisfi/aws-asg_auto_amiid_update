# EC2 Auto Scaling groups Instance Refresh (amiTag) solution

## Table of contents
* [General info](#general-info)
* [Step 1: Create an IAM role for EC2 Image Builder](#step-1-create-a-iam-role-for-ec2-image-builder)
* [Step 2: Create an IAM role for Lambda](#step-2-create-a-iam-role-for-lambda)
* [Step 3: Create a SNS topic](#step-3-create-an-sns-topic)
* [Step 4: Create your pipeline and run it](#step-4-create-your-pipeline-and-run-it)
* [Step 5: Create a Lambda function](#step-5-create-a-lambda-function)
* [Step 6: Create a Launch template](#step-6-create-a-launch-template)


## General info
These are step-by-step instructions on how to automatically update the AMI ID inside an Auto Scalig Group with an EC2 Image Builder generated image AMI ID. The goal is to have one function responsible for updating all desired ASG's instead of a function for each ASG separately (and because I was unsuccesfull with the original solution provided by aws-samples). I use a amiTag set in the EC2 Image Builder pipeline instead of a environment variable from within Lambda function.

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
We also need to create the role for the Lambda function to be able to update the Auto Scaling group. The process is quite similiar to what we did in when we created a role for EC2 Image Builder.

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

You have now succesfully created an SNS topic for your setup. Don't worry about a subscription, we'll do it later. Let's continue.

### Step 4: Create your pipeline and run it
Now we need to set a pipeline and run it for the first time to get a new AMI ID. To do that navigate to EC2 Image builder (https://console.aws.amazon.com/imagebuilder/). Here you can create your pipeline that will be creating your "golden image" for your Auto Scaling group.

Click the "Create image pipeline" button. On the next page select a Linux/Windows distribution you want to use and under "Select image" choose "Select managed images". Here you can choose the distribution version you wish to use. After selecting the image tick the "Always build latest version." box (may be ommited if your setup does not allow this).

Now we have to add at least one component to our image pipeline. Let's quickly create one if you don't have one already. For that click "Create build component". On the page you were taken to fill out the following:
* Component name = "Update OS"
* Component version = 1.0.0
* Content = remove whatever was there and replace it with:
```
name: 'Update OS'
description: 'This component updates the OS'
schemaVersion: 1.0
phases:
  - name: build
    steps:
      - name: UpdateOS
        action: UpdateOS
```
... and click "Create component" button at the bottom of the page.

Now close this window and return to the one where we were configuring the pipeline in (if you closed it for some reason, you will have to restart this step and instead of creating a new component we are going to select the one we have created a moment ago). You can add your component by clicking "Browse build components", selecting "Owned by me" instead of "Amazon owned", and selecting the desired component (in our case *Update OS ver. 1.0.0*).

If you have a valid test you can use, please add it in the "Test" area. For this example we will not be doing this as this step is optional. Click "Next" button on the bottom of the page.

Here you have to provide:
* Name = a name for your pipeline (we will use *pipeline-example*)
* IAM role = select the EC2 role you have selected in [Step 1](#step-1-create-a-iam-role-for-ec2-image-builder) (in our case it's *imagebuilder_pipeline_role*)
* Build schedule = select how you want your pipeline to be initiated
* Instance type = add all the instance types you wish to use
* SNS topic = select the topic we have created in [Step 3](#step-3-create-an-sns-topic) (in our case it's *image_builder-to-lambda*)
* VPC, subnet and security groups = select a VPC, subnet and security group you want to use
* Troubleshooting settings = select a key pair you want to use to log in to the machine later if needed

... and click "Next" button on the bottom of the page.

On the next page, under "Output AMI" create a tag with the key *ASGname* with a value equal to the Auto Scaling group name you will be updating (in our case we will use *server_asg_to_update*. Click "Review" on the bottom of the page and then click "Create pipeline" on the bottom of the next page.

Now select the pipeline you have just created, click the "Actions" button and select "Run pipeline".

You have now succesfully created a pipeline, which is now building the first AMI to be used in your Auto Scaling group.

### Step 5: Create a Lambda function
Now we need to create a Lambda function that will do all the job for us. To do that navigate to Lambda in your AWS Console (https://console.aws.amazon.com/lambda/) and click the "Create function" button. 

### Step 6: Create a Launch template
Get yourself some coffee and wait till the pipeline image is fully created and marked as "Available" in EC2 Image Builder (this may take some time). 

### Step 7: Create Auto Scaling group from Launch Template



Copy contents of [lambda.py](https://raw.githubusercontent.com/hellomynameisfi/aws-asg_auto_amiid_update/master/lambda.py?token=AQKBCKD2VERRINL55L7FQMK7GPPDO)

### Step 8: Run your pipeline to invoke a Lambda function and update AMI ID in Auto Scaling group


### Step 9: Profit!
It's much more clicking then the base example provided by aws-samples, but at the same time you can more easily tailor this solution to your needs and already created resources.

###### IF xou BTC: 1bcalksdjasdjfsdfum0s9mf8ßs9df8s0ßf98sß0.9

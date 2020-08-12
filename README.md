# EC2 Auto Scaling groups Instance Refresh (amiTag) solution

## Table of contents
* [General info](#general-info)
* [Setup](#setup)

## General info
These are step-by-step instructions on how to automatically update the ami-ID inside an Auto Scalig Group with an Image Builder generated image ami-ID. The goal is to have one function responsible for updating all desired ASG's instead of a function for each ASG separately (and because I was unsuccesfull with the original solution provided by aws-samples). This is done by reading a set amiTag value instead of a environment variable from within Lambda function.

This solution is building on "Sample EC2 Auto Scaling groups Instance Refresh solution" by aws-samples (https://github.com/aws-samples/ec2-auto-scaling-instance-refresh-sample).
	
## Setup
To run this project, install it locally using npm:

### Create a IAM Role for Image Builder
First of all we need to create a IAM role for our Image Builder pipeline. For that navigate to IAM in youe ARS Console (https://console.aws.amazon.com/iam/) and under Access management on the left panel select "Roles" (https://console.aws.amazon.com/iam/home?/roles#/) click "Create role" button.

After being taken to the next screen select:
* AWS service
* EC2

... and click the "Next: Permissions" button at the bottom of the page.

Here you will need to add the following Policies: 
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
Now click the "Reviev policy" button at the bottom of the page. Give the policy a name: *image_builder_s3-readonly-access*, and create the policy by clicking "Create policy" button at the bottom of the page.

You have now succesfully created a role for your Image Builder pipeline. Let's continue.

### Create your pipeline
asdasdasdasdsadasdasdd

### Create a IAM Role for Lambda
We also need to create the role for the Lambda function to be able to update the Auto Scaling Group. 

lambda_refresh_ami

### Create an SNS topic
asdasdasdasd

### Create a Lambda function
Now we need to create a Lambda function that will do all the job for us. To do that navigate to Lambda in your AWS Console (https://console.aws.amazon.com/lambda/) and click the "Create function" button. 


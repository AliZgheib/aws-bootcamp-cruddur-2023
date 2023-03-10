# Week 0 — Billing and Architecture

## Required Homeworks/Tasks

### Install & Verify AWS CLI 

#### Install AWS CLI

We can install the AWS CLI by updating our **.gitpod.yml** file to include the installation task:
 
```
tasks:
  - name: aws-cli
    env:
      AWS_CLI_AUTO_PROMPT: on-partial
    init: |
      cd /workspace
      curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
      unzip awscliv2.zip
      sudo ./aws/install
      cd $THEIA_WORKSPACE_ROOT
```
#### Verify AWS CLI on Gitpod

We can run ```aws --version``` on the gitpod environment to ensure that **AWS CLI** was installed successfully!

![AWS CLI on Gitpod](assets/week0/aws-cli.PNG)

### AWS Credentials

#### Create IAM user & Generate AWS Credentials

- Go to [IAM Users Console](https://us-east-1.console.aws.amazon.com/iamv2/home?region=us-east-1#/users) 
- Create a new user
- `Enable console access` for the user
- Create a new `Admin` Group and apply `AdministratorAccess`
- Create the user and go find and click into the user
- Click on `Security Credentials` and `Create Access Key`
- Choose AWS CLI Access
- Download the CSV with the credentials

#### Validate AWS Credentials on Gitpod

First of all, We launch our Gitpod workspace.

We will set these credentials for the current bash terminal
```
export AWS_ACCESS_KEY_ID=""
export AWS_SECRET_ACCESS_KEY=""
export AWS_DEFAULT_REGION=us-east-1
```

We'll tell Gitpod to remember these credentials if we relaunch our workspaces
```
gp env AWS_ACCESS_KEY_ID=""
gp env AWS_SECRET_ACCESS_KEY=""
gp env AWS_DEFAULT_REGION=us-east-1
```

We can confirm that these credentials are working by running the CLI command below:

```aws sts get-caller-identity```

Finally we receive a successful response

![AWS Credentials Response](assets/week0/aws-credentials.PNG)

### Enable Billing 

We need to turn on Billing Alerts to recieve alerts...

- In your Root Account go to the [Billing Page](https://console.aws.amazon.com/billing/)
- Under `Billing Preferences` Choose `Receive Billing Alerts`
- Save Preferences

### Creating a Billing Alarm

#### Create SNS Topic

- We need an SNS topic before we create an alarm.
- The SNS topic is what will delivery us an alert when we get overbilled
- [aws sns create-topic](https://docs.aws.amazon.com/cli/latest/reference/sns/create-topic.html)

We'll create a SNS Topic
```sh
aws sns create-topic --name azgheib-billing-alarm
```
which will return a TopicARN

We'll create a subscription supply the TopicARN and our Email
```sh
aws sns subscribe \
    --topic-arn arn:aws:sns:us-east-1:425617850831:azgheib-billing-alarm \
    --protocol email \
    --notification-endpoint zgheibali@gmail.com
```

Check your email and confirm the subscription

#### Create Alarm

- [aws cloudwatch put-metric-alarm](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-alarm.html)
- [Create an Alarm via AWS CLI](https://aws.amazon.com/premiumsupport/knowledge-center/cloudwatch-estimatedcharges-alarm/)
- We need to update the configuration json script with the TopicARN we generated earlier
- We are just a json file because --metrics is is required for expressions and so its easier to us a JSON file.

```sh
aws cloudwatch put-metric-alarm --cli-input-json file://aws/json/alarm_config.json
```
#### Confirm that the billing alarm was created on the AWS console

![AWS Billing Alarm](assets/week0/aws-billing-alarm.PNG)

### Create an AWS Budget

[aws budgets create-budget](https://docs.aws.amazon.com/cli/latest/reference/budgets/create-budget.html)

```sh
aws budgets create-budget \
    --account-id 425617850831 \
    --budget file://aws/json/budget.json \
    --notifications-with-subscribers file://aws/json/budget-notifications-with-subscribers.json
```

#### Confirm that the budget was created on the AWS console

![AWS Budget](assets/week0/aws-budget.PNG)

### Conceptual/Napkin Diagram

![Conceptual/Napkin Diagram](assets/week0/napkin-diagram.PNG)
[Link to the Diagram](https://lucid.app/lucidchart/36cfe8cb-9099-4b2c-910a-d0892eae2125/edit?viewport_loc=-483%2C309%2C2560%2C944%2C0_0&invitationId=inv_6d3228d3-0017-4035-868c-2b077b49fbab)

### Logical Diagram

- We use Route 53 as a DNS service to map our domain to our CloudFront distribution
- We use Cloudfront to cache static assets at the edge ( using AWS Edge Locations ) and forward dynamic requests to the Load Balancer
- We use AWS shield with CloudFront to protect our VPC and services ( attacks blocked at the Edge Locations) - this service can protect us from DDOS attacks
- We use use AWS WAF to protect our VPC and services from more sophisticated attacks and block IP addresses at the Edge Location
- We use ALB to load balance the traffic between the app and the api
- We use AWS cognito to authenticate the users at the ALB level
- We use DynamoDB and RDS as databases + Go Momento for serverless caching
- We use S3 + lambda services to process/resize/compress avatars/assets uploaded to our S3 bucket - the final result is also uploaded to S3

![Logical Diagram](assets/week0//conceptual-diagram.PNG)
[Link to the Diagram](https://lucid.app/lucidchart/2992a026-e6c0-4e5a-a6b6-bb33c988b55b/edit?viewport_loc=-899%2C575%2C2559%2C944%2C0_0&invitationId=inv_a6f6f910-6295-4aee-a171-f31b452c1462)

## Homework Challenges

### Delete Root credentials & Setup MFA

- I deleted root account credentials 
- I completed MFA Setup for the root account using Auth
- I created an IAM user + access credentials
- I completed MFA setup for my IAM user as well

![AWS IAM Dashboard](assets/week0//aws-mfa.PNG)

### Health Dashboard + SNS + EventBridge

[The AWS documentation](https://docs.aws.amazon.com/health/latest/ug/cloudwatch-events-health.html) provides a detailed step by step tutorial on how we can publish notifications based on AWS Health Dashboard events ( in case of services issues/downtime )

1. Create AWS Event Bridge rule and SNS topic

- on the AWS console we create a SNS topic
- We can susbcribe our ourselves to the SNS topic using our email address
- we confirm our subscription
- we create a new event bridge rule
- in the **Event Pattern** we use **AWS Health** as the source service
- we set our previously created SNS topic as target resource
- we finalize the creation of the event bridge rule

2. We confirm the creation of the Event Bridge rule

![AWS Event bridge source rule](assets/week0//aws-health-1.PNG)
![AWS Event bridge source rule](assets/week0//aws-health-2.PNG)


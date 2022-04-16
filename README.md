# AWS Lambda with Serverless Framework and Java/Maven

Serverless is a node.js based framework that makes creating, deploying, and managing serverless functions a breeze. We will use AWS as our FaaS (Function-as-a-Service) provider, although Serverless supports IBM OpenWhisk and Microsoft Azure as well.

In this tutorial, we will create and deploy a java-maven based AWS Lambda function. In Part-1 we will not modify any code, or even look at the generated code. We will focus on the deployment and the command line interface to manage lambda, provided out of the box by serverless framework.

## Pre-requisites
Here is what the setup on my Mac looks like (Sierra)

- brew (1.1.10) - you will need this if you do not have node/npm installed already.
- node (v7.6.0)
- npm (4.1.2)
- Apache Maven (3.2.5)
- Oracle JDK (1.8.0_121)
## Install Serverless

If you do not have node installed, use brew install node to install node and npm.

```sh
Manishs-MacBook-Pro:~ mpandit$ npm install serverless -g
/usr/local/bin/serverless -> /usr/local/lib/node_modules/serverless/bin/serverless
/usr/local/bin/slss -> /usr/local/lib/node_modules/serverless/bin/serverless
/usr/local/bin/sls -> /usr/local/lib/node_modules/serverless/bin/serverless


> serverless@1.7.0 postinstall /usr/local/lib/node_modules/serverless
> node ./scripts/postinstall.js
```
## Configure Serverless for AWS

Serverless needs to use AWS credentials for an IAM user. I recommend creating a user with no access, and we can add permissions as we go.

We create a programmatic-access-only user called serverless, and added it to a new group called Serverless. The group should have no permissions associated with it.

Next, copy paste the AWS credentials of this user in the command below.
```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless config credentials --provider aws --key <key> --secret <secret>

Serverless: Setting up AWS...
Serverless: Saving your AWS profile in "~/.aws/credentials"...
Serverless: Success! Your AWS access keys were stored under the "default" profile.
Serverless uses us-east-1 region by default, which can be overriden by providing the --region parameter.
```
## Create the Java project

```sh
Manishs-MacBook-Pro:~ mpandit$ cd ~/work
Manishs-MacBook-Pro:work mpandit$ mkdir serverless-java
Manishs-MacBook-Pro:work mpandit$ cd serverless-java
Manishs-MacBook-Pro:serverless-java mpandit$ serverless create --template aws-java-maven

Serverless: Generating boilerplate...
 _______                             __
|   _   .-----.----.--.--.-----.----|  .-----.-----.-----.
|   |___|  -__|   _|  |  |  -__|   _|  |  -__|__ --|__ --|
|____   |_____|__|  \___/|_____|__| |__|_____|_____|_____|
|   |   |             The Serverless Application Framework
|       |                           serverless.com, v1.7.0
 -------'

Serverless: Successfully generated boilerplate for template: "aws-java-maven"
Serverless: NOTE: Please update the "service" property in serverless.yml with your service name
```

## Set up IAM Policies

If you run serverless info, it will report an error as our user does not really have any permissions.

```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless info

  Serverless Error ---------------------------------------

     User: arn:aws:iam::123456789209:user/serverless is not
     authorized to perform: cloudformation:DescribeStacks
     on resource: arn:aws:cloudformation:us-east-1:123456789209:stack/aws-java-maven-dev/*

  Get Support --------------------------------------------
     Docs:          docs.serverless.com
     Bugs:          github.com/serverless/serverless/issues

  Your Environment Information -----------------------------
     OS:                 darwin
     Node Version:       7.6.0
     Serverless Version: 1.7.0
```

Lets add CloudFormation policy to this user. We will do this via adding the permission to the associated group.

You will notice that there is no such thing as AmazonCloudFormationFullAccess. Hence, we will need to create a custom policy.

Click the Group name on the IAM console, and click the Permissions tab. Expand Inline Policies, and click the link that allows creation of an inline policy.

In the page that opens up, select Policy Generator and proceed and make these selections -

- Effect : Allow
- AWS Service: AWS CloudFormation
- Actions - All Actions Selected
- ARN - *
- Click Add Statement, and click Next Step. This will show the policy we just created. Rename it to `${serverless-cloudformation-policy}`. Click Apply.
```sh
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1488265872000",
            "Effect": "Allow",
            "Action": [
                "cloudformation:*"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```
We will also need to give S3 access to this user, as serverless would need to upload the artifact to S3 for deployment via CloudFormation.
We now add `${AmazonS3FullAccess}` to this group, and our user will inherit it. This time we will click Attach Policy under Managed Policies instead of Inline Policies.

Another Managed Policy is for `${CloudWatch}` Logs access, so we will select `${CloudWatchLogsFullAccess}` and attach to the group.

As a part of the deployment process, serverless would need to associate the lambda with the invocation role. In order to do that, attach `${IAMFullAccess}` policy from the Managed Policies to the group as well.

Finally, we add `${AWSLambdaFullAccess}` to this group so the serverless framework can manage the services (lambda functions).

To summarize, our Serverless IAM Group should have following policies -

Managed -

- `${AmazonS3FullAccess}`
- `${CloudWatchLogsFullAccess}`
- `${IAMFullAccess}`
- `${AWSLambdaFullAccess}`

Inline - `${serverless-cloudformation-policy}` for CloudFormation.

#### One thing to keep in mind is that for functions that are triggered by an event, the event source would need access to invoke the function. This is managed via function policies, which are set up when the trigger is configured. For instance, if you want the lambda to be triggered on an S3 event on a bucket, the association between the S3 bucket and the function will need to have a function policy.

Now you’ll notice that the `${serverless info}` command returns no services.

```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless info

  Serverless Error ---------------------------------------

     Stack with id aws-java-maven-dev does not exist

  Get Support --------------------------------------------
     Docs:          docs.serverless.com
     Bugs:          github.com/serverless/serverless/issues

  Your Environment Information -----------------------------
     OS:                 darwin
     Node Version:       7.6.0
     Serverless Version: 1.7.0
```     
## Deploy the stack

Since deployment of the stack is essentially setting up our lambda function (the template has a simple function), we need to create the artifact (jar).

Since this is a maven project, we use maven to build the project.

```sh
Manishs-MacBook-Pro:serverless-java mpandit$ mvn clean install
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building hello dev
[INFO] ------------------------------------------------------------------------
...
...
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 14.709 s
[INFO] Finished at: 2017-02-27T23:26:00-08:00
[INFO] Final Memory: 23M/145M
[INFO] ------------------------------------------------------------------------
```

Now that we have the artifact in target folder (hello-dev.jar), we can go ahead deploy it.
```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless deploy
Serverless: Creating Stack...
Serverless: Checking Stack create progress...
.....
Serverless: Stack create finished...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading service .zip file to S3 (1.98 MB)...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
..................
Serverless: Stack update finished...
Service Information
service: aws-java-maven
stage: dev
region: us-east-1
api keys:
  None
endpoints:
  None
functions:
  aws-java-maven-dev-hello
```    
You can check the AWS CloudFormation section in the console to view details of the stack that has just been created.

Now that our service is deployed, it wil show up under serverless info
```sh  
Manishs-MacBook-Pro:serverless-java mpandit$ serverless info
Service Information
service: aws-java-maven
stage: dev
region: us-east-1
api keys:
  None
endpoints:
  None
functions:
  aws-java-maven-dev-hello
```  
Next, lets try to invoke it.
```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless invoke --function  hello
```  
```sh
{
    "statusCode": 200,
    "body": "{\"message\":\"Go Serverless v1.x! Your function executed successfully!\",\"input\":{}}",
    "headers": {
        "X-Powered-By": "AWS Lambda & serverless"
    },
    "isBase64Encoded": false
}
```  
The function can also be invoked with an input. The input is essentially a Map, so we can pass it a JSON string with the --data parameter.

```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless invoke --function hello --data '{"key":"value"}'
{
    "statusCode": 200,
    "body": "{\"message\":\"Go Serverless v1.x! Your function executed successfully!\",\"input\":{\"key\":\"value\"}}",
    "headers": {
        "X-Powered-By": "AWS Lambda & serverless"
    },
    "isBase64Encoded": false
}
```  
A good idea would be to explore the CloudWatch Logs and Lambda metrics for our function from the console.

The same can be done command line, like so -

```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless logs --function hello
START RequestId: f8390071-fd8d-11e6-9195-554889504121 Version: $LATEST
2017-02-28 08:14:48 <f8390071-fd8d-11e6-9195-554889504121> INFO  com.serverless.Handler:17 - received: {}
END RequestId: f8390071-fd8d-11e6-9195-554889504121
REPORT RequestId: f8390071-fd8d-11e6-9195-554889504121	Duration: 445.52 ms	Billed Duration: 500 ms 	Memory Size: 1024 MB	Max Memory Used: 57 MB
```  
We can also look at the metrics associated with our functions (The 1 error is because I had passed a malformed JSON input).
```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless metrics
Service wide metrics
February 27, 2017 1:59 AM - February 28, 2017 1:59 AM

Invocations: 7
Throttles: 0
Errors: 1
Duration (avg.): 276.55ms
```
```sh
Manishs-MacBook-Pro:serverless-java mpandit$ serverless metrics --function hello
hello
February 27, 2017 1:59 AM - February 28, 2017 1:59 AM

Invocations: 7
Throttles: 0
Errors: 1
Duration (avg.): 276.55ms
```

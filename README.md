# Serverless PHP in AWS
The term "Serverless" refers to a compute model that allows Developers to focus on writing code rather than building/running/scaling the infrastructure it's running on.
The AWS Lambda service lets users define Functions (self-contained units of code) in a High Availability managed environment that automatically scales to meet demand.

Interactive Web Applications often consist of HTML (generated by a backend script), running JavaScript (in the browser) that can call a backend API for extra functionality.
Although Lambda supports a number of languages by default, PHP is not one of them.
However, we are allowed to define custom runtimes so (with a little extra work) it's possible to execute PHP scripts.

This repository is made up of an ordered set of folders charting my progress towards a workable serverless PHP model:
0. Running PHP Hello World in Lambda
1. Configuring & Extending PHP in Lambda
2. Deploy PHP Lambda Functions with AWS CloudFormation
3. Building PHP Serverless Applications with AWS SAM
4. Building PHP Serverless Applications with Docker Images
5. Basic PHP Serverless framework with AWS SAM

# Resources

This has been a long-running (part-time) project with much research and reading, I have tried my best to track the resources I have used into the following list:

- [AWS Lambda Custom Runtime for PHP: A Practical Example](https://aws.amazon.com/blogs/apn/aws-lambda-custom-runtime-for-php-a-practical-example/)
- [Serverless PHP on AWS Lambda - Rob Allen's DevNotes](https://akrabat.com/serverless-php-on-aws-lamda/)
- [Creating Custom AWS Lambda Runtime using Lambda Layer - Parikshit Agnihotry](http://p.agnihotry.com/post/php_aws_lambda_runtime/)
- [Serverless PHP applications with Bref - Matthieu Napoli - Scotland PHP 2019](https://www.youtube.com/watch?v=oIOJulJlCb4)
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
  - [Using layers with your Lambda function](https://docs.aws.amazon.com/lambda/latest/dg/invocation-layers.html)
  - [Custom AWS Lambda runtimes](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html)
- [AWS Serverless Application Model Developer Guide](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html)
  - [Building custom runtimes](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/building-custom-runtimes.html)

# FAQs

## Lambda has been out for a while, hasn't this problem already been solved?

Yes.

If you don't want to get into the "nitty-gritty" of how everything works then you should stop reading this and visit the [Bref](https://bref.sh/) or [Laravel Vapor](https://vapor.laravel.com/) projects instead.

## How much does it cost to run a website this way?

Not much at all!

The majority of AWS Services are "Pay As You Go" and as Lambda charges on actual CPU/Memory usage, if a function sits unused for a month you have nothing to pay!

AWS offers a "Free Tier" for the first 365 days of any new account so you can get a proof of concept up and running for not much more than the price of a Domain name.

## How much will I need to know about AWS?

Although I provide code examples/cli commands, it would be beneficial to have a basic working understanding of the following AWS Services:

### Identity & Access Management
[IAM](https://aws.amazon.com/iam/) (pronounced "I am") is responsible for User/Group/Role definitions in your AWS account.
The majority of work in this repository will be performed using [AWS CLI](https://aws.amazon.com/cli/) components that depend on specific [user credentials](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys) (keys).

**note: `Just like your computer, it's very important that you don't use your default (root) account credentials for day-to-day work.**

### Cloud9

[Cloud based IDE](https://aws.amazon.com/cloud9/) that automatically integrates into your AWS account/environment making it much easier to build/manage PHP components.

### Lambda
[Lambda](https://aws.amazon.com/lambda/) is the AWS Serverless Service enabling you to execute code without having to worry about managing infrastructure.
You pay for requests and execution time, not the time your function exists.

### Amazon API Gateway
[API Gateway](https://aws.amazon.com/api-gateway/) lets you create/publish/manage/secure RESTful/WebSocket APIs.
You can define HTTP URLs that act as public endpoints for your Lambda Functions. 

### Simple Storage Service
[S3](https://aws.amazon.com/s3/) provides highly available, redundant and secure object storage.
It also provides an ability to host static web pages without needing a dedicated web server to respond to HTTP requests.

### CloudFront
[CloudFront](https://aws.amazon.com/cloudfront/) is primarily a Content Delivery Network (CDN) but can also provide TLS Encryption (HTTPS) for connections as
well as a Web Application Firewall (WAF) to help prevent malicious requests/attacks.

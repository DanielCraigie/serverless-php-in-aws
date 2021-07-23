# Running PHP Hello World in Lambda

## TL;DR
Lambda is a fully managed service providing scalable capacity for executing stand-alone Functions.
PHP isn't supported by default, however, you can build a custom PHP binary `Layer` to enable script execution.

This example creates and deploys a PHP "Hello World" Lambda Function.

## Background
In simple terms, a `Lambda Function` is a Docker Image and a `Runtime Environment` is a Container that executes code.
Runtime Environments poll the Lambda Service `Runtime API` for work.
After execution, the environments return the results to the Runtime API and are frozen (suspended) until there is more work to do.
If the environment remains frozen for long enough, it will be destroyed by the Lambda Service to free up resources.

Errors and messages sent to `stdout`/`stderr` during execution are forwarded to CloudWatch over the Lambda Service `Logging API`.

### Function Lifecycle

When the Lambda Service API receives a call to run a function:
1. `Runtime Environment` is un-frozen/created
2. environment calls `Runtime API` to retrieve the request details
3. the Functions code is executed
4. the execution result is returned to the `Runtime API`
5. `Runtime Environment` is frozen
6. after `x` seconds the frozen environment is destroyed

### Lambda Backend

The Lambda Service manages the number of running Runtime Environments based on the number of requests received.
If there aren't any frozen environments when a request is received, a new environment will be created to handle that request.
If there are more frozen environments than incoming requests, the service will destroy excess frozen environments.

The immense compute capacity provided by the Lambda Service means Functions operate in a naturally scalable architecture, accounts come with [default limits](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) that can be extended on request.
As it takes a short period of time to spin up each runtime environment there will always be a slight delay for a sub-set of Function calls over time.
It is important to bear this in mind when architecting Lambda into your Application, especially if Users might be interacting with Functions directly.

Function code/files are placed inside two main directories when a Runtime Environment is spun up:
1. ```/opt``` - contains the `bootstrap` file as well as the contents of any attached Layers (code shared between functions)
2. ```/task```- source code for the specific Function

### Custom Runtimes

Lambda Runtime Environments are governed by two things:
1. the Lambda Service that builds, freezes & destroys the environment
2. the `bootstrap` file that is automatically executed after the environment has been started

Lambda already provides runtime environments for Node.js/Python/Ruby/Java/Go/.NET leaving you to write the code to be executed.
To use something else, you first need to provide Lambda the means to execute your code.

This isn't as hard as you might imagine, looking at the list above the only thing that's in your control is the `bootstrap` file (pint 2).
The bootstrap file can be written in any language that can be executed in a default EC2 Amazon Linux 2 instance.
I use BASH bootstrap files with `curl` to interact with the HTTP APIs and gain a control wrapper around the PHP execution thread.

The bootstrap file is automatically executed once the Runtime Environment has started and should continuously poll the [Runtime API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html) for new requests to execute:
1. `/runtime/invocation/next` GET details of Next Invocation: [Request ID, Execution TTL, Function ARN, X-Ray Trace ID]
2. execute Function code with request data
3. `/runtime/invocation/AwsRequestId/response` POST execution response

If an error occurs during initialisation/runtime there are two extra Runtime API endpoints to report back to Lambda:
- `/runtime/init/error` POST initialisation error details
- `/runtime/invocation/AwsRequestId/error` POST execution error details

## Hello World example

### Step 1 - Create a Cloud9 Workspace

[Cloud9](https://aws.amazon.com/cloud9/) is browser based IDE powered by a custom EC2 instance.
The instance will automatically power down when not in use for a set period of time to save costs.

1. Navigate to the Cloud9 Dashboard in the AWS Console
2. Click the `Create environment` button
3. Name your new Environment (e.g. My PHP Lambda Builder) and click `Next step`
4. Configure the new Environment
    1. set `Environment type` to "Create a new EC2 instance for environment (direct access)" (default)
    2. the `Instance type` must have at least 2 GiB RAM & 2 vCPU to compile the PHP Binary
    3. leave the `Platform` set to "Amazon Linux 2"
    4. that should be enough, you are welcome to modify other configuration options like Tags to track your costs in the Billing Dashboard
    5. click `Next step`
5. Confirm settings and click `Create environment`

### Step 2 - Create an S3 Bucket to hold your Lambda assets
Once your Cloud9 environment has started, you should see an interactive Terminal below the welcome panel.
Execute the following command to create a new S3 Bucket to save the PHP Executable/Files.
S3 Buckets exist within a Global namespace so you will have to adjust the bucket name if someone else has already used it.

Note: it's best to set the `--region` & `LocationConstraint` to your closest [Geographic Region](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/#Region_Maps_and_Edge_Networks) and don't forget that S3 Buckets have [naming rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html). 

```aws s3api create-bucket --region eu-west-2 --acl private --bucket my-php-lambda-bucket --create-bucket-configuration LocationConstraint=eu-west-2```

### Step 3 - Compile a PHP Binary to execute scripts
1. Start by preparing the build environment
   1. ```sudo yum update -y```
   2. ```sudo yum install autoconf bison gcc gcc-c++ libxml2-devel -y```
2. Download PHP source code
   1. ```cd ~/environment```
   2. ```curl -sL https://www.php.net/distributions/php-8.0.8.tar.gz | tar -xvz```
   3. ```mkdir php-8.0.8-build```
   4. ```cd php-8.0.8```
3. Compile PHP Binary
   1. ```./buildconf --force```
   2. ```./configure --prefix=/home/ec2-user/environment/php-8.0.8-build```
   3. ```make install```
   4. ```cd ..```
   4. ```cp php-8.0.8-build/bin/php .```
   5. ```./php -v```
      1. this command should print correct PHP version
4. Save PHP Binary to S3 for later use
   1. ```aws s3 cp ./php s3://my-php-lambda-bucket/layers/bin/php/```

### Step 4 - Build PHP runtime Lambda Layer
Lambda Layers enable a single set of files to be used by multiple Functions without duplication.
When the Functions Runtime Environment is created, the contents of each attached Layer is placed in the `/opt` directory.
As we are creating a custom runtime for PHP our Layer must also include the `bootstrap` file which will be executed by the Lambda Service. 

1. ```cd ~/environment```
2. copy the contents of the `bootstrap` file (from this repository folder) into your Cloud9 home directory (using your preferred text editor)
3. Layer contents must be zipped together
   2. ```zip -r runtime.zip bootstrap php```
4. ```aws --region eu-west-2 lambda publish-layer-version --layer-name php-8-0-8 --zip-file fileb://runtime.zip```
5. Make a note of the `LayerVersionArn` returned by the command (you will need it to build the Function)

### Step 5 - Create an IAM Role for the Lambda Function
Lambda Functions must be given an IAM Role to use during execution.
AWS IAM allows you to attach Policies (permission definitions) to Roles that enable access to other resources.

1. Open [IAM](https://console.aws.amazon.com/iam/home#/home) in the AWS Console and click "Roles" in the left hand menu
2. Now click the "Create Role" button
3. The first page of the Create Role workflow asks you to define what the Role is being defined for
   1. Select "Lambda" in the "AWS service" tab 
   2. Continue to the Permissions page
4. You will now be asked to create/select policies to attach to the new Role
   1. For the moment find and click the checkbox next to the "AWSLambdaExecute" role
   2. Continue to the Tags page
5. Tags help you track the Roles usage
   1. Continue to the Review page
6. Give your new role a name (e.g MyLambdaExecutionRole)
   1. Click the "Create Role button"
7. You should be redirected back to the IAM Roles page
   1. Use the search input in the table to locate your new role
   2. Click on your new role to view the details
   3. Make a note of the "Role ARN" (you will need it to build the Function)

### Step 6 - Deploy your Hello World Function
1. Return to your Cloud9 Environment
   1. Environments will automatically switch off after a period of time as a cost saving measure, so you may need to restart it
2. ```cd ~/environment```
2. Copy the contents of the `helloWorld.php` file (from this repository folder) into your Cloud9 home directory (using your preferred text editor)
3. Function contents must be zipped together
   1. ```zip -r helloWorld.zip helloWorld.php```
4. Now put everything together to create your function
   1. ```aws lambda create-function --function-name hello-world --handler helloWorld.php --zip-file fileb://./helloWorld.zip --runtime provided --role "{Role.arn}" --region eu-west-2 --layers "{Layer Arn}"```

### Step 7 - Execute your Hello World Function
Lambda Functions can only be executed via API calls made to the Lambda Service.
In this example we will use the AWS CLI to make an `invoke` API call to trigger the function and collect its response.

1. ```cd ~/environment```
2. ```aws lambda invoke --function-name hello-world --region eu-west-2 hello-world-output.txt```
3. ```cat hello-world-output.txt```

## Resources
- [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [AWS Cloud9](https://aws.amazon.com/cloud9/)
- [AWS S3](https://aws.amazon.com/s3/)
- [AWS Lambda Custom Runtime for PHP: A Practical Example](https://aws.amazon.com/blogs/apn/aws-lambda-custom-runtime-for-php-a-practical-example/)

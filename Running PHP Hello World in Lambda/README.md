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

Note: every Runtime Environment serving a request counts towards the [total concurrency](https://docs.aws.amazon.com/lambda/latest/dg/invocation-scaling.html) for the Function in that particular Region & Account.
When the Concurrency [limit](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) is reached the Lambda service will stop creating new Runtime Environments and make new requests wait for an existing environment to complete its current request.
So although it's possible for the Lambda service to scale to meet your traffic demands, there are limitations that may need to be tuned to achieve this in busy environments/load spikes.

As it takes a short period of time to spin up each runtime environment there will always be a slight delay for a sub-set of Function calls over time.
It is important to bear this in mind when architecting Lambda into your Application, especially if Users might be interacting with Functions directly.

### Directory Structure

Function code/files are placed inside two main directories when a Runtime Environment is spun up:
1. ```/opt``` - contains the `bootstrap` file as well as the contents of any attached Layers (code shared between functions)
2. ```/task```- source code for the specific Function

### Custom Runtimes

Lambda Runtime Environments are governed by two things:
1. the Lambda Service that builds, freezes & destroys the environment
2. the `bootstrap` file that is automatically executed after the environment has been started

Lambda already provides [runtime environments](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html) for Node.js/Python/Ruby/Java/Go/.NET leaving you to write the code to be executed.
To use something else, you first need to provide Lambda the means to execute your code.

This isn't as hard as you might imagine, looking at the list above the only thing that's in your control is the `bootstrap` file (pint 2).
The bootstrap file can be written in any language that can be executed in a default EC2 Amazon Linux 2 instance.
In this example I use BASH bootstrap files with `curl` to interact with the HTTP APIs and gain a control wrapper around the PHP execution thread.

The bootstrap file is automatically executed once the Runtime Environment has started and should continuously poll the [Runtime API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html) for new requests to execute:
1. [HTTP GET] `/runtime/invocation/next` - returns details of Next Invocation: [Request ID, Execution TTL, Function ARN, X-Ray Trace ID]
2. execute Function code with request data
3. [HTTP POST] `/runtime/invocation/AwsRequestId/response` - execution response

If an error occurs during initialisation/runtime there are two extra Runtime API endpoints to report back to Lambda:
- [HTTP POST] `/runtime/init/error` - initialisation error details
- [HTTP POST] `/runtime/invocation/AwsRequestId/error` - execution error details

## Hello World example

This example will guide you though the process of creating a custom PHP Runtime and creating a Lambda function that will execute a PHP script.

You will need to have the [AWS CLI](https://aws.amazon.com/cli/) installed and configured on your computer to complete some of the steps.

### Step 1 - Create an S3 Bucket to hold your Lambda assets
Once your Cloud9 environment has started, you should see an interactive Terminal below the welcome panel.
Execute the following command to create a new S3 Bucket to save the PHP Executable/Files.
S3 Buckets exist within a Global namespace so you will have to adjust the bucket name if someone else has already used it.

Note: it's best to set the `--region` & `LocationConstraint` to your closest [Geographic Region](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/#Region_Maps_and_Edge_Networks) and don't forget that S3 Buckets have [naming rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html). 

```aws s3api create-bucket --region eu-west-2 --acl private --bucket my-php-lambda-bucket --create-bucket-configuration LocationConstraint=eu-west-2```

Don't forget to secure your new bucket from public access (isn't set by default when creating buckets from the CLI)

```aws s3api put-public-access-block --bucket my-php-lambda-bucket --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true```

### Step 2 - Compile a PHP Binary to execute scripts

#### Create a Cloud9 Workspace

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

#### Execute the following commands in your Cloud9 Workspace
1. Start by preparing the build environment
   1. ```sudo yum update -y```
   2. ```sudo yum install autoconf bison gcc gcc-c++ libxml2-devel -y```
2. Download PHP source code
   1. ```cd ~/environment```
   2. ```curl -sL https://www.php.net/distributions/php-8.1.10.tar.gz | tar -xvz```
   3. ```mkdir php-8.1.10-build```
   4. ```cd php-8.1.10```
3. Compile PHP Binary
   1. ```./buildconf --force```
   2. ```./configure --prefix=/home/ec2-user/environment/php-8.1.10-build```
   3. ```make install```
   4. ```cd ..```
   4. ```cp php-8.1.10-build/bin/php .```
   5. ```./php -v```
      1. this command should print correct PHP version
4. Save PHP Binary to S3 for later use
   1. ```aws s3 cp ./php s3://my-php-lambda-bucket/layers/bin/php/```

You can now delete the Cloud9 Workspace.

### Step 3 - Build PHP runtime Lambda Layer
Lambda Layers enable a single set of files to be used by multiple Functions without duplication.
When the Functions Runtime Environment is created, the contents of each attached Layer is placed in the `/opt` directory.
As we are creating a custom runtime for PHP our Layer must also include the `bootstrap` file which will be executed by the Lambda Service. 

1. navigate to the "0 - Running PHP Hello World in Lambda" directory in your command line tool
2. copy the PHP binary down from your S3 bucket
   1. `aws s3 cp s3://my-php-lambda-bucket/layers/bin/php/php .`
3. Layer contents must be zipped together
   2. ```zip -r runtime.zip bootstrap php```
4. `layerName="php-8-1-10"`
5. ```aws --region eu-west-2 lambda publish-layer-version --layer-name $layerName --zip-file fileb://runtime.zip```

### Step 4 - Create an IAM Role for the Lambda Function
Lambda Functions must be given an IAM Role to use during execution.
AWS IAM allows you to attach Policies (permission definitions) to Roles that enable access to other resources.

1. `region="eu-west-2"`
2. `roleName="MyLambdaExecutionRole"`
3. `aws iam create-role --role-name $roleName --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'`
4. `aws iam attach-role-policy --role-name $roleName --policy-arn $(aws iam list-policies --scope AWS --query "Policies[?PolicyName=='AWSLambdaExecute'].Arn" --output text)`

### Step 5 - Deploy your Hello World Function
1. navigate to the "0 - Running PHP Hello World in Lambda" directory in your command line tool
2. Function contents must be zipped together
   1. ```zip -r helloWorld.zip helloWorld.php```
4. Now put everything together to create your function
   1. `roleName="MyLambdaExecutionRole"`
   3. `layerName="php-8-1-10"`
   2. ```aws lambda create-function --function-name hello-world --handler helloWorld.php --zip-file fileb://./helloWorld.zip --runtime provided --role $(aws iam list-roles --query "Roles[?RoleName == '$roleName'].Arn" --output text) --region eu-west-2 --layers $(aws lambda list-layer-versions --layer-name $layerName --query "LayerVersions[0].LayerVersionArn" --output text)```

### Step 6 - Execute your Hello World Function
Lambda Functions can only be executed via API calls made to the Lambda Service.
In this example we will use the AWS CLI to make an `invoke` API call to trigger the function and collect its response.

1. ```aws lambda invoke --function-name hello-world --region eu-west-2 hello-world-output.txt```
2. ```cat hello-world-output.txt```

#### Update 09/2022
Since I last worked on this repository the supporting libraries in Lambda Runtime Environments have changed to the extent that they no longer meet the requirements for the PHP binary to execute.
When I attempted to invoke this function the runtime debug contained this error message: `/opt/php: /lib64/libc.so.6: version 'GLIBC_2.25' not found (required by /opt/php)`

This is another reason for skipping this workflow and moving straight to the [PHP HelloWorld AWS Lambda Container](https://github.com/DanielCraigie/hello-world-php-aws-lambda-container) repository.

## Resources
- [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [AWS Cloud9](https://aws.amazon.com/cloud9/)
- [AWS S3](https://aws.amazon.com/s3/)
- [AWS Lambda Custom Runtime for PHP: A Practical Example](https://aws.amazon.com/blogs/apn/aws-lambda-custom-runtime-for-php-a-practical-example/)

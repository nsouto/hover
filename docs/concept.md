# Archticture Concept

Hover was built with enterprise applications in mind and is optimized for security and flexibility. Each stage (dev, staging, production, sandbox, ...) of your application has its own manifest file in which you can configure the different AWS resources used by the stage. You can deploy each stage with a different AWS profile. Each profile can hold credentials to a different user, with granular permissions to manage this stage only, or even a different AWS account.

For each stage, Hover creates several Lambda functions, EventBridge rules and CloudWatch log groups. It also creates an HTTP API, CloudFront distribution, ECR Repository and S3 bucket.

On every deployment, the application code is packaged into a docker image along with the runtime. That image gets uploaded to an AWS ECR repository. Your application asset files are uploaded to an S3 bucket separately.

Let's learn how Hover works with AWS to deploy and run the different parts of your application.

![Hover Architecture Concept](images/arch-concept.png)

## AWS Lambda

Hover creates several AWS Lambda functions for each stage:

- One function to handle HTTP requests
- One function to execute CLI commands and run the Laravel scheduler
- Multiple functions to process jobs from different SQS queues

Inside the stage manifest file, you can configure the memory, timeout and concurrency of each of the functions. You can also configure the different aspects of each of the queue functions:

```yaml
http:
  memory: 512
  timeout: 30
  concurrency: 100
cli:
  memory: 512
  timeout: 900
  concurrency: 10
queue:
  default:
    memory: 512
    timeout: 120
    concurrency: 5
    tries: 3
    backoff: "5, 10"
    queues:
      - default
      - notifications
  priority:
    memory: 512
    timeout: 300
    concurrency: 10
    tries: 5
    backoff: "1"
    queues:
      - priority
```

In this example, Hover will create 4 functions. Two functions for handling HTTP requests and CLI invocations, and two functions for handling queued jobs.

The HTTP function will be invoked by ApiGateway each time a request comes to your application. While the CLI Lambda will be invoked by an EventBridge scheduling rule that runs every minute. It can also be invoked manually through the `hover command run` command.

For the queue functions, they will be invoked by the Lambda-SQS integration. This integration polls the specified queues in each Lambda waiting for a job to become available. Once a job is picked up by the integration, the function will be invoked with the job payload.

## APIGateway & Handling HTTP Requests

Hover configures an APIGateway HTTP API and integrates it with the HTTP function of the stage. Each time the API receives a request, the HTTP lambda is invoked by the gateway.

For each stage, AWS generates a unique domain, which can be used to test the application. It looks like this:

```
https://d3876dg38.execute-api.eu-west-1.amazonaws.com
```

To access the app using your own domain, Hover utilizes APIGateway custom domain mappings.

## EventBridge Rules

Hover utilizes Amazon EvenBridge rules to invoke the CLI function every minute with the `php artisan schedule:run` command. If any of your application's scheduled jobs is due, the command will execute them for you.

Another use case for EventBridge rules is a warmer ping that invokes the HTTP function every 5 minutes. The runtime that was added to your app when you ran `hover build` will handle this ping and invoke a specified number of HTTP function containers concurrently. Each of these containers will have PHP-FPM up and running waiting for HTTP requests to process.

## SQS Queues

For every queue specified in the stage manifest file, Hover configures an integration to poll the queue for jobs and invoke the corresponding function with the job payload.

## CloudWatch Log Groups

For each of the Lambda functions created by Hover, a CloudWatch log group is created. All invocation records and logs will be stored in the log group. You can inspect any of these log groups in your AWS console.

## Assets S3 Bucket & CloudFront Distribution

Hover creates an S3 bucket and a CloudFront distribution for each stage. Your application asset files are uploaded to the S3 bucket on every deployment and are served using the CloudFront CDN. Which reduces the latency when serving assets to users.

## Elastic Container Registery

An ECR repository is created for every stage. The repository will be used to publish the docker image tagged by Hover for every deployment. Hover then configures the different lambda functions to run the newly deployed tag.

## CloudFormation

Hover manages most of the AWS resources needed to deploy and run a stage using CloudFormation. A stack is created for every stage and is updated on every deployment.

## Key Management Service

KMS is used to store an encryption key for every stage. This key is generated by Hover and is never exposed. It is used internally to encrypt and decrypt the stage secrets when using the `hover secret encrypt` and `hover secret decrypt` commands. It is also used by the runtime to decrypt the secrets and populate environment variables each time a lambda container starts.
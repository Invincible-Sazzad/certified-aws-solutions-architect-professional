# AWS Lambda

- Lambda is a Function-as-a-Service (FaaS) product. We provide specialized short running a focused code for Lambda and it will take care running it and billing us for only what we consume
- Every Lambda function uses a supported runtime, example: Python 3.8
- Every Lambda function is loaded into an executed ina runtime environment
- When we create a function we define the resources the function will use. We define the memory directly and CPU usage allocation indirectly (based on the amount of memory)
- We are only billed for the duration the function is running based on the number of invocations and the resources specified
- Lambda is a key part of serverless architectures in AWS
- Lambda has support for the following runtimes:
    - Python
    - Ruby
    - Go
    - Java
    - C#
    - Custom using Lambda layers
- Lambda functions are stateless, meaning there is no data left over after an invocation
- When creating a Lambda function we define the memory. The memory can be between 128MB and 3GB in 64MB steps(historically, nowadays there is support for up to 10GB)
- We do not directly define the vCPU allocated to each function, it will automatically scale with the memory: 1792MB of memory gives 1 vCPU
- The runtime env. has a 512MB storage available as `/tmp`
- Lambda function can run up to 15 minutes, after this a timeout will occur

## Lambda Networking

- Lambda functions can have 2 types of networking modes:
    - Public (default):
        - Lambda can access public AWS services such as SQS, DynamoDB, etc. and also internet based services
        - Lambda has network connectivity to public services running on the internet
        - Offers the best performance for Lambda, no customer specific networking is required
        - With public networking mode Lambda function wont be able to access resources in a VPC unless the resources do have public IPs and security controls allow external access
    - VPC Networking:
        - Lambda functions will run inside a VPC, so they will access everything in a VPC, assuming NACLs and SGs allow access
        - They wont be able to access services outside of the VPC, unless networking configuration exists in the VPC to allow external access
        - At the creation of the function, a certain ENI might be created for accessing the VPC. The initial setup would take up to 90 seconds

## Lambda Security

- There are 2 key parts of the security model
    - Lambda Functions will assume an execution role in order to access other AWS resources
    - Resources policies: similar to resource policies for S3. Allows external accounts to invoke a Lambda functions, or certain services to use Lambda functions. Resources polices can be modified using the CLI (not with the console)

## Lambda Logging

- Lambda uses CloudWatch Logs and X-Ray
- Logs from Lambda executions are stored in CloudWatch Logs
- Details about Lambda metrics are stored in CloudWatch Metrics
- Lambda can be integrated with X-Ray for distributed tracing
- For Lambda to be able to log we need to give an execution log to it

## Lambda Invocations

- There are 3 ways Lambda functions can be invoked:
    - **Synchronous invocation**:
        - Command line or API directly invoking the function
        - The CLI or API will wait until the function returns
        - API Gateway will also invoke lambdas synchronously, use case for many serverless applications
        - Any errors or retries have to handled in the client side
    - **Asynchronous invocation**:
        - Used typically when AWS services invoke the function (example: S3 events)
        - The service will not wait for the response (fire and forget)
        - Lambda is responsible for any failure. Reprocessing will happen between 0 and 2 times
        - The function should be idempotent in order to be rerun
        - Lambda can be configured to send events to a DLQ in case of the processing did not succeed after the number of retries
        - Destination: events processed by Lambdas can be delivered to destinations like SQS, SNS, other Lambda, EventBride). Success and failure events can be sent to different destinations
    - **Event Source mapping**:
        - Typically used on streams or queues which don't generate events (Kinesis, DynamoDB streams, SQS)
        - Event Source mappers polls these streams and retrieves batches. These batches can be broken in pieces and sent to multiple Lambda invocations for processing
        - We can not have a partially successful batch, either everything works or nothing works
- In case of event processing in async invocation, in order to process the event we don't explicitly need rights to read from the sender
- In case of event source mapping the event source mapper is reading from the source. The event source mapping uses permissions from the Lambda execution role to access the source service
- Even if the function does not read data directly from the stream, the execution role needs read rights in order to handle the event batch
- Any batch that consistently fails to be processed, it can be sent to an SQS queue or SNS topic for further processing

## Lambda Versions

- We can define different versions for given functions
- A version of a function is the code + configuration of the function
- When we publish a version, it becomes immutable, it no longer can be changed. It event gets its own ARN (Amazon Resource Name)
- `$Latest` points to the latest version of Lambda version (it is not immutable)
- We can also define aliases (DEV, STAGE, PROD) which point to a version of the function. Aliases can be changed to point to other versions

## Lambda Start-up Times

- Lambda code runs inside of a runtime environment (execution context)
- At first invocation this execution context needs to be created and this will take time
- This process is known as cold start and it can take 100ms or more
- If the function is invoked again without too much of a gap, it might use the same execution context. This is called warm start
- One function invocation runs in an execution environment at a time. If multiple parallel instances are needed, the contexts will require cold starts
- Provisioned concurrency: we provision execution context in advance for Lambda invocations
- The improve performance we can use the `/tmp` folder to download data to it. If another invocations uses the same execution context, it will be able to access the previously downloaded data
- We can create database connections outside of the Lambda handler. These will also be available for other invocations afterwards

## Lambda Handler

- Lambda function executions have lifecycles
- The function code runs inside an execution environment
- Lifecycle phases:
    - INIT: creates on unfreezes the execution environment
    - INVOKE: runs the function handler (cold start)
    - NEXT INVOKE(s): warm start using the same environment
    - SHUTDOWN: execution environment is terminated
# Error Handling in AWS Lambdas

AWS Lambda is the cornerstone for building serverless scalable applications in serverless computing. While it simplifies deployment and code execution, robust error handling is a topic which is somewhat missed in the discussion.

Lets explore the best practices for error handling in AWS lambdas

## Types of Errors in AWS Lambda

1. **Invocation Errors:** Errors which occur when a lambda function is triggered but failes to execute properly.  
eg. Incorrrect Inputs or Insufficient Permissions

2. **Runtime Errors**: Errors which occur during execution of lambda function  
eg. Unhandled Exceptions, Syntax Errors or Error in External Dependencies

3. **Timeout Errors**: Errors which occur due to function taking longer than configured timeout  
eg. Remember when deploying a lambda we also configure the maximum amount of time in seconds that a Lambda function can run. The default value for this setting is 3 seconds, but you can adjust this in increments of 1 second up to a maximum value of 15 minutes.



## Best Practices for Error Handling


1. **Dead Letter Queues (DLQs)**  
A Dead Letter Queue (DLQ) in AWS SQS is a separate queue designed to capture and store messages that a Lambda function, processing an SQS queue, fails to process successfully.

Imagine you have a Lambda function that processes messages from an SQS queue. However, due to various reasons such as unexpected data formats, bugs in the processing logic, or intermittent issues with external dependencies, some messages consistently fail to be processed by the Lambda function.

**Solution**: Configure a Dead Letter Queue for the SQS queue to capture and store messages that couldn’t be processed successfully. Use the DLQ to investigate and reprocess failed messages.

2. **Retries with Exponential Backoff**  
Exponential backoff is a technique where the time between retry attempts grows exponentially. Instead of retrying immediately, the system waits for an increasing amount of time between each retry.

Imagine  you have a Lambda function which frequently encounters intermittent failures when making calls to an external service. These failures are often transient, possibly due to network glitches or temporary unavailability of the external service.

**Solution**: Implement automatic retries with exponential backoff to mitigate transient failures. This helps prevent overwhelming downstream services during temporary issues.

3. **Logging**
Implement detailed logging using the **logger** module. Utilize CloudWatch Logs to analyze logs and identify the root cause of unexpected behavior

### Advantage of using ***logger*** vs ***print*** (in python lambdas)

```python
import os
import logging
logger = logging.getLogger()
logger.setLevel("INFO")
  
def lambda_handler(event, context):
    logger.info('## ENVIRONMENT VARIABLES')
    logger.info(os.environ['AWS_LAMBDA_LOG_GROUP_NAME'])
    logger.info(os.environ['AWS_LAMBDA_LOG_STREAM_NAME'])
    logger.info('## EVENT')
    logger.info(event)
```

The output from logger includes the log level, timestamp, and **request ID**.

```console
START RequestId: 1c8df7d3-xmpl-46da-9778-518e6eca8125 Version: $LATEST
[INFO]  2023-08-31T22:12:58.534Z    1c8df7d3-xmpl-46da-9778-518e6eca8125    ## ENVIRONMENT VARIABLES
[INFO]  2023-08-31T22:12:58.534Z    1c8df7d3-xmpl-46da-9778-518e6eca8125    /aws/lambda/my-function
[INFO]  2023-08-31T22:12:58.534Z    1c8df7d3-xmpl-46da-9778-518e6eca8125    2023/01/31/[$LATEST]1bbe51xmplb34a2788dbaa7433b0aa4d
[INFO]  2023-08-31T22:12:58.535Z    1c8df7d3-xmpl-46da-9778-518e6eca8125    ## EVENT
[INFO]  2023-08-31T22:12:58.535Z    1c8df7d3-xmpl-46da-9778-518e6eca8125    {'key': 'value'}
END RequestId: 1c8df7d3-xmpl-46da-9778-518e6eca8125
REPORT RequestId: 1c8df7d3-xmpl-46da-9778-518e6eca8125  Duration: 2.75 ms   Billed Duration: 3 ms Memory Size: 128 MB Max Memory Used: 56 MB  Init Duration: 113.51 ms
XRAY TraceId: 1-5e34a66a-474xmpl7c2534a87870b4370   SegmentId: 073cxmpl3e442861 Sampled: true
```

As you can see, the Lambda runtime environment for python includes a customized logger that is smart to take advantage of.  
It features a formatter that, by default, includes the aws_request_id in every log message. This is the critical feature that allows for an Insights query, that filters on an individual @requestId, to show the full details of one Lambda invocation.  
It is now easier to identify the problem much quicker.

4. **Custom Error Responses**  
Establish a standardized set of error codes that your API can return. This ensures consistency and makes it easier for consumers to interpret error responses


5. **Custom Metrics and Dashboards**  
Extend your monitoring capabilities by creating custom CloudWatch Metrics for Lambda functions. Build dashboards that provide a visual representation of key metrics, aiding in proactive error detection and analysis.

6. **X-Ray Tracing**  
Integrate AWS X-Ray for distributed tracing and performance analysis. By visualizing the entire execution flow of your Lambda functions, you can pinpoint bottlenecks and identify the root causes of errors more effectively.

7. **Failure Injection Testing**
Proactively simulate failures in your Lambda functions using tools like AWS Fault Injection Simulator. This allows you to validate the resilience of your application by intentionally introducing errors and observing how your system responds.


## Best Practives for Lambda

1. Use Global Clients for External Services (e.g., DynamoDB, S3, SQS)
    - Why? Creating new connections on every request increases latency and resource consumption.
    - How? Declare clients outside the lambda_handler function.

```python
import boto3

# Initialize the client globally (outside the handler)
dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    table = dynamodb.Table('your_table_name')
    response = table.get_item(Key={'id': '123'})
    return response.get('Item', {})
```

2. Minimize Deployment Package Size
    - Why? A large package increases cold start time.
    - How?
        * Use Lambda layers for shared dependencies (e.g., boto3, numpy).
        * Remove unnecessary libraries (e.g., unused parts of boto3).
        * **Use a minimal runtime like AWS Lambda’s Graviton (ARM64) instead of x86.**

3. Use CloudWatch Logging Efficiently
    - Why? Excessive logging increases costs and slows down execution.
    - How?
        - Avoid printing large payloads.
        - Use structured logs with JSON for easy querying in CloudWatch.
```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(json.dumps({"message": "Processing request", "event": event}))
```

### Comparison: Boto3 resource vs client

When using Boto3 in AWS Lambda, choosing between resource and client depends on your use case.

| Feature	| boto3.resource	| boto3.client |
|---|---|---|
| Level of abstraction |	Higher-level API (Object-Oriented)	| Lower-level API (Closer to AWS SDK) |
| Ease of use	| More Pythonic & intuitive | More control, but verbose |
| Performance	| Slightly slower (built-in abstraction)	| Faster (direct API calls) |
| When to use |	Frequent CRUD operations (e.g., DynamoDB, S3)	| Performance-critical, high-throughput scenarios |
| Supports pagination?|	Yes (e.g., .scan())	| No, requires manual pagination |
| Response structure |	Returns objects (e.g., Table, Bucket) |	Returns JSON/dict responses |
| Example services|DynamoDB, S3 | S3, Lambda, SNS, SQS |

```python
import boto3

# Global resource connection (recommended for AWS Lambda)
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('your_table_name')

def lambda_handler(event, context):
    response = table.get_item(Key={'id': '123'})
    return response.get('Item', {})
```

```python
import boto3

# Global client connection
dynamodb_client = boto3.client('dynamodb')

def lambda_handler(event, context):
    response = dynamodb_client.get_item(
        TableName='your_table_name',
        Key={'id': {'S': '123'}}
    )
    return response.get('Item', {})
```


**TL;DR:** Use resource for ease of use, and client for performance & fine control.








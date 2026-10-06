# Assignment 4 – Serverless API with Lambda, API Gateway and DynamoDB

## Overview

In this assignment, I built a serverless backend API using AWS managed services.

The main architecture was:

`Client → API Gateway → Lambda → DynamoDB`

A client sends data to an API endpoint, API Gateway receives the request, Lambda processes the data, and DynamoDB stores it.

I also implemented several bonus features including:

- A `GET /students` endpoint
- API keys and usage plans
- AWS WAF rate limiting

<!-- IMAGE: Final architecture diagram -->

---

## 1. Creating the DynamoDB Table

I created a DynamoDB table called:

`students`

The partition key was:

`id`

The `id` attribute acts as the unique identifier for each item stored in the table.

Each submission stored in DynamoDB contains:

- A unique ID
- A timestamp
- The submitted payload

<!-- IMAGE: DynamoDB students table configuration -->

---

## 2. Creating the POST Lambda Function

I created a Lambda function called:

`student-submit`

The function was written in Python.

Its purpose was to:

1. Receive the request body
2. Convert the incoming JSON into Python data
3. Generate a unique UUID
4. Generate a timestamp
5. Store the resulting item in DynamoDB
6. Return a JSON response

<!-- IMAGE: student-submit Lambda creation -->

<!-- IMAGE: Lambda POST code -->

The function used the AWS SDK for Python, `boto3`, to communicate with DynamoDB.

For example:

```python
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("students")

```
The item was then written to DynamoDB using:

```python
table.put_item(Item=item)
```

---

## 3. Applying Least-Privilege IAM Permissions

The Lambda execution role already had permission to write logs to CloudWatch.

I then added a separate DynamoDB permission.

The POST Lambda only needed permission to perform:

`dynamodb:PutItem`

against the specific `students` table.

<!-- IMAGE: Lambda execution role -->

<!-- IMAGE: DynamoDB PutItem IAM policy -->

This follows the principle of least privilege because the Lambda function only receives the permissions required to perform its task.

It does not receive unnecessary permissions such as deleting the table or accessing unrelated AWS resources.

---

## 4. Error Handling and CloudWatch Logging

The Lambda function used `try` and `except` error handling.

If the request was processed successfully, Lambda returned a successful HTTP response.

If an error occurred, the function caught the error, logged it, and returned a `500` response.

CloudWatch logging allowed me to check:

- Whether the function executed
- Execution duration
- Errors
- Request information
- Where a failure occurred

---

## 5. Testing Lambda Before API Gateway

Before connecting API Gateway, I tested the Lambda function by itself.

This allowed me to verify the Lambda-to-DynamoDB connection independently.

<!-- IMAGE: Successful Lambda test -->

The Lambda test returned:

`201 Created`

I then checked:

`DynamoDB → Tables → students → Explore table items`

and confirmed that the item had been successfully stored.

<!-- IMAGE: DynamoDB record created by Lambda -->

The table contained:

- A generated UUID
- The request payload
- A timestamp

This confirmed that:

`Lambda → DynamoDB`

was working correctly.

---

## 6. Creating the REST API

I then created a REST API using Amazon API Gateway.

API Gateway acts as the public entry point for the backend.

It allows external clients such as `curl`, applications, or a frontend website to invoke the Lambda function through standard HTTP requests.

<!-- IMAGE: REST API configuration -->

---

## 7. Creating POST /submit

I created the API resource:

`/submit`

and added a:

`POST`

method.

The POST method used Lambda proxy integration and invoked the `student-submit` Lambda function.

<!-- IMAGE: POST submit integration -->

The request flow became:

```text
POST /submit
      |
      v
API Gateway
      |
      v
Lambda
      |
      v
DynamoDB
```

---

## 8. Lambda Proxy Integration

I used Lambda proxy integration.

With proxy integration, API Gateway passes the request through to Lambda in an event structure.

Lambda then handles the request and constructs the HTTP response.

This keeps the API configuration simpler because API Gateway does not need to manually transform the request before passing it to Lambda.

---

## 9. Configuring CORS

I enabled Cross-Origin Resource Sharing (CORS).

CORS controls whether a browser-based application from one origin is allowed to access a resource from another origin.

An origin is made up of:

`protocol + domain + port`

<!-- IMAGE: API Gateway CORS configuration -->

Because I was using Lambda proxy integration, the Lambda response also included the CORS header:

```python
"Access-Control-Allow-Origin": "*"
```

For this lab, `*` allowed requests from any origin.

In a production environment, a specific frontend origin would normally be used instead.

---

## 10. Deploying the API

I deployed the REST API to a stage called:

`prod`

A stage represents a deployed environment of an API.

<!-- IMAGE: API deployment to prod -->

The deployed API could then be accessed using its API Gateway invoke URL.

---

## 11. Testing POST /submit

I tested the API using `curl`.

The request submitted JSON data to the API.

<!-- IMAGE: Successful POST curl test -->

The successful flow was:

```text
curl
  ↓
API Gateway
  ↓
POST /submit
  ↓
Lambda
  ↓
DynamoDB
  ↓
Response returned to curl
```

I then checked DynamoDB and confirmed that the new record had been stored successfully.

<!-- IMAGE: DynamoDB records after API POST -->

I also checked the Lambda CloudWatch logs and confirmed that the invocation had been logged.

<!-- IMAGE: Lambda CloudWatch logs -->

---

# Bonus Features

## 12. GET /students Endpoint

I added a second endpoint:

`GET /students`

This endpoint allows stored student records to be retrieved from DynamoDB.

I created a separate Lambda function for this endpoint rather than reusing the POST Lambda.

This kept the permissions separate and followed least privilege.

The GET flow was:

```text
GET /students
      ↓
API Gateway
      ↓
student-list Lambda
      ↓
DynamoDB Scan
      ↓
Records returned
```

---

## 13. GET Lambda IAM Permission

The GET Lambda only needed permission to read the DynamoDB table.

I added an inline IAM policy allowing:

`dynamodb:Scan`

on the specific `students` table.

<!-- IMAGE: DynamoDB Scan IAM policy -->

This means the GET Lambda can read records but cannot write, delete or modify them.

---

## 14. Reading Records from DynamoDB

The GET Lambda used:

```python
response = table.scan()
students = response.get("Items", [])
```

The `scan()` operation retrieves the items stored in the table.

<!-- IMAGE: GET Lambda code -->

I tested the function and confirmed that it returned the existing student records.

<!-- IMAGE: Successful GET Lambda test -->

---

## 15. Adding GET /students to API Gateway

I created a new API Gateway resource:

`/students`

and added a `GET` method.

The GET method used Lambda proxy integration and invoked the new read-only Lambda function.

I also enabled CORS.

<!-- IMAGE: GET students API Gateway resource -->

I redeployed the API to the existing:

`prod`

stage.

---

## 16. Testing GET /students

I tested the endpoint using `curl`.

<!-- IMAGE: Successful GET students curl test -->

The response returned the student records stored in DynamoDB.

This confirmed the full read flow:

```text
Client → API Gateway → Lambda → DynamoDB → Client
```

---

## 17. API Keys and Usage Plans

I then added API keys and a usage plan.

An API key allows API Gateway to identify a client making requests.

A usage plan controls how much that client is allowed to use the API.

The flow became:

```text
Client + API Key
       ↓
API Gateway
       ↓
Lambda
       ↓
DynamoDB
```

<!-- IMAGE: API key configuration -->

I created:

- An API key
- A usage plan

I associated the usage plan with the `prod` stage and attached the API key to the usage plan.

<!-- IMAGE: Usage plan prod stage association -->

---

## 18. Requiring API Keys

I configured both API methods to require an API key:

`GET /students`

and:

`POST /submit`

<!-- IMAGE: API key required method configuration -->

After changing the methods, I redeployed the API to `prod`.

---

## 19. Testing API Key Protection

I first sent a request without an API key.

The API rejected the request.

<!-- IMAGE: API request without key rejected -->

I then sent the same request with a valid API key using the:

`x-api-key`

header.

The request succeeded.

<!-- IMAGE: GET request with API key successful -->

I also tested the POST endpoint with the API key and confirmed that it worked.

<!-- IMAGE: POST request with API key successful -->

This demonstrated that API Gateway was enforcing the API key requirement before Lambda was invoked.

---

## 20. AWS WAF Rate Limiting

I also configured AWS WAF to protect the API from excessive traffic.

AWS WAF sits in front of the API and can inspect or block incoming requests.

The request flow became:

```text
Client
   ↓
AWS WAF
   ↓
API Gateway
   ↓
Lambda
   ↓
DynamoDB
```

---

## 21. Usage Plans vs AWS WAF

I learned that usage plans and WAF serve different purposes.

A usage plan controls API consumption for clients using API keys.

AWS WAF provides security filtering and can block traffic based on rules such as excessive requests from a particular IP address.

---

## 22. Creating the WAF Web ACL

I created a WAF protection pack / Web ACL and associated it with the API Gateway `prod` stage.

<!-- IMAGE: API Gateway prod stage associated with WAF -->

I chose to build my own rule rather than use a larger managed rule package.

I then selected a:

`Rate-based rule`

<!-- IMAGE: WAF rate-based rule selection -->

---

## 23. Configuring the Rate Limit

I configured the WAF rule with:

- Action: Block
- Rate limit: 10 requests
- Evaluation window: 1 minute
- Request aggregation: Source IP address

<!-- IMAGE: WAF rate-limit configuration -->

This means WAF monitors traffic by source IP and can block a source that exceeds the configured request rate.

---

## 24. Testing the WAF Rule

I created a shell script to repeatedly send requests to the API.

<!-- IMAGE: WAF test script -->

The script repeatedly called:

`GET /students`

and printed the HTTP status code.

Initially, the API returned:

`200`

After the WAF rate limit was exceeded, requests began returning:

`403`

<!-- IMAGE: WAF 403 terminal result -->

This confirmed that AWS WAF was successfully blocking requests after the configured rate threshold had been exceeded.

<!-- IMAGE: WAF request metrics -->

After testing, I removed the Web ACL so it would not continue generating unnecessary costs.

---

## What I Learned

This assignment helped me understand how multiple AWS serverless services work together.

I developed a better understanding of:

- DynamoDB
- Partition keys
- AWS Lambda
- Lambda execution roles
- Least-privilege IAM policies
- Boto3
- UUID generation
- API Gateway REST APIs
- POST and GET HTTP methods
- Lambda proxy integration
- CORS
- API stages
- CloudWatch logs
- API keys
- Usage plans
- AWS WAF
- Rate-based security rules
- Serverless request flows

I also learned the importance of testing each layer separately.

I first verified Lambda and DynamoDB before adding API Gateway, which made troubleshooting easier because I could confirm that the backend worked independently before adding another service.

---

## Testing and Validation

I tested the project at several stages.

I first tested the POST Lambda directly and confirmed that it successfully wrote an item to DynamoDB.

I then tested `POST /submit` through API Gateway and confirmed that the record was stored.

I added and tested `GET /students` and successfully retrieved the stored records.

I then tested API key enforcement by sending requests both with and without a key.

Finally, I tested AWS WAF rate limiting and confirmed that repeated requests were eventually blocked with HTTP `403`.

These tests confirmed that the serverless API, access controls, and rate-limiting protections were functioning successfully.

---
title: AWS APi Gateway - Cloudwatch Logs integration
date: 2019-08-27T21:21:12+01:00                   
tags: [aws]                   

---

Expose your AWS CloudWatch logs over HTTPS with a web service - **serverless**!

BeeKube infrastructure is multi-cloud: we run services on Amazon AWS, Google Cloud and DigitalOcean. One of the biggest challenges is to have a centralized log platform, not only
for our internal services, but also for our customers.

We chosen [AWS CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) as the backbone of our logging service for a few benefits:
* integration with AWS Lambda and API Gateway (it's called *lock-in*, I guess...);
* support for [JSON filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html);
* support for [alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringPolicyExamples.html);
* built-in query engine - Log Insight;

In this post I'll deep dive in the integration of CloudWatch Logs with API Gateway

# How to expose cloudwatch logs via HTTPS   
API Gateway has integrations with different AWS services; unfortunately, their blog and their documentation describe integrations with S3, AWS Lambda and DynamodDB, CloudWatch is somehow left behind...

## Know your API...
The first step to integrate an AWS service with API Gateway is to know its API - the [official documentation](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_GetLogEvents.html) describes how to send a request to CloudWatch to get the logs.
To get the logs, we need to send a POST request to CloudWatch: 
```
POST / HTTP/1.1
Host: logs.<region>.<domain>
X-Amz-Date: <DATE>
Authorization: AWS4-HMAC-SHA256 Credential=<Credential>, SignedHeaders=content-type;date;host;user-agent;x-amz-date;x-amz-target;x-amzn-requestid, Signature=<Signature>
User-Agent: <UserAgentString>
Accept: application/json
Content-Type: application/x-amz-json-1.1
Content-Length: <PayloadSizeBytes>
Connection: Keep-Alive
X-Amz-Target: Logs_20140328.GetLogEvents
{
  "logGroupName": "my-log-group",
  "logStreamName": "my-log-stream"
}
```

API Gateway handles most of the headers - but not all of them.  


## Create an IAM role for API Gateway
API Gateway needs a role to call CloudWatch Logs on our behalf. Create an IAM Role selecting API Gateway as service. Then, attach an inline policy - this is our policy
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:GetLogEvents"
            ],
            "Resource": "arn:aws:logs:eu-central-1:123456789:log-group:*:*:*"
        }
    ]
}
```

You can add constraints about the log group name or log stream names. 

## Setup API Gateway

Create a new resource with a GET method. The integration will be of type `AWS`, configured as below
{% include blog/figure.html img="api_gateway_integration1.png" alt="AWS Api Gateway Integration Configuration" border=true %}

Remarkable points:
* AWS Service and Region are set to `CloudWatch Logs` - API Gateway use these informations for the cloudwatch endpoint
* The http method is `post`, as defined in the API (actually, almost all AWS webservices use POST requests)
* Action: `GertLogEvents`

This is not enough - if you test the method, you'll get
```
{
  "Output": {
    "__type": "com.amazon.coral.service#UnknownOperationException",
    "message": null
  },
  "Version": "1.0"
}
```

As described in the API docs, we have to send some HTTP Headers and a body to the CloudWatch Logs api:
* `X-Amz-Target`
* `Content-Type`

We can add these headers and the body in the method integration
{% include blog/figure.html img="api_gateway_headers.png" alt="AWS Api Gateway Integration Headers" caption="How to configure Headers in the API Gateway Integration" border=true %}

Finally we can set the body
{% include blog/figure.html img="api_gateway_body.png" alt="AWS Api Gateway Integration Headers" caption="How to configure Headers in the API Gateway Integration" border=true %}

In a forthcoming post we'll see how to configure API Gateway to map internal parameters to integration request params!

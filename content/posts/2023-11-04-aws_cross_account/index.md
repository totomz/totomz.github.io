---
title: "AWS: Iam role cross account Lambda/S3 with Terraform"  
date: 2023-11-04T07:58:00+01:00
tags: 
    - aws
---

# HowTo: AWS IAM cross-account role to write in an S3 bucket in a different Account

Context: the aws Lambda in account `lambda_account` must gain access to an S3 bucket in the `s3_account`.

How does it work?
- The Lambda has a role in its account;
- The lambda code use assume the role in `s3_account`;
- The lambda call the S3 api using the assumed role

## 1. The roles in `lambda_Account` 
This is the regular execution role for the lambda. This role nee permission to do `sts:AssumeRole` on the role in the other account
```terraform
resource "aws_iam_role" "lambda_inbound_email" {
  provider = aws.lambda_account  # <-- this role is in the lambda_account
  name = "lambda_role"
  assume_role_policy = jsonencode({
    Version   = "2012-10-17",
    Statement = [
      {
        Action    = "sts:AssumeRole",
        Effect    = "Allow",
        Principal = {
          Service = "lambda.amazonaws.com",
        }
      }
    ]
  })
  
  inline_policy {
    name = "assume-role"
    policy = jsonencode({
      Version = "2012-10-17",
      Statement = [
        {
          Effect = "Allow",
          Action = "sts:AssumeRole",
          Resource = "arn:aws:iam::${s3_account.account_id}:role/s3-account-role"  # <-- allow sts:AssumeRole on the role in the other account
        },
      ],
    })
  }
}

resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  provider = aws.my-ideas
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
  role       = aws_iam_role.lambda_inbound_email.name
}
```

## 2. Create the role in the other account (s3_account)
```terraform
resource "aws_iam_role" "inbound_role" {
  provider = aws.s3_account
  name     = "s3-account-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          AWS = aws_iam_role.lambda_inbound_email.arn  # <-- set the lambda_account role as the principal - allow the lambda_account to assume this role 
        },
      },
    ],
  })
  
  # regular policy for this role
  inline_policy {
    name = "write-videorequest"
    policy = jsonencode({
      Version = "2012-10-17",
      Statement = [
        {
          Action = [
            "s3:PutObject",
          ],
          Effect = "Allow",
          Resource = "${aws_s3_bucket.blabla.arn}/*",
        },
      ]
    })
  }
}
```

## 3. Minor updates to the lambda code
Lambda runs with the role we assigned to it. To call the API in a different acocunt, we need to assume the new role

```go
// Create a new AWS session
sess, err := session.NewSession()
if err != nil {
    return fmt.Errorf("failed to create session, %v", err)
}
// Assume the role in the different account
creds := stscreds.NewCredentials(sess, "arn:aws:iam::<s3_account>:role/s3-account-role")

// Create a new S3 service with the assumed role credentials
svc := s3.New(sess, &aws.Config{Credentials: creds})

// Create an upload input parameters
upParams := &s3.PutObjectInput{ Bucket: &bucket, Key:    &key, Body:   bytes.NewReader(body)}
_, err = svc.PutObject(upParams)
```


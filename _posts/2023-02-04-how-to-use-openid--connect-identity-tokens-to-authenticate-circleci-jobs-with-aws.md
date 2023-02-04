---
layout: post
title: "How to use OpenID Connect identity tokens to authenticate CircleCI jobs with AWS"
tags: ["aws", "circleci", "openid"]
---

## Why use OpenID Connect in CI/CD?

* We do not have to store long-lived credentials as secrets in our CI/CD tools.
* We do not have to rotate credentials since they are no longer static.
* We have more granular control over how workflows can use credentials.
* We follow best practices in terms of authentication and authorization.

## Overview

<blockquote class="prompt-info">
<a href="https://github.com/adamkielar/circleci-oidc-aws" target="_blank">Here is a link to GitHub repo with all files for reference</a>.
</blockquote>

## Prerequisites

* IAM user with access to AWS CLI
* AWS CLI
* Terraform (Optionally)
* A CircleCI project

## Create resources using AWS CLI

### Create an IAM OIDC identity provider

1. Create an example JSON file with information that we need to fill in. 
```bash
aws iam create-open-id-connect-provider --generate-cli-skeleton > circleci-provider.json
```
```json
{
    "Url": "https://oidc.circleci.com/org/ORGANIZATION_ID",
    "ClientIDList": [
        "ORGANIZATION_ID"
    ],
    "ThumbprintList": [
        "WE WILL GET IT"
    ],
    "Tags": [
        {
            "Key": "Name",
            "Value": "circleci-oidc"
        }
    ]
}
```
2. Retrieve ORGANIZATION_ID from CircleCI.
`ORGANIZATION_ID` is a UUID identifying the current job’s project’s organization. You can find CircleCI organization id by navigating to **Organization Settings > Overview** on the https://app.circleci.com/
3. Obtaining the thumbprint for an OpenID Connect Identity Provider.
* Update url with your `ORGANIZATION_ID` and open it in a browser.
```plantext
https://oidc.circleci.com/org/ORGANIZATION_ID/.well-known/openid-configuration
```
* Find `jwks_uri` in a response and copy server url without `https://` . In our case it is `oidc.circleci.com`
* User openssl to obtain the certificate of the top intermediate CA in the certificate authority chain.
Update `URL` with value obtain in last step.
```bash
openssl s_client -servername URL -showcerts -connect URL:443 > certificate.crt
```
* Get thumbprint
```bash
openssl x509 -in certificate.crt -fingerprint -sha1 -noout
```
* Remove `:` from result and update circleci-provider.json
4. Create identity provider. In response we will get OIDC provider ARN.
```bash
aws iam create-open-id-connect-provider --cli-input-json file://circleci-provider.json
```

![OIDC Provider](/assets/post7/provider.png)

### Create a role for OIDC
1. Create a JSON file for IAM role.
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
          "Federated": "ARN from last step" 
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
          "StringLike": {
              "oidc.circleci.com/org/ORGANIZATION_ID:sub": "org/ORGANIZATION_ID/project/PROJECT_ID/user/*"
          }
      }
    }
  ]
}
```
2. Retrieve `PROJECT_ID` and optionally `USER_ID`.
`PROJECT_ID` and `USER_ID` are UUIDs that identify the CircleCI project and the user that run the job. We can find PROJECT_ID in **Project Settings > Overview** and USER_ID in **User Settings > Account Integration**
3. Update `circleci-iam-role.json` file and create role.
```bash
aws iam create-role --role-name circleci-oidc --assume-role-policy-document file://circleci-iam-role.json
```
4. Attache built in `AmazonS3ReadOnlyAccess` policy for testing.
```
aws iam attach-role-policy --role-name circleci-oidc --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

## Create resources using Terraform

1. Create `main.tf` file.
```hcl
resource "aws_iam_openid_connect_provider" "this" {
  url             = format("https://%s", var.oidc_url)
  client_id_list  = [var.oidc_client_id]
  thumbprint_list = [var.oidc_thumbprint]
}

resource "aws_iam_role" "this" {
  name = var.role_name
  assume_role_policy = data.aws_iam_policy_document.this.json

  tags = var.default_tags
}

data "aws_iam_policy_document" "this" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type = "Federated"
      identifiers = [
        aws_iam_openid_connect_provider.this.arn
      ]
    }

    condition {
      test     = "StringEquals"
      variable = format("%s:aud", var.oidc_url)
      values   = [var.oidc_client_id]
    }

    condition {
      test     = "StringLike"
      variable = format("%s:sub", var.oidc_url)
      values = [
        var.project_repository_condition
      ]
    }
  }
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each   = { for k, v in var.policy_arns : k => v }
  policy_arn = each.value
  role       = aws_iam_role.this.name
}
```

2. Create `variables.tf`. Fill missing values.

```hcl
variable "role_name" {
  description = "Role for CircleCI."
  type        = string
  default     = "circleci-oidc-tf"
}

variable "oidc_url" {
  description = "The issuer of the OIDC token."
  type        = string
  default     = "oidc.circleci.com/org/ORGANIZATION_ID"
}

variable "oidc_client_id" {
  description = "Custom audience"
  type        = string
  default     = "ORGANIZATION_ID"
}

variable "oidc_thumbprint" {
  description = "Thumbprint of the issuer."
  type        = string
  default     = "Thumbprint"
}

variable "project_repository_condition" {
  description = ""
  type        = string
  default = "org/ORGANIZATION_ID/project/PROJECT_ID/user/*"
}

variable "policy_arns" {
  description = "A list of policy ARNs to attach the role"
  type        = list(string)
  default     = [
    "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
  ]
}

variable "default_tags" {
  description = "Default tags for AWS resources"
  type        = map(string)
  default     = {}
}
```

## Run CircleCI job to test identity provider.

1. Create `config.yml` in `.circleci` folder in your git repository.
```yaml
version: 2.1

orbs:
  aws-cli: circleci/aws-cli@3.1.4

jobs:
  circleci-oidc:
    parameters:
      aws_role_arn:
        type: string
        default: AWS_ROLE_ARN
    executor: aws-cli/default
    steps:
      - checkout
      - aws-cli/setup:
          role-arn: ${<< parameters.aws_role_arn >>}
      - run:
          name: Set default region
          command: echo "export AWS_REGION=us-east-1" >> $BASH_ENV
      - run:
          name: List S3 buckets
          command: aws s3 ls
      - run:
          name: List Elastic Container Registry (should fail, we did not grant permissions)
          command: aws ecr describe-repositories

workflows:
  main:
   jobs:
    - circleci-oidc:
       context:
        - just_oidc
```
2. Add environment variables in CircleCI
* Add `AWS_ROLE_ARN`, value is in format `arn:aws:iam::${AWS_ACCOUNT_ID}:role/circleci-oidc`

![CircleCI Secrets](/assets/post7/circleci-secret.png)

3. Add context to CircleCI.

In CircleCI jobs that use at least one context, the OpenID Connect ID token is available in the environment variable `$CIRCLE_OIDC_TOKEN`.

![Context](/assets/post2/context.png)

4. Push changes to git and check CircleCI

![Success](/assets/post7/circleci-check.png)
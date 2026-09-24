# AWS IAM Least-Privilege Security Lab

## Objective

Demonstrate how an overly permissive IAM policy can expose
S3 resources beyond an application's requirements, then
remediate the issue using least-privilege access controls.



## Scenario

The IAM user `app-dev` requires read access to application
documentation stored under:

s3://razvan-cloudsec-lab-23345/public/

The user should NOT be able to access objects under:

s3://razvan-cloudsec-lab-23345/private/

## Initial Configuration

The following intentionally vulnerable IAM policy was attached
to `app-dev`:

overly-permissive.json



## Security Findings

### 1. Account-wide S3 enumeration

```bash
╰─ aws sts get-caller-identity --profile app-dev
{
    "UserId": "<REDACTED>",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/app-dev"
}
```
```bash
╰─ aws s3 ls --profile app-dev
2026-09-18 15:17:32 razvan-cloudsec-lab-23345
```
```bash
╰─ aws s3 ls s3://razvan-cloudsec-lab-23345 --profile app-dev
                           PRE private/
                           PRE public/
```

                           
### 2. Private object enumeration and retrieval
                           
╰─ aws s3 ls s3://razvan-cloudsec-lab-23345/private/payroll.txt --profile app-dev
2026-09-24 11:18:23         61 payroll.txt

╰─ aws s3 cp s3://razvan-cloudsec-lab-23345/private/payroll.txt - --profile app-dev
CONFIDENTIAL TEST DATA - app-dev should NOT access this file

app-dev has access to confidential S3 objects in /private to list and read them.



## Root Cause

The problem is this overly permissive policy:
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*"



## Remediation

I replaced overly-permissive.json with a much secure policy least-privilege.json
The new policy:

- permits `ListBucket` only when the requested prefix is `public/`
- permits `GetObject` only against objects under `public/*`
- provides no Allow for private objects
- provides no Allow for account-wide bucket enumeration




## Validation

Can't list all S3 buckets:

╰─ aws s3 ls --profile app-dev
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListBuckets operation: User: arn:aws:iam::123456789012:user/app-dev is not authorized to perform: s3:ListAllMyBuckets because no identity-based policy allows the s3:ListAllMyBuckets action

Can't list /private/*:

╰─ aws s3 ls s3://razvan-cloudsec-lab-23345/private/ --profile app-dev
aws: [ERROR]: An error occurred (AccessDenied) when calling the ListObjectsV2 operation: User: arn:aws:iam::123456789012:user/app-dev is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::razvan-cloudsec-lab-23345" because no identity-based policy allows the s3:ListBucket action

Can't read /private/payroll.txt:

╰─ aws s3 cp s3://razvan-cloudsec-lab-23345/private/payroll.txt - --profile app-dev
download failed: s3://razvan-cloudsec-lab-23345/private/payroll.txt to - An error occurred (403) when calling the HeadObject operation: Forbidden

app-dev can read public and its content successfully:

╰─ aws s3 ls s3://razvan-cloudsec-lab-23345/public/ --profile app-dev
2026-09-18 15:20:12          0
2026-09-24 11:18:09         42 documentation.txt

╰─ aws s3 cp s3://razvan-cloudsec-lab-23345/public/documentation.txt - --profile app-dev
This file should be accessible by app-dev




---
layout: null
title:  "DDD Your Multi-Account Environments with StackSets"
date:   2020-06-21 08:50:27 +0100
archive: true
permalink: archive/CloudFormationIAM
comments: false
---
```json
{
    "Version": "2012-10-17",
    "Statement": [
       {
            "Effect": "Allow",
            "Action": [
                "cloudformation:ListStackSets"
            ],
            "Resource": "arn:aws:cloudformation:*:<AWS-ACCOUNT-ID>:stackset/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:*"
            ],
            "Resource": "arn:aws:cloudformation:*:<AWS-ACCOUNT-ID>:stackset/<STACKSET-NAME-PATTERN>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:HeadBucket",
                "s3:CreateBucket"
            ],
            "Resource": "arn:aws:s3:::cf-templates-*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<AWS-ACCOUNT-ID>:role/<CLOUDFORMATION-ADMIN-ROLE-NAME>"
        },
        {
            "Effect": "Allow",
            "Action": "sns:ListTopics",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:ListRoles",
            "Resource": "*"
        }
    ]
}
```
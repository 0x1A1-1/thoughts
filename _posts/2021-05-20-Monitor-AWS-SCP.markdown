---
layout: post
title:  "Terraform Monitoring to Your AWS Organization SCP"
name:   Monitor-AWS-SCP
date:   2021-05-20 17:24:36 +0100
categories: AWS
---


## Introduction
SCPs provide great means to manage accounts’ IAM at scale but also introduce a “Single Point of Failure” to effectively lock all accounts out. Detecting unintended/unauthorized changes made to SCP becomes increasingly critical to ensure the stability and availability of your cloud environment. 

With the wide adoption of AWS Organizations and separating workloads amongst different AWS accounts, [Service Control Policy(SCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) was introduced by AWS to make it easier to apply controls and guardrails for all accounts under an AWS Organization. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpg "Cover")

## Pre-requisites
- Access to your Organization Management Account 
- CloudTrail turned on in your Organization Management Account

## What to watch out for?
Refer to AWS Documentation on [API calls relevant to SCPs](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsorganizations.html#awsorganizations-actions-as-permissions), we notice the following action will likely indicator changes made to Organization SCPs:
```
AttachPolicy 
CreatePolicy
DeletePolicy
DetachPolicy
DisablePolicyType
EnablePolicyType
UpdatePolicy
```

## How does it look?

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/architecture.png "Architecture")

## How to set up monitoring with CloudWatch

### CloudWatch Rule Setup
Once we identify the API actions to watch out for, we can set up CloudWatch Rule that monitors the above CloudTrail event related to SCP. 
```
resource "aws_cloudwatch_event_rule" "monitor-CloudTrail-SCP" {
    description    = "scp-monitoring"
    event_bus_name = "default"
    event_pattern  = jsonencode(
        {
            detail      = {
                eventName   = [
                    "UpdatePolicy",
                    "CreatePolicy",
                    "DeletePolicy",
                    "DetachPolicy",
                    "DisablePolicyType",
                    "EnablePolicyType",
                    "AttachPolicy"
                ]
                eventSource = [
                    "organizations.amazonaws.com",
                ]
            }
            detail-type = [
                "AWS API Call via CloudTrail",
            ]
            source      = [
                "aws.organizations",
            ]
        }
    )
    is_enabled     = true
    name           = "scp-monitoring"
    tags           = {}
}
```

### SNS Topic Policy
We can then also set up the downstream for CloudWatch to an SNS topic that notifies certain email/SQS/etc. To set up the SNS, we first need to allow CloudWatch to publish the topic via the SNS topic policy
```
data "aws_caller_identity" "current" {}

data "aws_iam_policy_document" "scp_sns_topic_policy" {
  policy_id = "scp_sns_policy"
  statement {
    actions = [
      "SNS:Subscribe",
      "SNS:SetTopicAttributes",
      "SNS:RemovePermission",
      "SNS:Receive",
      "SNS:Publish",
      "SNS:ListSubscriptionsByTopic",
      "SNS:GetTopicAttributes",
      "SNS:DeleteTopic",
      "SNS:AddPermission",
    ]
    condition {
      test     = "StringEquals"
      variable = "AWS:SourceOwner"

      values = [
        data.aws_caller_identity.current.account_id,
      ]
    }
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = ["*"]
    }
    resources = [
      aws_sns_topic.scp-sns.arn,
    ]
  }

  statement {
    actions = ["SNS:Publish"]
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
    resources = [
      aws_sns_topic.scp-sns.arn,
    ]
    sid = "AWSEvents_scp-monitoring"
  }
}
```

### SNS Topic & Subscription Setup
With that set up, we can now create the SNS topic:
```
resource "aws_sns_topic" "scp-sns" {
  display_name      = "Org-SCP-Change-Detected"
  name              = "SCP-Change-Monitoring"
  tags              = {}
}

resource "aws_sns_topic_policy" "default" {
  arn = aws_sns_topic.scp-sns.arn
  policy = data.aws_iam_policy_document.scp_sns_topic_policy.json
}

resource "aws_sns_topic_subscription" "scp_sns_topic_subscription" {
  topic_arn = aws_sns_topic.scp-sns.arn
  protocol  = "email"
  endpoint  = "YOUR_TARGET_EMAIL_HERE"
}
```

### CloudWatch + SNS connection
Now comes the most important part, make the SNS topic a target of the CloudWatch Rule:
```
# aws_cloudwatch_event_target.scp-target:
resource "aws_cloudwatch_event_target" "scp-notify" {
    arn            = aws_sns_topic.scp-sns.arn
    rule           = aws_cloudwatch_event_rule.monitor-CloudTrail-SCP.name
    target_id      = "scp-notify"
    input_transformer {
        input_paths    = {
            "event"     = "$.detail.eventName"
            "event-id"  = "$.detail.eventID"
            "principal" = "$.detail.userIdentity.arn"
            "time"      = "$.detail.eventTime"
        }
        input_template = "\"The following <event> event for SCP was performed by <principal> at <time>. For more information please query CloudTrail with event ID: <event-id> or the accompanying secondary event payload email\""
    }
}

# aws_cloudwatch_event_target.scp-target2:
resource "aws_cloudwatch_event_target" "scp-target-payload" {
    arn            = aws_sns_topic.scp-sns.arn
    rule           = aws_cloudwatch_event_rule.monitor-CloudTrail-SCP.name
    target_id      = "scp-detailed-payload"
}
```

## Where to go from here

- `Lockdown` the Organization Management account. Policies can be applied to the root of an organization but they don’t have any effect on the root account. A good practice would therefore be to provide `minimal access` to your root account with IAM permissions, using it only to manage the Organization.

- Additionally, you would also want to set up some protection around the CloudWatch rule and SNS topics that we set up. A sophisticated actor could potentially “disabled” the monitoring before performing any action. One rule of thumb is to always keep the `defense-in-depth principle` in mind. 

- Another approach is to provide `time-based auditable IAM access` into the root account. For ideas, you can check out my posts on [Time-based control for IAM]({{ site.baseurl }}{% post_url 2021-04-19-Time-based-IAM %}) and [Tag enforcement]({{ site.baseurl }}{% post_url 2021-04-21-CF-Tag-Enforcement %}).

<br>

## Conclusion
While prevention is ideal, detection is a must. On top of locking down your Organization Management account, setting up proper alerting gives your security team the confidence of using SCP to manage cloud accounts at scale.

If you want the entire codebase that I was referring to for easier deployment, you can navigate over to my Github for the [Terraform template](https://github.com/0x1A1-1/AWS-SCP-Monitoring).
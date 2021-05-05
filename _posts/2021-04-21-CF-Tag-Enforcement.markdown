---
layout: post
title:  "Tag Enforcement for CloudFormation"
name:   CF-Tag-Enforcement
date:   2021-04-21 09:34:53 +0100
categories: AWS
---


## Introduction
AWS CloudFormation allows [Parameters](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html) with regular expression requirement. We can explore this for resource tag enforcement while interacting with CloudFormation. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpeg "Cover")

## Enforcing Strict Audit Trail

Follow my [previous post]({{ site.baseurl }}{% post_url 2021-04-19-Time-based-IAM %}) on Time-based IAM. Oftentimes, we want to leave an audit trail for any time someone escalates their privilege. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cloudtrail.png "Cloudtrail")

 Note that each CloudFormation action is indeed logged in CloudTrail but under the identity of `cloud_user` or `admin`. We wouldn't necessarily be able to infer `WHO` exactly invoked the escalation. As a result, we can enforce some sort of free text field during the creation process(e.g., JIRA ticket) so that the escalation of privilege is recorded in CloudTrail with a trackable reference. Here is a sample of enforcing JIRA ticket ID for the privilege IAM role creation.


``` yaml
AWSTemplateFormatVersion: 2010-09-09
Description: This template builds a time-boxed privilege IAM
Parameters:
  SourceAccountNumber:
    Description: Source account number
    Type: String
  TicketID:
    Description: JIRA ticket ID for escalation justification
    Type: String
    AllowedPattern: "[a-zA-Z]+-[0-9]+"
Resources:
  ElevatedAdmin:
    Type: 'AWS::IAM::Role'
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              AWS:
                - !Sub 'arn:aws:iam::${SourceAccountNumber}:role/<SOURCE_IAM_ROLE>'
            Action:
              - 'sts:AssumeRole'
      Path: /Admin/
      RoleName: Timed-Elevated-Admin
      Policies: [...]
      Tags:
        - Key: JIRA_Ticket
          Value: !Ref TicketID

```
Once you set this up, you can further monitor escalation by setting up a [CloudWatch rule](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/monitor-cloudtrail-log-files-with-cloudwatch-logs.html) in each trail to monitor the creation of the privileged role and notify the related party accordingly.

Additionally, you can even build a Lambda function the accepts incoming event bus from CloudWatch to verify the legitimacy of that tag field(e.g., JIRA ticket).

<br>

## Conclusion
In this post, we went through tag enforcement for CloudFormation deployment. With the regular expression field for the parameter, you can be creative and draft whatever requirement your organization has regarding the audit trail.

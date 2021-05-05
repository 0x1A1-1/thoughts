---
layout: post
title:  "Time-based control for IAM"
name:   Timed-IAM
date:   2021-04-19 16:22:03 +0100
categories: AWS
---

## Introduction
Roles with persistent escalated permissions are considered risky and provide a high-value target for attackers. However, Infrequent elevated privileges are still required for business needs on managing cloud infrastructure. A time-based pattern provides access for the platform and security team while ensuring the security of our cloud infrastructure by limiting the lifespan of escalated permissions. Requests for elevated privilege should be logged for future audit and threat detection. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpg "Cover")

## The idea
Follow my [previous post]({{ site.baseurl }}{% post_url 2020-06-21-StackSets %}){:target="_blank"} on StackSet. You already have a way to manage your multi AWS accounts at scale. Now the time comes you need to perform some administrative task in one of the target accounts, what should you do? 

The security and DevOps team occasionally needs a powerful IAM role to operate in AWS accounts with administrative permission for escalation. However, if the over-permissive role has a persistent presence in a deployed account, it becomes a clear target since it can be assumed by a human. 

Luckily, something that IAM offers in its policy language is the condition field. We can parametrize the CloudFormation template of the IAM role with a validity timestamp in its condition field, as defined by IAM [timed condition](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_multi-value-conditions.html). This approach will expire the IAM permissions after the denoted timestamp.


![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/condition.png "IAM-Condition")

So the Cloud Formation template would look something like the following:

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: This template builds a time-boxed privilege IAM
Parameters:
  SourceAccountNumber:
    Description: Source account number
    Type: String
  PermissionStartTime:
    Description: Timestamp for when the permission starts. https://www.w3.org/TR/NOTE-datetime
    Type: String
    Default: '2019-07-16T12:00:00Z'
    AllowedPattern: .+
  PermissionTerminateTime:
    Description: Timestamp for when the permission ends. https://www.w3.org/TR/NOTE-datetime
    Type: String
    Default: '2019-07-16T15:00:00Z'
    AllowedPattern: .+
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
      Policies:
        - PolicyName: Timed-Elevated-Policy
          PolicyDocument:
            Version: 2012-10-17
            Statement:
              - Effect: Deny
                Action:
                  - 'iam:AttachRolePolicy'
                  - 'iam:DeleteRolePolicy'
                  - 'iam:DetachRolePolicy'
                  - 'iam:PutRolePolicy'
                  - 'iam:UpdateAssumeRolePolicy'
                Resource: !Sub 'arn:aws:iam::${AWS::AccountId}:role/Admin/Timed-Elevated-Admin'
              - Effect: Allow
                Action: '*'
                Resource: '*'
                Condition:
                  DateGreaterThan:
                    'aws:CurrentTime': !Ref PermissionStartTime
                  DateLessThan:
                    'aws:CurrentTime': !Ref PermissionTerminateTime
```
It is important to always think further in terms of IAM privilege escalation, therefore more restriction around the IAM role itself is created in the CloudFormation template. However, if you notice that `CreateRole` is not blocked, and one can pretty easily set up another shadow admin. One can always add in more restrictions in creating roles, attaching policy, update policy, etc.  

<br>

## Removal of the role
We can either use the AWS Step Function `wait state` timer module or a CloudWatch timer to kick off Lambda before we trigger the removal of the target IAM role. Below is the sample code to remove the IAM role provisioned by the StackSets execution.
  
``` python
class CloudFormation(object):
    def __init__(self, region):
        self.client = boto3.client("cloudformation", region)

    def delete_stack_instances(self, stack_set_name, accounts, regions, region_order=None):
        if region_order is None:
            region_order = []

        operation_id = self.client.delete_stack_instances(
            StackSetName=stack_set_name,
            Accounts=accounts,
            Regions=regions,
            OperationPreferences={
                "RegionOrder": region_order,
                "FailureTolerancePercentage": 0,
                "MaxConcurrentPercentage": 100
            },
            RetainStacks=False
        )["OperationId"]
        return operation_id
```

<br>

## What Do You Need to Protect Now?

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/scp.png "Service Control Policy")

By relying on the `AWSCloudFormationStackSetExecutionRole` Role, we assumed that the ExecutionRole will have the permission in the account to do as instructed. To ensure this is always the case, we can create an organization-level [Service Control Policy](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html)(SCPs) to protect the role once it’s created. 


``` json
{
    "Version": "2012-10-17",
    "Statement": {
    	 "Sid": "DenyAccessToImportantRole",
            "Effect": "Deny",
            "Action": [
                "iam:AttachRolePolicy",
                "iam:DeleteRole",
                "iam:DeleteRolePermissionsBoundary",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PutRolePermissionsBoundary",
                "iam:PutRolePolicy",
                "iam:UpdateAssumeRolePolicy",
                "iam:UpdateRole",
                "iam:UpdateRoleDescription"
            ],
            "Resource": [
                "arn:aws:iam::*:role/AWSCloudFormationStackSetExecutionRole"
            ]
     }
}
```


We went over protecting privilege IAM roles in the member accounts, but we still have...ahem...a single point of failure, which is our StackSets master account. We want the following IAM rule to be attached to the operator, so they cannot arbitrarily modify the CloudFormation template used to deploy the IAM. 


<details>
<summary>
{% highlight raw %}
IAM Policy for Operator
{% endhighlight %}
</summary>

{% highlight json %}
{% raw %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudformation:ListStackSets"
            ],
            "Resource": "arn:aws:cloudformation:*:<ACCOUNT-ID>:stackset/*",
            "Effect": "Allow"
        },
        {
            "Action": [
                "cloudformation:CreateStackInstances",
                "cloudformation:CreateUploadBucket",
                "cloudformation:DeleteStackInstances",
                "cloudformation:DescribeAccountLimits",
                "cloudformation:DescribeChangeSet",
                "cloudformation:DescribeStackDriftDetectionStatus",
                "cloudformation:DescribeStackEvents",
                "cloudformation:DescribeStackInstance",
                "cloudformation:DescribeStackResource",
                "cloudformation:DescribeStackResourceDrifts",
                "cloudformation:DescribeStackResources",
                "cloudformation:DescribeStackSet",
                "cloudformation:DescribeStackSetOperation",
                "cloudformation:DescribeStacks",
                "cloudformation:DescribeType",
                "cloudformation:DescribeTypeRegistration",
                "cloudformation:DetectStackDrift",
                "cloudformation:DetectStackResourceDrift",
                "cloudformation:DetectStackSetDrift",
                "cloudformation:EstimateTemplateCost",
                "cloudformation:GetStackPolicy",
                "cloudformation:GetTemplate",
                "cloudformation:GetTemplateSummary",
                "cloudformation:ListChangeSets",
                "cloudformation:ListExports",
                "cloudformation:ListImports",
                "cloudformation:ListStackInstances",
                "cloudformation:ListStackResources",
                "cloudformation:ListStackSetOperationResults",
                "cloudformation:ListStackSetOperations",
                "cloudformation:ListStacks",
                "cloudformation:ListTypeRegistrations",
                "cloudformation:ListTypeVersions",
                "cloudformation:ListTypes",
                "cloudformation:UpdateStackInstances",
                "cloudformation:ValidateTemplate"
            ],
            "Resource": "arn:aws:cloudformation:*:<ACCOUNT-ID>:stackset/<ADMIN-IAM-STACKSET>*",
            "Effect": "Allow"
        }
    ]
}

{% endraw %}
{% endhighlight %}
</details> 


## Conclusion
In this post, we went through using the time condition field for IAM policy, protecting IAM role in both identity-based policy and Service Control Policy(SCP). The journey to protect IAM never ends, some additional idea worth exploring include: [`Tag enforcement for CloudFormation deployment`]({{ site.baseurl }}{% post_url 2021-04-21-CF-Tag-Enforcement %}), `timed access for user-based removal of Active Directory group`, `Using Hashicorp Vault to provide short term IAM credentials`

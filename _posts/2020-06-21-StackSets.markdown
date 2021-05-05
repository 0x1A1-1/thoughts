---
layout: post
title:  "Manage Your Multi-Account Environments with StackSets"
name:   StackSets
date:   2020-06-21 08:50:27 +0100
categories: AWS
kramdown:
  coderay_wrap: true
---
## Introduction
So you have decided to move into the Multi-Account model for your AWS account structure. Now the natural topic that comes to mind is how to centrally manage infrastructures in all the accounts. This blog is to equip you with some knowledge and toolings to make life _easier_. We are going to explore using AWS StackSets to deploy Infrastructure as Code(IaC) in all of your AWS Accounts.

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpg "Cover")

## What is StackSets?
StackSets, as its name suggests, are sets of CloudFormation stacks. It was announced by AWS to enable the management of multiple AWS accounts and regions. By using this service, you effectively delegate AWS to assume a role into each of your target accounts(known as Stack Instances) to perform CloudFormation Stack actions, which removes the need for you to configure access into all the accounts every time you try to run an update on your infrastructure. 


#### StackSets Prerequisites:
* StackSets functions under the spoke-hub model(e.g, Administrator Account vs. Member Accounts). Therefore, you must create the correct account access linkage between the controller account and the target account, during the account provisioning process. You can create the execution role with [AWSCloudFormationStackSetExecutionRole.yml](https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetExecutionRole.yml)
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/spoke-hub.png "spoke-hub")

* In the Administrator account, you must create a role named AWSCloudFormationStackSetAdministrationRole, which is used by StackSets service to assume into target accounts to perform operations. The CloudFormation link for this role creation is [AWSCloudFormationStackSetAdministrationRole.yml](https://s3.amazonaws.com/cloudformation-stackset-sample-templates-us-east-1/AWSCloudFormationStackSetAdministrationRole.yml)


* Lastly, you will need a role in your Administrator account to have [sufficient permission](/thoughts/archive/CloudFormationIAM){:target="_blank"} to manage StackSets

* Another common gotcha is that stack set operations involving regions that are disabled by default would result in an error. (Documentation on enabling needed region can be found [here](https://docs.amazonaws.cn/general/latest/gr/rande-manage.html))

## The Meat and Potatoes
Once you have StackSets permissions configured, it is time to unleash the power of IaC at scale. We are going to take a simple CloudFormation template that creates an IAM user and multiple groups with different policies in all of the member accounts. 

Within the CloudFormation console, you can select Create StackSet in us-east-1 region
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/step0.png "Step 0")
1. We start off with this [CloudFormation template](
https://s3.amazonaws.com/cloudformation-templates-us-east-1/IAM_Users_Groups_and_Policies.template)
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/step1.png "Step 1")
2. Next step, Give your StackSet a distinctive name, which could later be used by IAM least privilege scoping, and fill in the [parameter](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html) details for the CloudFormation template. 
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/step2.png "Step 2")
3. Now you are moving on to instructing StackSet which IAM role it should use to execute the CloudForamtion on your behalf. Refer to the StackSets prerequisites section on the role if you don’t know how to fill in this part.
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/step3.png "Step 3")
4. Lastly, if you are creating IAM resources, you must check the IAM Capability box, otherwise you will get an `InsufficientCapabilities` later on
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/step4.png "Step 4")

After the StackSets are created, you can mass onboard accounts into StackSets with a comma separated list. Or if you already have an organizational structure, you can onboard an entire Organization Unit(OU). To achieve this, you must first enable [StackSets integration with AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/services-that-can-integrate-cloudformation.html). 
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/deploy-step1.png "Deploy Step 1")

### CLI Way
Or if you’re like me and prefer have all the things running in terminal, you can use the following command with you AWS CLI

#### Create StackSets
`aws cloudformation create-stack-set --stack-set-name YOUR_STACK_SET_NAME --template-url YOUR_CF_TEMPLATE_URL --parameters ParameterKey=PARAMETERS_KEY,ParameterValue=PARAMETERS_VALUE --region STACKSET_REGION(e.g us-east-1) --capabilities TARGET_CAPABILITIES`. You can learn more about StackSet Parameters [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html)


In your terminal with IAM credential loaded in environment, run: 
```shell
~ ❯ aws cloudformation create-stack-set --stack-set-name IAM-StackSets --template-url https://s3.amazonaws.com/cloudformation-templates-us-east-1/IAM_Users_Groups_and_Policies.template --parameters ParameterKey=Password,ParameterValue=YOUR_SUPER_SECRET_PASSWORD  --region us-east-1 --capabilities CAPABILITY_NAMED_IAM
```


#### Create Stack Instances
We can now push out the StackSets to all of our target accounts
`aws cloudformation create-stack-instances --stack-set-name YOUR_STACK_SET_NAME --accounts ACCOUNT_1 ACCOUNT_2 ACCOUNT_3 --regions REGION_1 REGION_2 --operation-preferences FailureToleranceCount=1 --region STACKSET_REGION`. 

For our example, we will run the following:
```
~ ❯ aws cloudformation create-stack-instances --stack-set-name IAM-StackSets --accounts TARGET_ACCOUNT_NUMBER --regions us-east-1 --operation-preferences FailureToleranceCount=1 --region us-east-1
```
Which would send OperationId in response body that looks like:
```json
{
    "OperationId": "40e86391-ff04-4334-850d-f49e1axxxxxxx"
}
```

#### Check Stack Operation
You can now check deployment statue with CLI 
```
~ ❯ aws cloudformation describe-stack-set-operation --stack-set-name IAM-StackSets --operation-id 40e86391-ff04-4334-850d-f49e1axxxxxxx --region us-east-1
```
with response body look like the following:
```json
{
    "StackSetOperation": {
        "OperationId": "40e86391-ff04-4334-850d-f49e1axxxxxxx",
        "StackSetId": "IAM-StackSets:0539314c-660e-4296-963c-029bd9bc0144",
        "Action": "CREATE",
        "Status": "SUCCEEDED",
        "OperationPreferences": {
            "RegionOrder": [],
            "FailureToleranceCount": 1
        },
        "AdministrationRoleARN": "arn:aws:iam::xxxxxxxxx:role/AWSCloudFormationStackSetAdministrationRole",
        "ExecutionRoleName": "AWSCloudFormationStackSetExecutionRole",
        "CreationTimestamp": "2020-06-21T05:31:27.075000+00:00",
        "EndTimestamp": "2020-06-21T05:32:32.521000+00:00"
    }
}
```

#### Verify 
As last step, you can confirm if the IAM Resource is created in the target account.
![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/verify.png "Verify IAM")

By the way, this example is terrible, because Access Key and Secret Key are printed on the Stack output. But the workflow should give you a good idea on how to use StackSets to deploy IaC at scale.

## Conclusion
StackSets extends the Infrastructure as Code(IaC) capability across multiple accounts and regions. By proper initial setup, you could manage accounts at scale with IaC CRUD efficiently and securely. 

You can even have a CloudFormation template of your StackSets configuration: <https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-stackset.html>

Some future ideas I plan to explore are: [`Time-based control for IAM`]({{ site.baseurl }}{% post_url 2021-04-19-Time-based-IAM %}), [`Tag enforcement in StackSets`]({{ site.baseurl }}{% post_url 2021-04-21-CF-Tag-Enforcement %})


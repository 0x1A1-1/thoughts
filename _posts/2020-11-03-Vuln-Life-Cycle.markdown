---
layout: post
title:  "Alert & Remediate AWS Cloud Misconfigurations with Step Functions"
name: Socless
date:   2020-11-03 14:32:27 +0100
categories: AWS
---

## Introduction
In the modern cloud where democratized access to environments is granted to engineers, setting up guardrail is extremely important. While prevention is ideal, detection is a must. How does the security better scale its detection and response capability with the (hyper)growth of the organization? In this post, I will briefly go over some of the lessons learned for remediating cloud misconfigurations/vulnerability through AWS Step Functions.  

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpg "Cover")

## What are Step Functions, and Why Socless?
> [AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/tutorial-creating-lambda-state-machine.html) is a serverless function orchestrator that makes it easy to sequence AWS Lambda functions and multiple AWS services into business-critical applications. Through its visual interface, you can create and run a series of checkpointed and event-driven workflows that maintain the application state. 

Twilio’s Security team built on top of Step Functions and open-sourced [SOCless](https://twilio-labs.github.io/socless). SOCless, deployed with Serverless Framework, does ***more than*** just orchestrating lambda for you. It constructs a dedicated VPC, provisions NAT gateway and Elastic IPs, and creates various DynamoDB for you on top of the Lambda and Step Function management. 

```yaml
{% raw %}
custom:
  soclessPackager:
    buildDir: build
resources:
  - ${{file(resources/dynamodb.yml)}} # DynamoDB Tables
  - ${{file(resources/iam.yml)}} # IAM Resources
  - ${{file(resources/sfn.yml)}} # Step Functions Resources
  - ${{file(resources/s3.yml)}} # S3 Resources
  - ${{file(resources/kms.yml)}} # KMS Resources
  - ${{file(resources/vpc.yml)}} # VPC Resources
  - ${{file(resources/sg.yml)}} # Security Group resources
  - ${{file(resources/apigateway.yml)}} # API Gateway
{% endraw %}
```

***

## Prevention is ideal, but detection is a must. 
To trigger Step Function, we will need a security event, being it resource modification or indicator of compromise, that invokes what we call an Endpoint Lambda. There are several ingress event source you can configure, really, the sky's the limit:
* CloudWatch for your CloudTrail
* [S3 event notification](https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html)
* Third-party detection engine 

The [Endpoint Lambda](https://twilio-labs.github.io/socless/your-first-endpoint/) function essentially parses the event and hydrates details for the Step Function execution and the fleet of Lambda it commands. Moreover, SOCless helps you save the event details into a DynamoDB so you will never lose inflight events. You can even consume those details into your Datadog or Grafana Dashboard if you’d like. But yes, I hear you: an event ingestion engine/dashboard for the event ingestion engine, turtles all the way down!

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/Socless.png "Architecture")

`Another key point here, if you craft your endpoint lambda generic enough, you can rely on only one endpoint for each event source. `

***

# Command your army of Lambdas

Now with your event details recorded and dispatched to your Step Functions execution. You can create Lambdas that **1)** Further process the event and enrich the details by making correlation to another source **2)** alert resource owner and/or security team via Slack or JIRA **3)** Immediately fix the problem if it’s high risk. The beauty is that you can string as many Lambda together as you’d like. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/Playbook.png "Playbook")

Following the [first playbook](https://twilio-labs.github.io/socless/your-first-playbook/) example, you can get a sense of how to feed the output of one Lambda to another. Seriously, you can write your Lambda to do anything you want as part of the detection and response flow. What’s even better, is that you can create the Lambda once, and re-use it many times in various response playbooks. 

Some common gotchas I’ve learned “painfully” while dealing with Lambdas are:
Lambda IAM profile needs permissions to do what it needs 
Keep your lambda warm
Lambda will fail. Make sure you build them with that in mind. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/LambdaFail.png "LambdaFail")

Good news for the last part, SOCless already tried to construct built-in retry logic for your Step Function executions while [rendering](https://github.com/twilio-labs/sls-apb/blob/master/lib/constants.ts#L2) the playbook:
``` javascript
export const DEFAULT_RETRY = Object.freeze({
  ErrorEquals: [
    "Lambda.ServiceException",
    "Lambda.AWSLambdaException",
    "Lambda.SdkClientException",
  ],
  IntervalSeconds: 2,
  MaxAttempts: 6,
  BackoffRate: 2,
});
```

***

## Conclusion
To scale your SIEM in the modern cloud environment with hypergrowth in the organization, relying on SOC operation engineers is no longer an option. **Lambdas and Step Functions** allow you to build once, re-use many times. With extreme flexibility and extensibility, you can adapt to any new cloud resource type while freeing up your security engineering team to more sophisticated tasks and goals. The automation also gives a faster feedback loop in the event of misconfiguration, significantly reducing the detection to remediation time, therefore improving the maturity of your security organization.

### Where Next?
Set up customized auto-remediation where AWS Config doesn’t allow you to.
Additionally, you can create your own pipeline in deploying the everything with Serverless framework

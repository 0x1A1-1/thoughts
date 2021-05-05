---
layout: post
title:  "CI/CD Pipeline for Serverless Framework"
name:   Serverless-CICD
date:   2021-01-22 08:50:27 +0100
categories: AWS
---
## Introduction
[Serverless Framework](https://www.serverless.com/) provides you with scaffolding, workflow automation, and best practices for developing and deploying your serverless architecture. However, as [part of the setup steps](https://www.serverless.com/framework/docs/providers/aws/guide/credentials/), it instructs you to create IAM user and static IAM access keys. Creating access keys is almost **never** a good practice. Instead, we are going to set up a deployment pipeline for your Serverless application, removing the dependency on static credentials and improving the resiliency of your system. 

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/cover.jpg "Cover")

## Create local unit test suites
To create a robust deployment pipeline, verifying any breaking changes you’ve made is instrumental in keeping the overall system stable. It will be really interesting to run integration tests without tests themselves. The fact that unit tests are worth their dedicated book means that I cannot extend too far in this topic. Instead, I recommend you to explore [Pytest](https://docs.pytest.org/) and [Localstack](https://github.com/localstack/localstack) for testing framework and mocking upstream/downstream dependencies. 

My recommendation is to run all of the unit test suites in a Docker container. It not only creates an isolated environment for your software packages but also mimics the real deployment environment provided by AWS CodeBuild, to give you the best chance of not running into any “Stackoverflow worthy” error. 

`Dockerfile`
```
FROM python:3.6

# Unit Testing Env VAR
ENV MOTO_ACCOUNT_ID 123456789012
ENV AWS_REGION us-west-2
ENV AWS_DEFAULT_REGION us-west-2
ENV AWS_ACCESS_KEY_ID testing
ENV AWS_SECRET_ACCESS_KEY testing
ENV AWS_SECURITY_TOKEN testing
ENV AWS_SESSION_TOKEN testing
ENV PYTHONWARNINGS ignore::DeprecationWarning:(boto.*|werkzeug.*|socks.*)

# application folder
ENV APP_DIR /app

# app dir
ADD . ${APP_DIR}
WORKDIR ${APP_DIR}

RUN pip install pytest boto3 pytest-cov
RUN pip install -r functions/requirements.txt
```


`docker-compose.yml`
``` yaml
version: '2'
services:
  demo-project:
    build: .
    # enable this while doing local test
    # network_mode: host 
    container_name: DEMO-PROJECT
    volumes:
      - .:/app
    environment:
      - MOTO_ACCOUNT_ID=123456789012
      - AWS_REGION=us-west-2
      - AWS_DEFAULT_REGION=us-west-2
      - AWS_ACCESS_KEY_ID=testing
      - AWS_SECRET_ACCESS_KEY=testing
      - AWS_SECURITY_TOKEN=testing
      - AWS_SESSION_TOKEN=testing
      - PYTHONWARNINGS=ignore::DeprecationWarning:(boto.*|werkzeug.*|socks.*)
    command: pytest tests/ --cov --cov-report term-missing
```

<br>

## Instantiate the Pipeline

### Create from an existing pipeline
If you have an existing CodeBuild and CodePipeline flow that you liked, you can use [batch-get-projects](https://docs.aws.amazon.com/cli/latest/reference/codebuild/batch-get-projects.html) and [get-pipeline](https://docs.aws.amazon.com/cli/latest/reference/codepipeline/get-pipeline.html) command with AWS CLI to “reverse-engineer” a CloudFormation Template. You can keep the json returned by AWS CLI, or paste it into the CloudFormation visual tool provided by AWS and convert it into YAML. 
### Start from Scratch using CloudFormation 
You can refer to [CodeBuild template](https://github.com/stelligent/cloudformation_templates/blob/master/labs/codebuild/codebuild.yml) and  [CodePipeline template](https://github.com/stelligent/cloudformation_templates/blob/master/labs/codepipeline/codepipeline-canonical.yml). There are also many other existing templates out in Github, feel free to explore around too. 

### Here’s what my CI/CD pipeline’s CloudFormation template and workflow look like 

``` yaml
{% raw %}
service: socless-divvycloud

provider:
  name: aws
  runtime: python3.7
  variableSyntax: "\\${{([ ~:a-zA-Z0-9._\\'\",\\-\\/\\(\\)]+?)}}"
  stage: ${{opt:stage}}
  stackName: ${{self:service}}

resources:
  - Description: Cloudformation stack for ${{self:service}} CI/CD Pipeline
  - ${{file(resources/CodeBuild.yml)}} # Unit test build ci
  - ${{file(resources/CodePipeline.yml)}} # Continous delivery pipeline
{% endraw %}
```

<details>
<summary>
CodeBuild.yml
</summary>

{% highlight yaml %}
{% raw %}
Resources:
  CodeBuildeCI:
    Type: 'AWS::CodeBuild::Project'
    Properties:
      Name:
        Fn::Sub: "${AWS::StackName}-build-ci"
      Description:
        Fn::Sub: "CI Codebuild for ${AWS::StackName}"

      Source:
        BuildSpec: buildspec/buildspec-ci.yml
        Type: GITHUB
        Location: 'YOUR-REPO-PATH'
        GitCloneDepth: 0
        GitSubmodulesConfig:
          FetchSubmodules: false
        ReportBuildStatus: false
        InsecureSsl: false
      SecondarySources: []
      SecondarySourceVersions: []

      Artifacts:
        Type: S3
        Location: BUILD-ARTIFACT-S3-BUCKET-NAME
        Path: build-artifacts/
        NamespaceType: NONE
        Name:
          Fn::Sub: "${AWS::StackName}-build-ci"
        Packaging: ZIP
        OverrideArtifactName: true
        EncryptionDisabled: false
      SecondaryArtifacts: []
      Cache:
        Type: NO_CACHE
      Environment:
        Type: LINUX_CONTAINER
        Image: 'aws/codebuild/standard:4.0'
        ComputeType: BUILD_GENERAL1_SMALL
        EnvironmentVariables: []
        PrivilegedMode: true
        Certificate:
          Fn::Sub: "${{self:custom.githubCertificatePath}}"
        ImagePullCredentialsType: CODEBUILD

      ServiceRole: arn:aws:iam::ACCOUNT:role/service-role/CODEBUILD-SEVICE-ROLE-NAME
      TimeoutInMinutes: 60
      QueuedTimeoutInMinutes: 480
      EncryptionKey:
        Fn::Sub: "arn:aws:kms:${{self:provider.region}}:${AWS::AccountId}:alias/aws/s3"
      Tags: []
      BadgeEnabled: false
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED
          GroupName: demo-project
          StreamName:
            Fn::Sub: "${AWS::StackName}-unit-test"
      FileSystemLocations: []

      Triggers:
        Webhook: true
        FilterGroups:
          - - Type: EVENT
             Pattern: PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED, PULL_REQUEST_MERGED
             ExcludeMatchedPattern: false


{% endraw %}
{% endhighlight %}
</details>

<details>
<summary>
CodePipeline.yml
</summary>

{% highlight yaml %}
{% raw %}
Resources:
  CodeBuildeCD:
    Type: 'AWS::CodeBuild::Project'
    Properties:
      Name:
        Fn::Sub: "${AWS::StackName}-build-cd"
      Description:
        Fn::Sub: "CD Codebuild for ${AWS::StackName}"
      Source:
        Type: CODEPIPELINE
        BuildSpec: buildspec/buildspec-cd.yml
        InsecureSsl: false
      SecondarySourceVersions: []
      Artifacts:
        Type: CODEPIPELINE
        Name:
          Fn::Sub: "${AWS::StackName}-build-cd"
        Packaging: NONE
        EncryptionDisabled: false
      Cache:
        Type: NO_CACHE
      Environment:
        Type: LINUX_CONTAINER
        Image: 'aws/codebuild/standard:4.0'
        ComputeType: BUILD_GENERAL1_SMALL
        EnvironmentVariables:
          - Name: DEPLOYMENT_ENV
            Value: sandbox
            Type: PLAINTEXT
        PrivilegedMode: true
        ImagePullCredentialsType: CODEBUILD

      ServiceRole: arn:aws:iam::ACCOUNT:role/service-role/CODEBUILD-SEVICE-ROLE-NAME
      TimeoutInMinutes: 60
      QueuedTimeoutInMinutes: 480

      EncryptionKey:
        Fn::Sub: "arn:aws:kms:${{self:provider.region}}:${AWS::AccountId}:alias/aws/s3"
      Tags: []
      BadgeEnabled: false
      LogsConfig:
        CloudWatchLogs:
          Status: ENABLED
          GroupName: demo-project
          StreamName:
            Fn::Sub: "${AWS::StackName}-deployment"
        S3Logs:
          Status: DISABLED
          EncryptionDisabled: false
      FileSystemLocations: []


  CodePipeline:
    Type: AWS::CodePipeline::Pipeline
    Properties:
      Name:
        Fn::Sub: "${AWS::StackName}-cd-pipeline"

      ServiceRole: arn:aws:iam::ACCOUNT:role/service-role/CODEPIPELINE-SEVICE-ROLE-NAME

      ArtifactStores:
        -
          Region: us-east-1
          ArtifactStore:
            Type: S3
            Location: BUILD-ARTIFACT-S3-BUCKET-NAME
      Stages:
        - Name: Source
          Actions:
            - Name: Source
              ActionTypeId:
                Category: Source
                Owner: AWS
                Provider: S3
                Version: '1'
              RunOrder: 1
              Configuration:
                PollForSourceChanges: 'false'
                S3Bucket: BUILD-ARTIFACT-S3-BUCKET-NAME
                S3ObjectKey:
                  Fn::Sub: "build-artifacts/${AWS::StackName}-PULL_REQUEST_MERGED"
              OutputArtifacts:
                - Name: SourceArtifact
              InputArtifacts: []

              Region: us-east-1
              Namespace: SourceVariables
        - Name: Deploy-To-Dev
          Actions:
            - Name: Deploy-to-dev
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              RunOrder: 1
              Configuration:
                EnvironmentVariables: '[{"name":"DEPLOYMENT_ENV","value":"dev","type":"PLAINTEXT"}]'
                ProjectName:
                  Fn::Sub: "${CodeBuildeCD}"
              OutputArtifacts:
                - Name: BuildArtifacts
              InputArtifacts:
                - Name: SourceArtifact
              Region: ${{self:provider.region}}
              Namespace: BuildVariables
        - Name: Manual-Approval
          Actions:
            - Name: Manual-Approval
              ActionTypeId:
                Category: Approval
                Owner: AWS
                Provider: Manual
                Version: '1'
              RunOrder: 1
              Configuration:
                NotificationArn: 'arn:aws:sns:REGION:ACCOUNT_ID:SNS_NAME'
              OutputArtifacts: []
              InputArtifacts: []
              Region: ${{self:provider.region}}
        - Name: Deploy-To-Prod
          Actions:
            - Name: Deploy-To-Prod
              ActionTypeId:
                Category: Build
                Owner: AWS
                Provider: CodeBuild
                Version: '1'
              RunOrder: 1
              Configuration:
                EnvironmentVariables: '[{"name":"DEPLOYMENT_ENV","value":"prod","type":"PLAINTEXT"}]'
                ProjectName:
                  Fn::Sub: "${CodeBuildeCD}"
              OutputArtifacts: []
              InputArtifacts:
                - Name: SourceArtifact
              Region: ${{self:provider.region}}

{% endraw %}
{% endhighlight %}
</details> <br>

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/workflow.png "workflow")

<br>


## CI/CD Triggers

### Continuous Integration via Codebuild & Github
When a Pull Request event is generated amongst PULL_REQUEST_CREATED, PULL_REQUEST_UPDATED, PULL_REQUEST_REOPENED, PULL_REQUEST_MERGED, the CodeBuild execution downloads the branch from GitHub, verifies the code can install, and zips the repo files to an S3 bucket for CICD Artifacts under the name `<StackName>-<PULL_REQUEST_EVENT>`, in our example it will be `demo-project-PULL_REQUEST_CREATED`.

`buildspec/buildspec-ci.yml`
``` yaml 
version: 0.2
phases:
  build:
    commands:
      - docker-compose up --abort-on-container-exit
artifacts:
  files:
    - '**/*'
  # Use webhook event so that merge event can be filtered by code pipeline
  name: demo-project-$CODEBUILD_WEBHOOK_EVENT
```

### Continuous Delivery via CodePipeline & CodeBuild
A CloudWatch Event rule for this repo watches the CICD artifact’s bucket for a PutObject event of file `<StackName>-PULL_REQUEST_MERGED`. When this event is detected, CloudWatch triggers this repo’s CodePipeline.

CodePipeline retrieves the repo file from S3 and sends it to this repo’s CodeBuild “CD” buildspec, which installs and deploys the repo to DEV environment via npm & serverless framework. Note that the docker username and password are required because of Docker limiting [anonymous pull](https://docs.docker.com/docker-hub/download-rate-limit/).

`buildspec/buildspec-cd.yml`
```yaml
version: 0.2

env:
  parameter-store:
     DOCKER_USERNAME: /demo-project/docker/username
     DOCKER_PASSWORD: /demo-project/docker/password

phases:
  install:
    commands:
      - npm install
  build:
    commands:
      - "docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD"
      - npm run $DEPLOYMENT_ENV
```
The deployment uses CodeBuildeCD service role's IAM permission, instead of the static IAM credential we mentioned above. If deployment and testing pass in DEV, CodePipeline will send a message to the Manual Approval SNS Topic. The pipeline pauses until an engineer approves the message, and the pipeline will fail if it is not approved within 7 hours. When an engineer manually authorizes the Pipeline, it will continue the deployment process above in the Production environment.

![Alt text]({{ site.baseurl }}/assets/{{ page.name }}/pipeline.png "pipeline")

### IAM Consideration for CodeBuild and CodeDeploy role
The fact that these projects can run arbitrary code should make you feel somehow at unease. If not, considering someone can wrap an AWS CLI command to `Create Admin IAM Role that’s assumed by an entity they own` and subsequently own your entire AWS account. 

You should build everything with the principle of least privilege in mind, even your almighty pipeline. Give it only all the permission it absolutely needs, and nothing more. Here’s what my CodeBuild and CodePipeline role’s permission set look like. 

<br>

***

## Conclusion
Creating a CI/CD pipeline for your Serverless project can greatly accelerate the development feedback cycle in a secure manner. You never have to worry about static IAM credentials or someone `accidentally` destroy your production services running. The repository linked pipeline also enables collaboration amongst devs to work on their separate branches and test as they build. But with great power comes great responsibility, you should keep in mind that CodeBuild itself is essentially `RCE as a service`, and IAM Permission scoping must be done in conjunction with the pipeline buildout itself. 


### Where Next?
Remove `deployment permission` from all IAM users/roles other than the CodePipeline deployment role. 

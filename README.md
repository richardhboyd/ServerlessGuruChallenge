# ServerlessGuruChallenge

## The Challenge

Build a Serverless Framework REST API with AWS API Gateway which supports CRUD functionality (Create, Read, Update, Delete) *don't use service proxy integration directly to DynamoDB from API Gateway

Please use GitHub Actions CI/CD pipeline, AWS CodePipeline, or Serverless Pro CI/CD to handle deployments.

You can take screenshots of the CI/CD setup and include them in the README.

The CI/CD should trigger a deployment based on a git push to the master branch which goes through and deploys the backend Serverless Framework REST API and any other resources e.g. DynamoDB Table(s).

### Requirements

0. All application code must be written using NodeJS, Typescript is acceptable as well

1. All AWS Infrastructure needs to be automated with IAC using [Serverless Framework](https://www.serverless.com)

2. The API Gateway REST API should store data in DynamoDB

3. There should be 4-5 lambdas that include the following CRUD functionality (Create, Read, Update, Delete) *don't use service proxy integration directly to DynamoDB from API Gateway

3. Build the CI/CD pipeline to support multi-stage deployments e.g. dev, prod

4. The template should be fully working and documented

4. A public GitHub repository must be shared with frequent commits

5. A video should be recorded (www.loom.com) of you talking over the application code, IAC, and any additional areas you want to highlight in your solution to demonstrate additional skills

Please spend only what you consider a reasonable amount of time for this.

## Approach

### Serverless App

I created a very basic CRUD app with the Serverless Framework.
The API Gateway REST API uses request model validation validation to (1) ensure that requests are well-formed and (2) provide a documented schema/SDK for consumers using API Gateway's [`get-sdk` method](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-sdk.html). The schemas for requests are stored in the `/schemas/` directory.


Example with malformed request (missing 'text' field in the request)
```shell
$ curl -X POST \
  -d '{"test":"some bad text based on model"}' \
  --header "Content-Type: application/json" \
  --url https://18u2vxay4e.execute-api.us-east-1.amazonaws.com/dev/widget 

> {"message": "Invalid request body"}
```

Normally, I would also include models for the response objects as well, but it appears that the Serverless Framework strips these out when using the Lambda Proxy integration. This means that we'd have to rely on comprehensive regression testing of the Lambda Functions to ensure we don't accidentally introduce a breaking change to the API.

### CICD

I cut quite a few corners on the CI/CD process that I will highlight here. These aren't shortcuts I'd use in a production setup, but a release pipeline could easily have 50-60 hours worth of work in setting it up following best practices for a large number of use-cases. The CloudFormation template that defines the infrastructure is located within the `/infrastructure/` folder

- The `codestar::connection` resource was manually created in the console because it relies on an OAuth flow that would be fairly tedious to automate into CloudFormation. Some type of either CloudFormation Resource Type or fancy conditional statements with CloudFormaiton Macros might be able to accomplish it, but it seemed a bit out of scope for this exercise. I did something similar for the Serverless Framework API Key that *might* work for the GH connection.

- The Serverless Framework Access Key needed for the `serverless deploy` step in the release process is stored in an AWS Secrets Manager Secret and then referenced by the CodeBuild Project as an environment variable. I use a dummy value in the CloudFormation Template then update it later with the correct key so that I don't accidentally commit the API Key to my GitHub repository.

- The CodeBuild Project that deploys the serverless application currently has administrator privileges (i.e. `*/*` permissions). Usually we'd have an IAM Role in the target account that can only be assumed by the IAM Role that CodeBuild is using in the Infrastructure account and it would only have permissions to start a CloudFormation deployment. Additionally, the CloudFormation template would be deployed using a separate IAM Role in the target account that has more broad permissions, but can only be assumed by the CloudFormation Service (i.e. the IAM Role has broad permissions, but only for managing resources via CloudFormation).  I gave a talk on how to do this at [DevOpsDays Boston in 2019](https://devopsdays.org/events/2019-boston/program/richard-boyd) and have the associated code examples [here](https://github.com/rhboyd/DevOpsDaysBoston)

- I use a single account for both the app and the CI/CD infrastructure. Typically, you'd have one AWS account for a specific application's CI/CD infrastructure, and a separate AWS account for each stage of the pipeline, plus an account for each region. i.e. the accounts used would be:
  - (1) Infrastructure 
  - (2) pre-production 
  - (3) prod - IAD (North America)
  - (4) prod - NRT (Asia Pacific)
  - (5) prod - DUB (Europe)
  - etc ...

- I only created a single pipeline that is triggered by git pushes to the `main` branch. Typically, you'd have another pipeline that is triggered by a push to other branches, such as `feat-*` if you use feature branches, and it would perform more CI-related tasks, such as regression tests, without starting a deployment. Additionally, you could have a pipeline that is triggered by pull-requests that would run basic integration tests and many of the 'cheaper' tests so that you're not spending a lot of resources on deep/expensive tests on every pull request.
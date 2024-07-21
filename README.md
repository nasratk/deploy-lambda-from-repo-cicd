# CI/CD Pipeline for Deploying a Lambda Function using AWS CodePipeline, CodeBuild, and AWS SAM

This repository contains an AWS CloudFormation template to create a CI/CD pipeline for deploying a Lambda function directly from a GitHub repository. The pipeline leverages AWS CodePipeline, CodeBuild, and AWS SAM.

## Why do you need this solution?

Automatic deployment of code changes to Lambda function is not so straightforward. While AWS CodeDeploy replaces the code hosted on EC2 instances (and other hosts) in response to new commits to the code repository, it does not replace the code hosted on Lambda function. 

Earlier, any changes to source code were deployed completely manually by packaging the source code, saving to S3 and redeploying to Lambda. With SAM, this process has been simplified considerably, but still requires manual touchpoints, requiring build and deployment commands.

This is where this solution helps. It provides a completely automated CI/CD pipeline, where any commits made to GitHub repository can lead to automatic deployment of the entire code to Lambda, without manual intervention. In doing so, it use SAM framework, but instead of manually running build and deploy SAM commands, they are run within build and deploy stages of CodePipeline respectively. So in effect, this approach is an extension of SAM framework for Lambda deployment.

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Parameters](#parameters)
- [Deployment](#deployment)
- [Outputs](#outputs)
- [License](#license)

## Architecture

The CloudFormation template creates the following AWS resources:

1. **IAM Roles**:
    - `CodeBuildServiceRole`: Role for CodeBuild with permissions to access S3, CloudWatch Logs, and other required services.
    - `CodePipelineServiceRole`: Role for CodePipeline with permissions to access S3, CodeBuild, Lambda, CloudFormation, and IAM.
    - `CloudFormationRole`: Role for CloudFormation with permissions to manage resources such as Lambda, EC2, IAM, CloudWatch Logs, S3, and CloudFormation.
    
    The template only contains a sample of permissions, but these will need to be modified depending on what your build, pipeline, or cloudformation stack will require. Permissions are created in-line, rather than as a custom policy.

2. **CodeBuild Project**:
    - Builds the Lambda function code and packages it for deployment.

3. **CodePipeline**:
    - Orchestrates the CI/CD process, with stages for Source, Build, and Deploy. 
    - 'Source' stage pulls in the source code from GitHub repository.
    - 'Build' stage runs the build using SAM build command.
    - 'Deploy' stage sets of a CloudFormation stack that will ultimately build and deploy the Code Pipeline.

## Prerequisites

Before deploying this CloudFormation stack, ensure you have the following:

1. An AWS account with the necessary permissions to create the required resources.
2. An existing GitHub repository containing your Lambda function code.
3. A GitHub Personal Access Token stored securely in AWS Systems Manager Parameter Store.
4. An S3 bucket which will be used by CodePipeline to exchange artifacts between various pipeline stages. This template will create a new prefix to be used specifically for this deployment, but the bucket itself should exist prior to running this template.
5. You have created a SAM template, named 'template.yaml' that specifies your Lambda function, your Lambda Execution Role, and any associated resources, such as API Gateway. Make sure you've placed template.yaml inside the root folder of your repository.
6. Copy 'buildspec.yaml' from this repository to the root folder of your Lambda repository.

## Parameters

The CloudFormation template requires the following parameters:

- `GitHubRepositoryOwner`: The GitHub user or organization that owns the repository.
- `GitHubRepositoryName`: The name of the GitHub repository.
- `GitHubBranch`: The branch of the GitHub repository to use (default is `main`).
- `GitHubPersonalAccessToken`: The name of the SSM parameter storing the GitHub Personal Access Token.
- `LambdaFunctionName`: Name of the deployed Lambda function.
- `CodePipelineBucketName`: Name of the S3 bucket that will host the CodePipeline artifacts.

## Deployment

1. **Create GitHub Personal Access Token**:
    - Go to your GitHub account settings.
    - Navigate to **Developer settings** > **Personal access tokens**.
    - Generate a new token with `repo` and `admin:repo_hook` scopes.
    - Store the token in AWS Systems Manager Parameter Store.

    ```sh
    aws ssm put-parameter --name "/my-app/github-token" --value "YOUR_GITHUB_TOKEN" --type "SecureString"
    ```

2. **Copy over buildspec.yaml**
    - As specified in the pre-requisites section, copy 'buildspec.yaml' from this repository to the root folder of your Lambda repository.

3. **Deploy the CloudFormation Stack**:
    - Use the AWS Management Console, AWS CLI, or AWS CloudFormation console to deploy the template.

    If you are using a parameters file to specify your parameters or to override defaults, e.g. LambdaDepCP-Pars.json, then use the following command:

    ```sh
    aws cloudformation create-stack --stack-name MyLambdaPipeline \
      --template-body file://LambdaDepCP.yaml \
      --parameters file://LambdaDepCP-Pars.json \
      --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
    ```
    If you wish to specify parameters in line, use the following command:

    ```sh
    aws cloudformation create-stack --stack-name MyLambdaPipeline \
      --template-body file://LambdaDepCP.yaml \
      --parameters ParameterKey=GitHubRepositoryOwner,ParameterValue=YOUR_GITHUB_USER \
                   ParameterKey=GitHubRepositoryName,ParameterValue=YOUR_REPO_NAME \
                   ParameterKey=GitHubBranch,ParameterValue=main \
                   ParameterKey=GitHubPersonalAccessToken,ParameterValue=/my-app/github-token \
                   ParameterKey=LambdaFunctionName,ParameterValue=MyLambdaFunction \
                   ParameterKey=CodePipelineBucketName,ParameterValue=my-codepipeline-bucket \
      --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
    ```

## Outputs

- **PipelineName**: Name of the CodePipeline pipeline.
- **LambdaFunctionName**: Name of the deployed Lambda function.
- **CodePipelineBucketPrefix**: S3 bucket for storing CodePipeline artifacts.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
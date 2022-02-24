---
layout: default
title: Deployer AWS Permissions
nav_order: 5
---
# AWS permissions required to deploy

Generally, an admin or a power user of AWS should be able to deploy
tasks using an example project or a project set up to deploy with the
[aws-ecs-cloudreactor-deployer](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer) Docker image. However, if the AWS user account
you intend to use to deploy tasks does not have sufficient permissions,
deployment won't succeed. The exact permissions needed are:

* ecr:BatchCheckLayerAvailability
* ecr:CompleteLayerUpload
* ecr:CreateRepository
* ecr:DescribeRepositories
* ecr:GetAuthorizationToken
* ecr:InitiateLayerUpload
* ecr:PutImage
* ecr:UploadLayerPart
* ecs:RegisterTaskDefinition
* iam:GetRole (to any role with name containing "taskExecutionRole")
* iam:PassRole (to any role with name containing "taskExecutionRole")

You can either add these permissions to the AWS user account that
will be used to deploy, or upload the
[CloudReactor AWS deployer CloudFormation template](https://raw.githubusercontent.com/CloudReactor/aws-role-template/master/cloudreactor-aws-deploy-role-template.json) to CloudFormation.
For instructions on how to do that, see the
main project page for [aws-role-template](https://github.com/CloudReactor/aws-role-template/).

Once you have the access key and secret key output by the template,
you can add them to `deploy_config/deploy.env` , or arrange to have the environment variables
`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` set, along with
`PASS_AWS_ACCESS_KEY=FALSE` before you run your deployment script
(`cr_deploy.sh` or a custom wrapper script that calls `cr_deploy.sh`).
If you are using the [GitHub Action](https://docs.cloudreactor.io/build_customization.html#setup-deployment-via-github-action) you
would set these variables as secrets in your GitHub account
and use them to set the `aws-access-key-id` and `aws-secret-access-key`
input parameters of the GitHub Action.

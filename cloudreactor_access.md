---
layout: default
title: Granting CloudReactor Task Managment Acesss
nav_order: 3
---
# Granting CloudReactor Task Managment Access

## Background

CloudReactor monitors and manages Tasks but the Tasks themselves run in your
own infrastructure. That way the Tasks have access to resources internal to
your infrastructure. Currently, CloudReactor can only start Tasks running in
AWS Fargate. (In the future, we plan on supporting AWS Lambda, Kubernetes,
and other execution methods.) CloudReactor needs your permission to start ECS
tasks in your AWS account, so we provide a
[CloudFormation template](https://github.com/CloudReactor/aws-role-template)
that creates a role that has the required permissions.
CloudReactor assumes the role to gain the access it needs. These permissions
are required for both the *Simple* and *Full* integration levels.

You can install the template either manually using the AWS Console, or
programmatically using a command-line wizard provided by CloudReactor.
The wizard is also able to set up the infrastructure required to run tasks
using ECS Fargate if you don't have that setup already. Alternatively, it
can use your existing infrastructure. The wizard will send your
AWS settings to create a Run Environment suitable for starting Tasks, so
you don't need to copy the settings yourself.

## Security considerations

Installing the CloudFormation template and having the wizard create
infrastucture on your behalf requires AWS permissions to create resources,
including roles, VPCs, subnets, internet gateways, NAT gateways,
VPC endpoints, and ECS clusters. To install the template manually or run
the wizard, we recommend you use a Power User or Admin account so that
these resources can be created.

Understandably, you are probably concerned with granting broad permissions
to a third-party program. To alleviate your concerns, the
[CloudFormation template](https://github.com/CloudReactor/aws-role-template/blob/master/cloudreactor-aws-role-template.json) and
[wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard) are
both source-available, so you can inspect them to ensure they are acting
responsibly. Additionally, rest assured that your AWS credentials are
never sent to any entity besides AWS.

## Running the wizard

We recommend that you install the CloudFormation template using the wizard,
as it can set up your infrastructure for ECS and reduces a lot of manual steps.
Assuming you want to run the wizard, follow the steps below:

1. If you haven't already, install Docker. From the Docker website, download
and install [Docker Desktop](https://www.docker.com/products/docker-desktop){:target="_blank"} (macOS or Windows) or [Docker Compose](https://docs.docker.com/compose/install/){:target="_blank"} (Linux). Once installed, start the Docker daemon.

2. Next, create a directory somewhere that the wizard can use to save your
settings, between runs. For example,

    `mkdir saved_state`

3. Now, run the image:

    `docker run --rm -it -v $PWD/saved_state:/usr/app/saved_state cloudreactor/aws-setup-wizard`

which will use the saved_state subdirectory of the current directory to
save settings.

4. Go through the wizard. The wizard lets you select a region, ECS cluster, VPC, subnet, etc. for you to run tasks in (and which CloudReactor will manage). And if any of those pieces don't exist, it can create them for you.
    - Questions with square brackets at the end indicate a default value: simply hit enter to accept the default. For example, `What do you want to name the CloudFormation stack to create a VPC? [ECS-VPC]` -- here, the default value is `ECS-VPC`.
    - "Which AWS region will you run ECS tasks in?": you should choose a region where other resources that you wish your tasks to access are located. For example, if you have an RDS instance in `us-west-2` that you wish to access from your tasks, be sure to choose `us-west-2` here.
    - "What is the AWS access key you want to use for this wizard?": we recommend providing AWS Administrator user credentials. This is because the wizard will be creating resources (ECS cluster, VPC, subnets, etc.). The code behind the wizard is publicly viewable on GitHub:
    [https://github.com/CloudReactor/cloudreactor-aws-setup-wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard){:target="_blank"}
    - "What do you want to name your Run Environment?": this is a name used in CloudReactor to refer to the infrastructure you're setting up. If you're setting up a cluster to run production or staging tasks, you might call this "production" or "staging" respectively for example. **Remember this name for use later.**

The wizard will create a Run Environment in CloudReactor that is used to
group Tasks together. If you use
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper]) to
run a Task in ECS Fargate, and the Task is grouped in a Run Environment
created by the wizard, you will be able to manually start the Task
in the CloudReactor dashboard or with an API call, and the Task can participate
in Workflows.

## Manual setup

If you don't want to use the Setup Wizard for security reasons, you can refer
to the [manual setup instructions](/manual_setup.md) -- but
note that setting up each piece of infrastructure manually will be
time-consuming and error-prone.

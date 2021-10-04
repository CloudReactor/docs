---
layout: default
title: Full Integration
nav_order: 4
---

# Full Integration

## Background

The *Full* integration level provides all the features that CloudReactor has to
offer. Compared to the *Simple* integration level, it has these additional
features:

* Task scheduling
* ECS service setup, including load balancer configuration
* Tasks show up in the CloudReactor dashboard before they execute

To gain these features, you need to register your Task beforehand with
the CloudReactor API. Below, we'll use the
[aws-ecs-cloudreactor-deployer](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
Docker image to deploy to ECS as well do the CloudReactor registration. (It's
also possible to make API calls to create Tasks directly if you know the ECS
task definition ARN, but we won't cover that here.)

## Prequisites

The steps to [grant CloudReactor Task management access](/cloudreactor_access.html)
are a pre-requisite for full integration.

Additionally, you should create another CloudReactor API key, this time with
the `Task` access level. Follow the instructions for
[creating an API key](/index.html#create-api-key), except select the
`Task` access level instead of the `Developer` access level in step 2.
Your Task will use this
key to report its state to CloudReactor while it is running. For security's
sake, it has more limited permissions.

## Using aws-ecs-cloudreactor-deploy to deploy your project

The easiest way to gain full integration capabilities is to use the
aws-ecs-cloudreactor-deploy Docker image to build and deploy your project.
It can deploy multiple tasks per project to both ECS and CloudReactor, so it
alleviates the need to do the setup yourself.

### Setting up deployment permissions
{: #setting-up-deployment-permissions }

aws-ecs-cloudreactor-deployer needs to be configured with AWS credentials that
allow it to deploy Docker images to AWS ECR and create tasks in ECS on your
behalf.

You can either:
1. Use access keys or a role associated with an admin user or a power user with broad permissions; or
2. Create a user and role with specific permissions for deployment.

**If you're using an admin or power user, feel free to skip to the next step.**

If you want to use a new user and role, we've prepared a
["CloudReactor AWS deployer" CloudFormation template](https://github.com/CloudReactor/aws-role-template).
Note that an admin user has to upload this template, as it creates an
additional user. When ready, log into the AWS management
console, select the CloudFormation service, and upload this template.
It will create a user with all the necessary permissions.
**When the CloudFormation template has completed deployment, it will output
an access key ID and secret, save these for use later!**).

For more details, see [AWS permissions required to deploy](/deployer_aws_permissions.md).

### Shortcut for example projects

If you code is written in python, you can fork the
[cloudreactor-python-ecs-quickstart](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart)
 project which also includes a lot of goodies for python development, like:

 * Runs, tests, and deploys everything with Docker, no local python installation
 required
* Uses [pip-tools](https://github.com/jazzband/pip-tools) to manage only
top-level python library dependencies
* Uses [pytest](https://docs.pytest.org/en/latest/) (testing),
[pylint](https://www.pylint.org/) (static code analysis),
[mypy](http://mypy-lang.org/) (static type checking), and
[safety](https://github.com/pyupio/safety) (security vulnerability checking)
for quality control
* Uses [GitHub Actions](https://github.com/features/actions) for
Continuous Integration (CI)

You can now skip to [setting Task properties](#set-task-properties).

### Existing projects

If you have an existing project you want to deploy, follow the steps below.

First, if you haven't already, Dockerize your project. The steps are
language dependent, so do a web search for "dockerize [your language]".

Next, ensure that you Docker image contains the files
necessary to run [proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper).
This could either be a standalone executable, or having python 3.6+ installed
and installing the
[cloudreactor-procwrapper PyPi package](https://pypi.org/project/cloudreactor-procwrapper/)
. Your Dockerfile should call proc_wrapper as its entrypoint. If using
a standalone Linux executable:

```
# Local copy of https://github.com/CloudReactor/cloudreactor-procwrapper/raw/v3.0.1/pyinstaller_build/platforms/linux-amd64/proc_wrapper
COPY deploy_config/files/proc_wrapper .
RUN chmod +x proc_wrapper

...

ENTRYPOINT ./proc_wrapper $TASK_COMMAND
```

If using the python package:

```
# Local copy of https://github.com/CloudReactor/cloudreactor-procwrapper/blob/v3.0.1/proc_wrapper-requirements.txt
COPY deploy_config/files/proc_wrapper-requirements.txt .
RUN pip3 install -r proc_wrapper-requirements.txt
RUN pip3 install cloudreactor-procwrapper==3.0.1

...

ENTRYPOINT python3 -m proc_wrapper $TASK_COMMAND
```

Now let's work on the build and deployment process. First copy
`cr_deploy.sh` from the
[aws-ecs-cloudreactor-deployer](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
repository into your project root directory. This is a driver script
that will run the deployer's Docker image with various options.

Afterwards, copy the `deploy_config` directory from the
aws-ecs-cloudreactor-deploy repository into your project's root directory.
We'll modify the contained files with the Task properties next.

### Set Task properties
{: #set-task-properties }

Whether you are starting with an example quickstart project or modifying
an existing project, the next step is to set properties for each
Task in each deployment environment. Each deployment environment is
associated with a Run Environment, and in most cases the mapping is one-to-one.

Common properties for all Tasks and deployment environments can
be entered in `deploy_config/vars/common.yml`.
For every deployment environment ("staging", "production") that
you have, create a file `deploy_config/vars/<environment>.yml` that
is based on `deploy_config/vars/example.yml` and add your settings there.

Both `common.yml` and `example.yml` have many properties you can uncomment
and set, such as the subnets that you want your Task running in. In most
cases, you can leave properties unset, defaulting to the properties in the
Run Environment associated with the deployment environment.
[Task Configuration for aws-ecs-cloudreactor-deploy](/configuration.md)
contains more details on how to set properties.

Two notable properties you can set in the deployment-specific file are the
Deploy API Key and the Task API Key:

    cloudreactor:
      ...
      deploy_api_key: xxx
      task_api_key: yyy

Setting these properties in plaintext is insecure, but in the interest of
expediency, you make want to do that for now. After you've evaluated if
CloudReactor suits your needs, you can secure your secrets using one of the
methods described in [secret management](/secrets.md).

### Customize the build

At a minimum, you need to inject AWS credentials and two CloudReactor API keys
into the build environment. You may have injected the CloudReactor API keys
in your `deploy_config/vars/<environment>.yml` file already, but you still
need to inject AWS credentials.

To help with variable injection,
the Ansible playbook also reads environment variables which you can set in
`deploy.env` or `deploy.<environment>.env`. Using this functionality,
an easy but insecure way to inject AWS credentials is to copy
`deploy.env.example` to `deploy.env` and
and fill in your AWS access key, access key secret, and default
region:

    # deploy.staging.env

    AWS_ACCESS_KEY_ID=XXX
    AWS_SECRET_ACCESS_KEY=XXX

    # Change to the region your ECS cluster is in.
    AWS_DEFAULT_REGION=us-east-1

The access key and secret would be for the AWS user you plan on using
to deploy with, as described in
[Deployer API Permissions](/deployer_api_permissions.html).

To use secrets or perform custom build steps (such as compilation), see
[Build Customization](/build_customization.html).

### Deploy

Finally, deploy. In a bash shell, run:

    ./cr_deploy.sh <environment> [TASK_NAMES]

or in Windows:

    .\cr_deploy.cmd <environment> [TASK_NAMES]

where `TASK_NAMES` is an optional, comma-separated list of Tasks to deploy.
If omitted, all tasks defined in `deploy_config/vars/common.yml` will be
deployed.

If you wrote a wrapper over `cr_deploy.sh`, use that instead.

## Further steps

* To avoid leaking secrets (passwords, API keys, etc.), see the guide on
[secret management](/secrets.md)
* You can also configure deployment to happen after pushing to GitHub,
see [setup deployment via GitHub Action](/build_customization.html#github-action)
* If you're having problems, see the [troubleshooting guide](/troubleshooting.md)

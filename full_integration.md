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

To gain these features, you need to register your Task beforehand with the
CloudReactor API. Below, we'll use the
[aws-ecs-cloudreactor-deploy](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
Docker image to deploy to ECS as well do the CloudReactor registration. It's
also possible to make API calls to create Tasks directly if you know the ECS
task definition.

The steps to [grant CloudReactor Task management access](/cloudreactor_access.html)
are a pre-requisite for full integration.

## Using aws-ecs-cloudreactor-deploy to deploy your project

The easiest way to gain full integration capabilities is to use the
aws-ecs-cloudreactor-deploy Docker image to build and deploy your project.
It deploys multiple tasks per project to both ECS and CloudReactor, so it
alleviates the need to do the setup yourself.

### Optional: setting up deployment permissions
{: #setting-up-deployment-permissions }

aws-ecs-cloudreactor-deploy needs to be configured with AWS credentials that
allow it to deploy Docker images to AWS ECR and create tasks in ECS on your behalf.

You can either:
1. Use access keys or a role associated with an admin user or a power user with
broad permissions; or
2. Create a user and role with specific permissions for deployment.

**If you're using an admin or power user, feel free to skip to the next step.**

If you want to use a new user and role, we've prepared a
["CloudReactor AWS deployer" CloudFormation template](https://github.com/CloudReactor/aws-role-template){:target="_blank"}.
Feel free to inspect the template. Note than admin user has to upload this
tempate, as it creates an additional user. When ready, log into the AWS management
console, select the CloudFormation service, and upload this template.
It will create a user with all the necessary permissions.
**When the CloudFormation template has completed deployment, it will output
user credentials, save these for use later!**).

For more details, see [AWS permissions required to deploy](/deployer_aws_permissions.md)

### Create CloudReactor API keys

If you followed the instructions to grant
[CloudReactor Task management access](/cloudreactor_access.html),
you've already created an API key with the `Developer` access level. Otherwise,
do that now.

Next, create another API key that your Task will use to communicate with
CloudReactor while it's running. Give the API key a name and associate it with
the Run Environment you created. Ensure the Group is correct, the Enabled
checkbox is checked, and the Access Level is `Task`. Then select the Save
button. You should then see your new API key listed. Copy the value of the key.
This is the `Task API key`.

### Shortcut for python projects

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

First, if you haven't already, Dockerize your project. The steps are very
language dependent, so do a web search for "dockerize [your language]".

If you haven't already, ensure that you Docker image contains the files
necessary to run [proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper).
This could either be a standalone executable, or having python 3.6+ installed
and installing the
[cloudreactor-procwrapper](https://pypi.org/project/cloudreactor-procwrapper/)
package. Your Dockerfile should call proc_wrapper as its entrypoint. If using
a standalone Linux executable:

```
ENTRYPOINT ./proc_wrapper $TASK_COMMAND
```

If using the python package:

```
ENTRYPOINT python -m proc_wrapper $TASK_COMMAND
```

Now let's work on the build and deployment process. If you need to add custom
build steps, such as compilation, see
[Build Customization](/build_customization.html).

Following that, copy and optionally modify `deploy.sh`
(or `deploy.cmd` and `docker-compose-deploy.yml` if you are working on a
Windows machine) from the
[aws-ecs-cloudreactor-deploy](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
repository into your project root directory.



You will need to remap your files your project
into the Docker build context like this:

      -v $PWD/Dockerfile:/work/docker_context/Dockerfile
      -v $PWD/src:/work/docker_context/src

Then your Dockerfile will see the contents of your `src` directory in `src`.

Afterwards, copy the `deploy_config` directory from the
aws-ecs-cloudreactor-deploy repository into your project's root directory;
we'll modify the contained files with the Task properties next.

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

### Set deployment secrets

Copy `deploy.env.example` to `deploy.env` and
and fill in your AWS access key, access key secret, and default
region. The access key and secret would be for the AWS user you plan on using
to deploy with,
possibly created in the section "Select or create user and/or role for deployment".
You may also populate this file with a script you write yourself,
for example with something that uses the AWS CLI to assume a role and gets
temporary credentials. If you are running this on an EC2 instance with an
instance profile that has deployment permissions, you can leave this file blank.

### Deploy

Finally, deploy. In a bash shell, run:

    ./deploy.sh <environment> [TASK_NAMES]

or in Windows:

    .\deploy.cmd <environment> [TASK_NAMES]

where `TASK_NAMES` is an optional, comma-separated list of Tasks to deploy.
If omitted, all tasks defined in `./deploy/vars/common.yml` will be deployed.

## More information

* To avoid leaking secrets (passwords, API keys, etc.), see the guide on
[secret management](/secrets.md)
* If you're having problems, see the [troubleshooting guide](/troubleshooting.md)

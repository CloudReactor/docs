---
layout: default
title: Home
nav_order: 1
---
# Easily create, deploy, orchestrate and manage tasks in AWS
{: .no_toc }
{: .fs-10 }

CloudReactor provides Dockerfiles and scripts that enable you to get up and running with a local Python development environment, deploy code seamlessly to your AWS environment, and monitor, manage and orchestrate deployed tasks with CloudReactor -- all in record time and with a minimum of fuss.
{: .fs-4 .fw-300 }

[Get started](#getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 } [Learn more about CloudReactor](./cloudreactor.html){: .btn .fs-5 .mb-4 .mb-md-0 }

## Table of contents
{: .no_toc .text-delta .mt-6 }

1. TOC
{:toc}

---

## Getting Started


### Set up AWS infrastructure, link to CloudReactor

CloudReactor requires 3 things to be setup:
1. Infrastructure to run tasks in your AWS environment ECS cluster, VPC, etc.
2. A role in AWS that allows CloudReactor to schedule and manage tasks that you deploy
3. Letting CloudReactor know what that role and other AWS settings is

You might already have some of this set up (e.g. an ECS cluster, or a VPC) -- or you might not. Either way, we've created a [super easy AWS Setup Wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard) that can ensure you have everything you need. It takes < 15 minutes.

Because the AWS Setup Wizard will be setting up ECS clusters, VPCs, subnets etc., you'll need Administrator user privileges to run it. The code behind the wizard can be inspected at the above link.

If you don't want to use the Setup Wizard for some reason, you can refer to the [manual setup instructions](docs/manual_setup.md).

--- 

### Note: setting up an AWS user with deployment permissions
{: .no_toc}
Note that below, we'll be using AWS user credentials to deploy tasks to AWS ECS.

The AWS user credentials we use must therefore have permissions to deploy Docker images to ECR and to create tasks in ECS. You can either:
1. Use an admin user or a power user with broad permissions; or
2. Create a user and role with specific permissions for deployment.

If you're using an admin or power user, feel free to skip to the next step.

If you want to create a new user and role, we've prepared a ["CloudReactor AWS deployer" CloudFormation template](https://github.com/CloudReactor/aws-role-template). In AWS, upload this to CloudFormation, and it will create a user with all the necessary permissions.

For more details, see [AWS permissions required to deploy](docs/deployer_aws_permissions.md).
{: .mb-6}

---

### Deploy example tasks to AWS
With the underlying infrastructure set-up, we can now go ahead and deploy tasks to AWS and have them managed by CloudReactor.

We do this by providing a "quickstart" repo. The repo contains simple toy tasks, as well as scripts that enable easy deployment to AWS. You can replace these toy tasks with your own scripts.

First, fork [this repo](https://github.com/CloudReactor/cloudreactor-ecs-quickstart.git). Then, once forked, clone it

```
git clone [https://github.com/link to the forked repo]
```

This repo contains a Dockerfile (container) that has all the dependencies (python, ansible, aws-cli etc.) required to build and deploy your tasks. Our next step is to build this "local" container, and then use it to deploy tasks from your local machine. This is the most straightforward way to configure and deploy, since:
- you don't need to have python installed directly on your machine
- you don't need to add another set of dependencies to your libraries
- you can deploy irrespective of your OS (e.g. if you're running Windows).

Note: You can also use this method on an EC2 instance that has an instance profile containing a role that has permissions to create ECS tasks. When deploying, the AWS CLI in the container will use the temporary access key associated with the role assigned to the EC2 instance.

However, if you want to deploy natively -- perhaps you have Python installed (possibly in a VM), and you want to use Python directly to deploy -- see [this section](#).

Otherwise, let's continue:

1. Install [Docker Compose](https://docs.docker.com/compose/install/) and run (if running Windows or Mac, Docker Desktop includes Docker Compose)
2. **AWS configuration:** Copy `deploy/docker_deploy.env.example` to `deploy/docker_deploy.env`
    - Fill in your `AWS access key`, `access key secret`, and `default region`. The AWS keys used here must be for a user with privileges to deploy tasks to AWS ECS, as mentioned above.
    - The access key and secret should be for the AWS user you plan on using to deploy with, possibly created above in [Prerequisites: AWS user with deployment permissions](#prerequisites-aws-user-with-deployment-permissions).
    - You may also populate this file with a script you write yourself, for example with something that uses the AWS CLI to assume a role and gets temporary credentials.
    - If you are running this on an EC2 instance with an instance profile that has deployment permissions, you can leave this file blank.
3. **CloudReactor configuration:** Copy `deploy/vars/example.yml` to `deploy/vars/<environment>.yml`, where `<environment>` is the name of the Run Environment created in CloudReactor above (e.g. `staging`, `production`)
    - Open the .yml file you just created, and enter your CloudReactor API key next to `api_key`; or, if you're not using CloudReactor, set `enabled: false` instead
4. Build the Docker container that will deploy the project.
    - In a bash shell, run:
    ```
    ./docker_build_deployer.sh <environment>
    ```

    - In a Windows command prompt, run:
    ```
    docker_build_deployer <environment>
    ```
    `<environment>` is a required argument, which is the name of the Run Environment and .yml file created immediately above.
    
    This step is only necessary once, unless you add additional configuration to `deploy/Dockerfile`.
5. With the deployment container created, we can deploy the tasks
    - In a bash shell, run:
    ```
    ./docker_deploy.sh <environment> [task_name]
    ```
    - In a Windows command prompt, run:
    ```
    docker_deploy <environment>  [task_names]
    ```
    In both of these commands, `<environment>` is a required argument, which is the name of the Run Environment. `[task_names]` is an optional argument, which is a comma-separated list of tasks to be deployed. In this project, this can be one or more of `task_1`, `file_io`, etc, separated by commas. If `[task_names]` is omitted, all tasks will be deployed.
    
    To troubleshoot deployment issues:
    - In a bash shell, run:
    ```
    ./docker_deploy_shell.sh <environment>
    ```
    - In a Windows command prompt, run:   
    ```
    docker_deploy_shell.bat <environment>
    ```
    These commands will take you to a bash shell inside the deployer Docker container where you can re-run the deployment script with `./deploy.sh` and inspect the files it produces in the `build/` directory.

---

## The example tasks

Successfully deploying this example project will push two ECS tasks to AWS. You can log into [CloudReactor](https://dash.cloudreactor.io) to see these tasks.

The code for each tasks is in the `./src` folder, i.e. `./src/task_1.py` and `./src/file_io.py`. Feel free to take a look.

Next, open `./deploy/vars/common.yml` -- you'll see entries for both `task_1` and `file_io`. You can think of this as a manifest of tasks to push to ECS; the CloudReactor deployment script you just ran will look for the files defined here, push them to ECS, and register them with CloudReactor.

These tasks have the following behavior:
* *task_1* prints 30 numbers and exits successfully. While it does so, it updates the "successful" count and the "last status message" that is shown in CloudReactor, using the CloudReactor status updater library. It is configured to run daily via `deploy/vars/common.yml`
* *file_io* uses [non-persistent file storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html) to write and read numbers
* *web_server* uses a python library dependency (Flask) to implement a web server and shows how to link an AWS Application Load Balancer (ALB) to a service. It requires that an ALB and target group be setup already, so it is not enabled by default (i.e. is commented out in the `./deploy/vars/common.yml` file).

---

## Development workflow

### Running the tasks locally

The tasks are setup to be run with Docker Compose in `docker-compose.yml`. For example, you can build the Docker image that runs the tasks by typing:

    docker-compose build

(You only need to run this again when you change the dependencies required by the project.)

Then to run, say `task_1`, type:

    docker-compose run --rm task_1

Docker Compose is setup so that changes in the environment file `deploy/files/.env.dev`
and the files in `src` will be available without rebuilding the image.

---


### More development options

See the [development guide](docs/development.md) for instructions on how to debug, 
add dependencies, and run tests and checks.

---


## Deploying your own tasks

### Adding new tasks
Now that you have deployed the example tasks, you can move your existing code to this project. To add your own task:
1. Place task code itself in a new file in `./src`, e.g. `new_task.py`
2. Add a configuration block for the task in `deploy/vars/common.yml`, below `task_name_to_config:` A minimal configuration block is:
    <div class="code-example" markdown="1">
    ```python
    new_task:
        <<: *default_task_config
        command: "python src/new_task.py"
    ```
    </div>
    - `<<: *default_task_config` allows new_task to inherit properties from the default task configuration
    - `command: "python src/new_task.py"` contains the command to run (in this case, to execute new_task via python)
    - Additional parameters include the run schedule (cron expression), retry parameters, and environment variables. See [additional configuration](docs/configuration.md).

### Removing tasks
You can delete any tasks you don't need (e.g. the example `task_1` and `file_io` tasks), just by removing the top level keys and associated configuration blocks below `task_name_to_config`. These tasks won't 

---

## Next steps

* [Additional configuration](docs/configuration.md) options can be set or overridden
* If you want to be alerted when task executions fail, setup an [Alert Method](docs/alerts.md)
* To avoid leaking secrets (passwords, API keys, etc.), see the guide on [secret management](docs/secret_management.md)
* For more secure [networking](docs/networking.md), run your tasks on private subnets and/or tighten your security groups.
* If you're having problems, see the [troubleshooting guide](docs/troubleshooting.md)


---

## Contact us

Hopefully, this example project has helped you get up and running with ECS and
CloudReactor. Feel free to reach out to us at support@cloudreactor.io to setup 
an account, or if you have any questions or issues!
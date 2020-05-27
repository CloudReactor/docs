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
Broadly speaking, we'll need to:
- [Set up a cluster in AWS ECS](#set-up-a-cluster-in-aws-ecs) (this is where your tasks will be deployed to)
- [(Optional) Set up AWS role permissions to allow CloudReactor to stop, start and schedule ECS tasks](#optional-set-aws-role-permissions-to-allow-cloudreactor-to-stop-start-and-schedule-ecs-tasks)
- [(Optional) Configure CloudReactor with AWS ECS and AWS role settings](#optional-configure-cloudreactor-with-aws-ecs-and-aws-role-settings)
- [Clone & configure the CloudReactor quickstart repo; deploy tasks!](#clone--configure-the-cloudreactor-quickstart-repo-deploy-tasks)

Configuration will require some keys and other parameters to be entered. We'll note anything you need to record in  <span style="color: red">red</span> -- open up a text file to hold those variables as we go along.

### Prerequisites: AWS user with deployment permissions
{: .no_toc}
Note that the AWS user credentials you use to deploy to AWS ECS must have permissions to deploy Docker images to ECR and to create tasks in ECS. You can either:
1) Use an admin user or a power user with broad permissions; or
2) Create a user and role with specific permissions for deployment; the [CloudReactor AWS deployer CloudFormation template](https://github.com/CloudReactor/aws-role-template) can help you create a user with the necessary permissions.

For more details, see [AWS permissions required to deploy](doc/deployer_aws_permissions.md).
{: .mb-6}


---

### Set up a cluster in AWS ECS
AWS provides a wizard that creates an ECS cluster in just a few steps. This is appropriate if you want to get started quickly. The wizard can optionally create a new VPC and new public subnets on that VPN -- but cannot create private subnets.

To run the wizard, your account needs to have the permissions listed under ["Amazon ECS First Run Wizard Permissions" here](https://docs.aws.amazon.com/AmazonECS/latest/userguide/security_iam_id-based-policy-examples.html#first-run-permissions).

The steps to run the wizard are:

1. Go to https://aws.amazon.com/ecs/getting-started/
2. Click the `ECS console walkthrough` button (log into AWS if necessary)
3. Change the region to your default AWS region
4. Click the `Get started` button
5. Choose the `nginx` container image and click the `Next` button
6. On the next page, the defaults are sufficient, so hit `Next` again
7. On the next page, name your cluster the desired name of your deployment environment -- for example `staging`. If you have an existing VPC and subnets you want to use to run your tasks, you can select them here. Otherwise, the console will create a new VPC and subnets for you.
After entering your desired cluster name, hit `Next` again.
8. On the next and final page, review your settings and hit the `Create` button. You'll see the status of the created resources on the next page. <span style="color: red">If you didn't choose existing subnets, record the subnet IDs</span>

AWS will create:
- A cluster named as you chose on step 7 above.
- A VPC named `ECS [cluster name] - VPC`
- 2 subnets in the VPC named `ECS [cluster name] - Public Subnet 1` and `ECS [cluster name] - Public Subnet 2`.
You can see these in VPC .. Subnets. Note that these subnets will be public, unless you selected existing (private) subnets above. If you haven't already, <span style="color: red">record the Subnet IDs</span>
- A security group named `ECS staging - ECS Security Group` in the VPC. You can find it in `VPC .. Security Groups`. <span style="color: red">Record the Security Group ID</span>
- Once you've recorded the Subnet IDs and Security Group IDs, under "ECS resource creation", you'll see `Cluster [the name of the cluster you created]`. Clicking this link will take you to the cluster's details page; <span style="color: red">record the `Cluster ARN`</span> you see here.


{: .mt-5}
- [x] ECS cluster created!

---

### (Optional) Set AWS role permissions to allow CloudReactor to stop, start and schedule ECS tasks
To have CloudReactor manage your tasks in your AWS environment, you'll need to give CloudReactor permissions in AWS to run tasks, schedule tasks, create services, and trigger Workflows by deploying the [CloudReactor AWS CloudFormation template](https://github.com/CloudReactor/aws-role-template), named `cloudreactor-aws-role-template.json`.

Follow the instructions in the [README.md](https://github.com/CloudReactor/aws-role-template/blob/master/README.md#allowing-cloudreactor-to-manage-your-tasks), in the section "Allowing CloudReactor to manage your tasks".

<span style="color: red">Be sure to record the ```ExternalID```, ```CloudreactorRoleARN```, ```TaskExecutionRoleARN```, ```WorkflowStarterARN```, and ```WorkflowStarterAccessKey``` values.</span>


{: .mt-5}
- [x] AWS role permissions created for CloudReactor!


---

### (Optional) Configure CloudReactor with AWS ECS and AWS role settings
Contact us at support@cloudreactor.io and we'll create an account for you and give you an API key.

Then login to the [CloudReactor dashboard](https://dash.cloudreactor.io/). We'll create a Run Environment in CloudReactor; a Run Environment contains settings that tell CloudReactor how to run tasks in AWS.

1. Click on "Run Environments", then "Add Environment"
2. Name your environment (e.g. "staging", "production"). You may want to keep the name in all lowercase letters without spaces or symbols besides "-" and "_", so that filenames and command-lines you'll use later will be sane. <span style="color: red">Note the exact name of your Run Environment</span>, as you'll need this later.
3. Fill in your AWS account ID and default region. Your AWS account ID is a 12-digit number that you can find by clicking "Support" then "Support Center". For default region, select the region that you want CloudReactor to run tasks / workflows in (e.g.`us-west-2`).
4. For `Assumable Role ARN` fill in the value of `CloudreactorRoleARN` from the output of the CloudFormation stack.
5. For `External ID`, use the same External ID you entered when you created the CloudFormation stack.
6. For `Workflow Starter Lambda ARN`, fill in the value of `WorkflowStarterARN` from the output of the CloudFormation stack.
7. For `Workflow Starter Access Key`, fill in the value of `WorkflowStarterAccessKey` from the output of the CloudFormation stack.
8. Add the subnets and security group created by the ECS getting started wizard above
9. Under AWS ECS Settings, choose a `Default Launch Type` of `Fargate` and check FARGATE under Supported Launch Types.
10. For `Default Cluster ARN`, fill in the `Cluster ARN` of the ECS cluster you created above
11. For `Default Execution Role` and `Default Task Role`, fill in the value of `TaskExecutionRoleARN` from the output of the CloudFormation stack.
12. Click on the `Save` button


{: .mt-5}
- [x] CloudReactor configured to run tasks in ECS!

---

### Clone & configure the CloudReactor quickstart repo; deploy example tasks!
This repo contains everything you need to deploy tasks immediately to AWS, and have these tasks orchestrated, monitored and managed by CloudReactor. The repo contains simple toy tasks that are pushed to ECS below.

First, you'll need to get this project's source code onto a filesystem where you can make changes. You can either clone this project directly, or fork it first, then clone it.

If cloning directly:

```
git clone https://github.com/CloudReactor/cloudreactor-ecs-quickstart.git
```

This repo contains a Dockerfile (container) that has all the dependencies (python, ansible, aws-cli etc.) required to build and deploy your tasks. We will build this container, and use it to deploy tasks from your local machine. This is the most straightforward way to configure and deploy, since:
- you don't need to have python installed directly on your machine
- you don't need to add another set of dependencies to your libraries
- you can deploy irrespective of your OS (e.g. if you're running Windows).

You can also use this method on an EC2 instance that has an instance profile containing a role that has permissions to create ECS tasks. When deploying, the AWS CLI in the container will use the temporary access key associated with the role assigned to the EC2 instance.

We'll assume this is satisfactory. However, if you want to deploy natively -- perhaps you have Python installed (possibly in a VM), and you want to use Python directly to deploy -- see [this section](#).

1. Ensure you have Docker running locally, and have installed [Docker Compose](https://docs.docker.com/compose/install/).
2. **AWS configuration:** Copy `deploy/docker_deploy.env.example` to `deploy/docker_deploy.env`
    - fill in your `AWS access key`, `access key secret`, and `default region`
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

{: .mt-5}
- [x] Local repo configured with AWS and CloudReactor settings; tasks pushed to AWS ECS!

---

## The example tasks

Successfully deploying this example project will create two ECS tasks. These tasks are defined in the `./src` folder, and are `task_1.py` and `file_io.py`. These tasks are configured in `deploy/vars/common.yml`.

These tasks have the following behavior:
* *task_1* prints 30 numbers and exits successfully. While it does so, it updates the "successful" count and the "last status message" that is shown in CloudReactor, using the CloudReactor status updater library. It is configured to run daily via `deploy/vars/common.yml`
* *file_io* uses [non-persistent file storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html) to write and read numbers
* *web_server* uses a python library dependency (Flask) to implement a web server and shows how to link an AWS Application Load Balancer (ALB) to a service. It requires that an ALB and target group be setup already, so it is not enabled by default.

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
* For more secure networking, run your tasks on a [private subnet](docs/networking.md)
* If you're having problems, see the [troubleshooting guide](docs/troubleshooting.md)


---

## Contact us

Hopefully, this example project has helped you get up and running with ECS and
CloudReactor. Feel free to reach out to us at support@cloudreactor.io to setup 
an account, or if you have any questions or issues!
---
layout: default
title: Home
nav_order: 1
---
# Easy workflow automation & monitoring
{: .no_toc }
{: .fs-10 }

Deploy tasks with a single command to AWS ECS. Monitor, manage and orchestrate tasks with CloudReactor's easy-to-use dashboard.

Use this one-page guide to learn everything you need to get up and running with key CloudReactor features. See additional pages in left nav bar for advanced topics.
{: .fs-4 .fw-300 }

[Get started](#getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 } [Learn more about CloudReactor](/cloudreactor.html){:target="_blank"}{: .btn .fs-5 .mb-4 .mb-md-0 }

## Table of contents
{: .no_toc .text-delta .mt-6 }

1. TOC
{:toc}

---

## Getting Started

### Set up AWS infrastructure, link to CloudReactor

First, we have to select or setup the serverless AWS infrastructure where your tasks will run, and link that infrastructure with CloudReactor.

1. From the Docker website, download and install [Docker Desktop](https://www.docker.com/products/docker-desktop){:target="_blank"} (macOS or Windows) or [Docker Compose](https://docs.docker.com/compose/install/){:target="_blank"} (Linux). Once installed, start Docker.

2. In a temporary directory, clone our [AWS Setup Wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard){:target="_blank"}:
```
git clone https://github.com/CloudReactor/cloudreactor-aws-setup-wizard.git
```

3. cd into the cloned repo, run `./build.sh` to build the wizard and then `.wizard.sh` to run:
```
cd cloudreactor-aws-setup-wizard
./build.sh
./wizard.sh (or .\wizard.bat if using Windows)
```

4. Go through the wizard. The wizard lets you select a region, ECS cluster, VPC, subnet, etc. for you to run tasks in (and which CloudReactor will manage). And if any of those pieces don't exist, it can create them for you.
    - Questions with square brackets at the end indicate a default value: simply hit enter to accept the default. For example, `What do you want to name the CloudFormation stack to create a VPC? [ECS-VPC]` -- here, the default value is `ECS-VPC`.
    - "Which AWS region will you run ECS tasks in?": you should choose a region where other resources that you wish your tasks to access are located. For example, if you have an RDS instance in `us-west-2` that you wish to access from your tasks, be sure to choose `us-west-2` here.
    - "What is the AWS access key do you want to use for this wizard?": we strongly recommend providing AWS Administrator user credentials. This is because the wizard will be creating resources (ECS cluster, VPC, subnets, etc.). The code behind the wizard is publicly viewable on GitHub: [https://github.com/CloudReactor/cloudreactor-aws-setup-wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard){:target="_blank"}
    - "What do you want to name your Run Environment?": this is a name used in CloudReactor to refer to the infrastructure you're setting up. If you're setting up a cluster to run production or staging tasks, you might call this "production" or "staging" respectively for example. **Remember this name for use later.**

If you don't want to use the Setup Wizard for some reason, you can refer to the [manual setup instructions](/manual_setup.md){:target="_blank"} -- but note that setting up each piece of infrastructure manually will be time-consuming and error-prone.

**Either run the wizard, or complete manual setup, before moving to the next step**.

---

### Optional: setting up a new AWS user with deployment permissions
CloudReactor needs to be configured with AWS credentials that allow it to deploy Docker images to AWS ECR and create tasks in ECS on your behalf.

You can either:
1. Use an admin user or a power user with broad permissions; or
2. Create a user and role with specific permissions for deployment.

**If you're using an admin or power user, feel free to skip to the next step.**

If you want to use a new user and role, we've prepared a ["CloudReactor AWS deployer" CloudFormation template](https://github.com/CloudReactor/aws-role-template){:target="_blank"}. Feel free to inspect the template. When ready, log into the AWS management console, select the CloudFormation service, and upload this template. It will create a user with all the necessary permissions. **When CloudFormation outputs user credentials, save these for use later!**).

For more details, see [AWS permissions required to deploy](/deployer_aws_permissions.md){:target="_blank"}.
{: .mb-6}

---

### Deploy example tasks to AWS
With the underlying infrastructure set-up, we can now go ahead and deploy tasks to AWS and have them managed by CloudReactor.

We do this by providing a "quickstart" repo. The repo contains the command-line tools that enable easy deployment to AWS, and simple toy tasks that illustrate features of CloudReactor.

1. Fork or clone [the quickstart repo](https://github.com/CloudReactor/cloudreactor-ecs-quickstart.git){:target="_blank"}. Then, once forked, clone it locally. (if you don't use GitHub, feel free to just clone; you just may want to check in every now and then see if there is an updated repo available):
```
git clone [https://github.com/link_to_your_forked_repo]
```

2. In your favorite code editor, copy `deploy/docker_deploy.env.example` to `deploy/docker_deploy.env`. Open this file and fill in your `AWS access key`, `access key secret`, and `default region`.
    - **The AWS keys used here must be for a user with privileges to deploy tasks to AWS ECS.** As mentioned [above](#optional-setting-up-a-new-aws-user-with-deployment-permissions), you could use an Admin or power user, or create a new user specifically for deployment via CloudReactor.
    - You may also populate this file with a script you write yourself, for example with something that uses the AWS CLI to assume a role and gets temporary credentials.
    - If you are running this on an EC2 instance with an instance profile that has deployment permissions, you can leave this file blank.

3. Copy `deploy/vars/example.yml` to `deploy/vars/<environment>.yml`, where `<environment>` is the name of the Run Environment created during the Wizard [earlier](#set-up-aws-infrastructure-link-to-cloudreactor) (e.g. `staging`, `production`)
    - Open the .yml file you just created, and enter your **CloudReactor API key** next to `deploy_api_key`
    - This allows the task to be registered with CloudReactor when you deploy the task from your local machine
    - If you're not using CloudReactor (i.e. you just want to use our tools to deploy to AWS, but don't want to manage tasks with CloudReactor): set `enabled: false` instead

4. After the task is deployed, when it actually runs, it also needs to know the CloudReactor API key in order to communicate with CloudReactor (e.g. status updates, start/stop, etc.). As a security best practice, we will add the CloudReactor API key to AWS Secrets Manager, and allow your task to read that secret during runtime. Note that you can follow a similar process for adding your own runtime secrets (e.g. database credentials) to tasks.
    - In the AWS Console, go to Secrets Manager, and "Store a new secret". Select "other types of secrets (e.g. API key)". Select "plaintext", and paste in your CloudReactor API key as the entire field (i.e. no need for newline, braces, quotes etc.). On the next page, for "secret name", type `CloudReactor/<cloudreactor_runenvironment_name>/common/cloudreactor_api_key`. **Replace `<cloudreactor_runenvironment_name>` with whatever your CloudReactor run environment (and .yml file above) is called**. You can also choose a different name to cloudreactor_api_key if you want to. Create the secret. Then, select the secret in the Secrets dashboard, and **copy the full "secret ARN"**; it'll look something like this:
    ```
    arn:aws:secretsmanager:<region>:<account_id>:secret:CloudReactor/prod/common/cloudreactor_api_key-xxx123
    ```
    - Note that by default, the CloudReactor AWS Setup Wizard that you probably ran earlier allows tasks that are deployed with CloudReactor to read secrets that are stored under `CloudReactor/<cloudreactor_runenvironment_name/common/`.
    - In your code editor, open the .yml file from the previous step (e.g. `/deploy/vars/staging.yml`). You will find the following code block. **Replace valueFrom with your CloudReactor secret key ARN**:

    ```
        secrets: &default_env_secrets
        - name: PROC_WRAPPER_API_KEY
          valueFrom: "arn:aws:secretsmanager:<region>:<account_id>:secret:CloudReactor/prod/common/cloudreactor_api_key-xxx123"
    ```
    - Finally, we need to ensure that your task inherits the ability to read this secret. In that same .yml file, scroll down to the section starting:
    ```
    task_name_to_env_config:
          task_1:
            <<: *default_env_task_config
    ```
    As written, this enables `task_1` to read the "secrets" block above (which includes e.g. `PROC_WRAPPER_API_KEY`). If you want to enable other tasks to read these secrets, you would simply add a new block for that task; for example:
    ```
    task_name_to_env_config:
          # add the two lines below
          new_task:
            <<: *default_env_task_config
    ```
    - See the page on [secrets](/secrets.md) for further information on how to add your own runtime secrets.

5. Ensure Docker is running (you should have installed it [above](#set-up-aws-infrastructure-link-to-cloudreactor)). Deployment with Docker is highly recommended because:
    - you don't need to have python installed directly on your machine
    - you don't need to add another set of dependencies to your libraries
    - you can deploy irrespective of your OS (e.g. if you're running Windows).

    If you don't want to use Docker, you'll need to set up your local environment to deploy natively. This requires installing python and various dependencies. See [this section](#){:target="_blank"} for details, and skip steps 5 & 6 below.

6. Build the Docker container that will deploy the project. This may take a few minutes to complete so be ready to make some tea / coffee while you wait.

    Replace `<environment>` below with the name of the Run Environment and .yml file created immediately above (e.g. "staging", "production").

    - In a bash shell, run:
    ```
    ./docker_build_deployer.sh <environment>
    ```

    - In a Windows command prompt, run:
    ```
    docker_build_deployer <environment>
    ```

    Running `docker_build_deployer` is only necessary once, unless you add additional configuration to `deploy/Dockerfile`.

7. Deploy the example tasks to AWS.

    This will push two toy tasks to AWS and register them with CloudReactor. Successfully pushing these tasks to AWS confirms that setup is complete. The tasks can be easily deleted later.

    The code for each tasks is in the `./src` folder, i.e. `./src/task_1.py` and `./src/file_io.py`. Feel free to take a look; they're simple toy tasks that illustrate some features of CloudReactor / ECS.
    - *task_1* prints 30 numbers and exits successfully. While it does so, it updates the "successful" count and the "last status message" that is shown in CloudReactor, using the CloudReactor status updater library.
    - *file_io* uses [non-persistent file storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html){:target="_blank"} to write and read numbers.

    Again, replace `<environment>` below with the name of the Run Environment you're deploying to. `[task_names]` is an **optional** argument, which is a comma-separated list of tasks to be deployed e.g. `task_1, file_io`. If `[task_names]` is omitted, all tasks will be deployed.

    - In a bash shell, run:
    ```
    ./docker_deploy.sh <environment> [task_name]
    ```
    - In a Windows command prompt, run:
    ```
    docker_deploy <environment> [task_names]
    ```

    If you encounter issues, see the [troubleshooting guide](/troubleshooting.md){:target="_blank"}.

8. Log into [dash.cloudreactor.io](https://dash.cloudreactor.io){:target="_blank"}. You should see the two example tasks listed there. You can manually start and stop these tasks, or schedule them to run. They'll run on the AWS ECS infrastructure we created earlier. You can confirm this by starting a task, then navigating to the ECS cluster in the AWS console (i.e. AWS console > ECS > select cluster). Under the "tasks" tab, you'll see a running task that corresponds to the task just started via CloudReactor.

---


### Deploy tasks

When ready to deploy your tasks to AWS, as before:

- In a bash shell, run:

    ```
    ./docker_deploy.sh <environment> [task_name]
    ```

- In a Windows command prompt, run:
    ```
    docker_deploy <environment> [task_names]
    ```

`task_names` is optional. If omitted, all tasks defined in `./deploy/vars/common.yml` will be pushed.


### More development options

See the [development guide](/development.md) for instructions on how to debug, add dependencies, and run tests and checks.

---



---

## Removing tasks
Delete tasks within the [CloudReactor dashboard](https://dash.cloudreactor.io){:target="_blank"}. This will remove the task from AWS also.

You should also remove the reference to the tasks in `./deploy/vars/common.yml`.
- If you don't, if you run `./docker_deploy.sh [environment]` (without task names), this will (re-)push all tasks -- which might include tasks you had intended to remove.

You may also want to remove the task code itself from `/src/`

For example, if you want to delete the `task_1` task:
1. In [dash.cloudreactor.io](https://dash.cloudreactor.io){:target="_blank"}, hit the delete icon next to `task_1` and hit "confirm".
2. Open `./deploy/vars/common.yml` and delete the entire `task_1:` code block i.e.:

    ```python
    task_1:
    <<: *default_task_config
    description: "This description shows up in CloudReactor dashboard"
    command: "python src/task_1.py"
    schedule: cron(9 15 * * ? *)
    wrapper:
        <<: *default_task_wrapper
        enable_status_updates: true
    ```
3. Optionally, delete `/src/task_1.py`.

---

## More information

* [Additional configuration](/configuration.md) options can be set or overridden
* If you want to be alerted when task executions fail, setup an [Alert Method](/alerts.md)
* To avoid leaking secrets (passwords, API keys, etc.), see the guide on [secret management](/secrets.md)
* For more secure [networking](/networking.md), run your tasks on private subnets and/or tighten your security groups.
* If you're having problems, see the [troubleshooting guide](/troubleshooting.md)


---

## Contact us

Hopefully, the example projects have helped you get up and running with ECS and
CloudReactor. Feel free to reach out to us at support@cloudreactor.io to setup
an account, or if you have any questions or issues!
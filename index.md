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

4. Go through the wizard. The wizard will ask "What do you want to name your Run Environment?" -- this is a name used in CloudReactor to refer to the infrastructure you're setting up. If you're setting up a cluster to run production or staging tasks, you might call this "production" or "staging" respectively for example. **Remember this name for use later.**

The AWS Setup Wizard will let you select a region, ECS cluster, VPC, subnet, etc. for CloudReactor to use. If any of those pieces don't exist, it can create them for you.

Because the wizard automates the creation of resources in AWS, it will ask for your AWS user credentials. <strong>We strongly recommend providing it with an AWS Administrator user</strong>. The code behind the wizard is publicly viewable on GitHub: [https://github.com/CloudReactor/cloudreactor-aws-setup-wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard){:target="_blank"}.

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

1. Fork [the quickstart repo](https://github.com/CloudReactor/cloudreactor-ecs-quickstart.git){:target="_blank"}. Then, once forked, clone it locally. (if you don't use GitHub, feel free to just clone; you just may want to check in every now and then see if there is an updated repo available):
```
git clone [https://github.com/link_to_your_forked_repo]
```

2. In your favorite code editor, copy `deploy/docker_deploy.env.example` to `deploy/docker_deploy.env`. Open this file and fill in your `AWS access key`, `access key secret`, and `default region`.
    - **The AWS keys used here must be for a user with privileges to deploy tasks to AWS ECS.** As mentioned [above](#note-setting-up-an-aws-user-with-deployment-permissions), you could use an Admin or power user, or create a new user specifically for deployment via CloudReactor.
    - You may also populate this file with a script you write yourself, for example with something that uses the AWS CLI to assume a role and gets temporary credentials.
    - If you are running this on an EC2 instance with an instance profile that has deployment permissions, you can leave this file blank.

3. Copy `deploy/vars/example.yml` to `deploy/vars/<environment>.yml`, where `<environment>` is the name of the Run Environment created during the Wizard [earlier](#set-up-aws-infrastructure-link-to-cloudreactor) (e.g. `staging`, `production`)
    - Open the .yml file you just created, and enter your **CloudReactor API key** next to `api_key`
    - If you're not using CloudReactor (i.e. you just want to use our tools to deploy to AWS, but don't want to manage tasks with CloudReactor): set `enabled: false` instead

4. Ensure Docker is running (you should have installed it [above](#set-up-aws-infrastructure-link-to-cloudreactor)). Deployment with Docker is highly recommended because:
    - you don't need to have python installed directly on your machine
    - you don't need to add another set of dependencies to your libraries
    - you can deploy irrespective of your OS (e.g. if you're running Windows).
    
    If you don't want to use Docker, you'll need to set up your local environment to deploy natively. This requires installing python and various dependencies. See [this section](#){:target="_blank"} for details, and skip steps 5 & 6 below.

5. Build the Docker container that will deploy the project. This may take a few minutes to complete so be ready to make some tea / coffee while you wait.

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

6. Deploy the example tasks to AWS.

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

7. Log into [dash.cloudreactor.io](https://dash.cloudreactor.io){:target="_blank"}. You should see the two example tasks listed there. You can manually start and stop these tasks, or schedule them to run. They'll run on the AWS ECS infrastructure we created earlier.

---

## Development workflow: adding your own tasks

### Add task code

1. Place task code itself in a new file in `./src`, e.g. `new_task.py`

    Add any dependencies to `/requirements.in`. For example:
    ```
    psycopg2==2.8.5
    ```

### Add task to manifest and docker-file

1. Open `./deploy/vars/common.yml`.

    You'll see entries for both `task_1` and `file_io`. E.g.:

    ```python
    # This task sends status back to CloudReactor as it is running
    task_1:
        <<: *default_task_config
        description: "This description shows up in CloudReactor dashboard"
        command: "python src/task_1.py"
        schedule: cron(9 15 * * ? *)
        wrapper:
            <<: *default_task_wrapper
            enable_status_updates: true
    ```

    You can think of `common.yml` as a manifest of tasks. Running `./docker_deploy.sh` will push the files defined here to ECS, and register them with CloudReactor.

2. Add a configuration block for your new task, below `task_name_to_config:`

    The minimum required block is:

    ```python
    new_task:
        <<: *default_task_config
        command: "python src/new_task.py"
    ```

    - `new_task`: a name for the configuration block
    - `<<: *default_task_config` allows new_task to inherit properties from the default task configuration
    - `command: "python src/new_task.py"` contains the command to run (in this case, to execute `new_task` via python)
    
    Additional parameters include the run schedule (cron expression), retry parameters, and environment variables. See [additional configuration](/configuration.md) for more options.

3. Open `/docker-compose.yml` (in the root folder). Under the section `services` add a reference to your task:

    ```
    new_task:
        <<: *service-base
        command: python src/new_task.py
    ```

### Run & test tasks locally

1. If this is the first time running the task locally, run:

    ```
    docker-compose run --rm pip-compile
    docker-compose build
    ```
    - *docker-compose run --rm pip-compile*: this generates a new `requirements.txt` (used by Docker Compose) from `/requirements.in`
    - *docker-compose build*: this builds the container

2. Then to run e.g. `new_task` locally:

    ```
    docker-compose run --rm new_task
    ```

3. As you continue to make changes to `new_task.py`, just re-run `docker-compose run --rm new_task` to execute that task locally.

    **You do not need to run `docker-compose build` each time you make changes to `new_task`.** There won't be any adverse consequences from doing so; this step just adds time.

    Under the hood, changes to files in `/src` and in the environment file `deploy/files/.env.dev` will be copied into the docker container automatically. This is why you don't need to run `docker-compose build` to rebuild the container.

4. Run `docker-compose run --rm pip-compile` and `docker-compose build` if `requirements.in` has changed.


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

## Tracking rows processed, custom status messages

Two frequent needs when it comes to monitoring tasks is understanding how many rows have been processed, and what a given task is doing.

The CloudReactor dashboard provides pre-defined fields where this information can be viewed. **If you want to see data in these fields (as opposed to blanks), you must instrument your task to send this data to CloudReactor.**

To see a working example of how to do this, see `/src/task_1.py` in the quickstart repo. Further instructions below!

### Check that status updates are enabled

In `/deploy/vars/common.yml`, look for the block starting with `wrapper: &default_task_wrapper`.

Add the line `enable_status_updates: True` inside this block if it's not there already. It should look like this:

    wrapper: &default_task_wrapper
        enable_status_updates: True

### Call `send_update` in your task

1. In your .py file in `/src/`, import the status_updater library:
    ```
    from status_updater import StatusUpdater
    ```

2. Create a new instance of StatusUpdater e.g.

    ```
    updater = StatusUpdater()
    ```

3. To send number of rows:
    ```
    updater.send_update(success_count=success_num_rows)
    ```
    
    where `success_num_rows` is the number of rows of "successful" records processed. This number will show up as the "processed" column in the CloudReactor dashboard.

    You need to write the logic that tracks this number as part of your code (e.g. as rows are processed, increment the variable).

    Other "counts" that can be sent are below. These will show up in the relevant columns in CloudReactor.
    - `error_count`
    - `skipped_count`
    - `expected_count`


4. The CloudReactor dashboard also shows "last status message" for each task. This allows you to report custom messages during your task execution, improving visibility into what stage each task is at.

    For example a given task might report status messages such as "getting auth token", "fetching data", "saving data" etc. 

    Send a custom “status message” via StatusUpdater() like this:
    ```
    updater.send_update(last_status_message="started data ingestion")
    ```
5. Bundle multiple row counts and status message into a single call like this:
    ```
    updater.send_update(success_count=success_num_rows, error_count=error_num_rows, last_status_message="finished data ingestion")
    ```
6. Call `updater.shutdown()` to close this socket when your task completes.

    StatusUpdater uses a UDP socket to send data. Shutdown() ensures that socket is closed.

    Although it's good practice to clean up in this way, it's not strictly necessary since when the task finishes, all processes will end anyway.


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

## The example tasks

The example tasks show a few helpful features of CloudReactor that can be used in your own code.

- *task_1* prints 30 numbers and exits successfully. While it does so, it uses the CloudReactor status updater library to update the "successful" count and the "last status message" that is shown in the CloudReactor dashboard. These can be used to help track "# of rows processed successfully" or progress through the code. Note that task_1 is configured to run daily via `deploy/vars/common.yml`: `schedule: cron(9 15 * * ? *)`.
- *file_io* uses [non-persistent file storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html){:target="_blank"} to write and read numbers
- *web_server* uses a python library dependency (Flask) to implement a web server and shows how to link an AWS Application Load Balancer (ALB) to a service. It requires that an ALB and target group be setup already, so it is not enabled by default (i.e. is commented out in the `./deploy/vars/common.yml` file).

---

## More information

* [Additional configuration](/configuration.md) options can be set or overridden
* If you want to be alerted when task executions fail, setup an [Alert Method](/alerts.md)
* To avoid leaking secrets (passwords, API keys, etc.), see the guide on [secret management](/secret_management.md)
* For more secure [networking](/networking.md), run your tasks on private subnets and/or tighten your security groups.
* If you're having problems, see the [troubleshooting guide](/troubleshooting.md)


---

## Contact us

Hopefully, this example project has helped you get up and running with ECS and
CloudReactor. Feel free to reach out to us at support@cloudreactor.io to setup 
an account, or if you have any questions or issues!
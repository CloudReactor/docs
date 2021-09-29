---
layout: default
title: Task Configuration for aws-ecs-cloudreactor-deploy
nav_order: 6
---
# Task configuration for aws-ecs-cloudreactor-deploy

The [aws-ecs-cloudreactor-deploy](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
Docker image can be used to deploy projects to both AWS ECS and CloudReactor.
It reads configuration from your `deploy_config` directory set properties in
both the
[ECS task definition](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html)
and the [CloudReactor Task](https://apidocs.cloudreactor.io/). We'll show you
how to modify the files in this directory below.

## Configuration hierarchy

The settings are all (deeply) merged together with Ansible's Jinja2
[combine](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html#combining-hashes-dictionaries)
filter. The precedence of settings, from lowest to highest is:

1. Settings found in your Run Environment that you set via the CloudReactor dashboard
2. Deployment environment AWS settings -- found in `project_aws` in `deploy/vars/<environment>.yml`
3. Default Task settings -- found in `default_task_config` in `deploy/vars/common.yml`
4. Per Task settings -- found in `task_name_to_config.<task_name>` in `deploy/var/common.yml`
5. Per environment settings -- found in `default_env_task_config` in `deploy/vars/<environment>.yml`.
See `deploy/vars/example.yml` an example.
6. Per Task, per environment settings -- found in `task_name_to_env_config.<task_name>` in `deploy/vars/<environment>.yml`

## ECS Task Definition settings

* You can add additional properties to the main container running each Task,
such as `mountPoints` and `portMappings`  by setting
`extra_main_container_properties` in common.yml or `deploy/vars/<environment>.yml`.
See the `file_io` Task for an example of this.
* You can add AWS ECS task properties, such as `volumes` and `secrets`,
by setting `extra_task_definition_properties` in the `ecs` property of each task
configuration. See the `file_io` Task for an example of this.
* You can add additional containers to the Task by setting `extra_container_definitions`
in `deploy/vars/common.yml` or `deploy/vars/<environment>.yml`.


## Setting management

We recommend that you keep as many default settings in CloudReactor as possible
and don't override them in your project unless required. To manage a setting in
the CloudReactor UI, omit the property name and value. That will tell
CloudReactor to use the value that you have set either in the Task or
Run Environment editor.

It's possible that you overrode a Task setting during a previous deployment, but
later decide to use the default value in the Run Environment. To do that, set
the property value to null.

## Removing Tasks

To remove a Task, delete the Tasks within the
[CloudReactor dashboard](https://dash.cloudreactor.io){:target="_blank"}.
This will remove the Task rom AWS also.

You should also remove the reference to the Tasks in `./deploy/vars/common.yml`.
- If you don't, if you run `./cr_deploy.sh [environment]` (without task names), this will (re-)push all tasks -- which might include tasks you had intended to remove.

You may also want to remove the task code itself from `/src/`

For example, if you want to delete the `task_1` Task:

1. In [dash.cloudreactor.io](https://dash.cloudreactor.io){:target="_blank"},
hit the delete icon next to `task_1` and hit "confirm".
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

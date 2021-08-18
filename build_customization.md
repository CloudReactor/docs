---
layout: default
title: Build customization of aws-ecs-cloudreactor-deploy
nav_order: 5
---
# Build customization of aws-ecs-cloudreactor-deploy

[aws-ecs-cloudreactor-deploy](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
is a Docker image that is able to deploy Tasks to AWS ECS and CloudReactor.
aws-ecs-cloudreactor-deploy uses
[Ansible](https://docs.ansible.com/ansible/latest/index.html)
to execute a playbook that includes steps to resolve the configuration
properties, before deploying to both AWS and CloudReactor.

Out of the box, it only supports unencrypted secrets, or secrets encrypted with
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html).
It is geared toward code written in dynamic languages that do not require
compilation. However, you can customize the image to support other ways of
handling secrets and additional build steps such as compilation.

## Custom build steps

You can run custom build steps by adding steps to
`deploy_config/hooks/pre_build.yml` and
`deploy_config/hooks/post_build.yml` as necessary.

If you need to use libraries (e.g. compilers) not available in
the aws-ecs-cloudreactor-deploy image, your custom build steps can either:

1) Use the `docker` command to build intermediate files (like JAR files or
executables). Use `docker build` to build images, `docker create` to
create containers, and finally, `docker cp` to copy files from containers
back to the host. When docker runs in the container, it will use the
host machine's docker service.

2) Use build tools installed in the deployer image. In this case, you'll
want to create a new image based on
`cloudreactor/aws-ecs-cloudreactor-deployer`:

    ```
    FROM cloudreactor/aws-ecs-cloudreactor-deployer:1.1.0

    # Example: get the JDK to build JAR files
    RUN apt-get update && \
      apt-get -t stretch-backports install openjdk-11-jdk

    ...
    ```

Then set the `DOCKER_IMAGE` environment variable to the name of your new image,
or change the deployment command in `cr_deploy.sh` to use your new image
instead of `cloudreactor/aws-ecs-cloudreactor-deployer`. Your ansible tasks
can now use `javac`. If you create a Docker image for a specific language, we'd
love to hear from you!

Also, check out
[multi-stage Dockerfiles](https://docs.docker.com/develop/develop-images/multistage-build/)
as a way to build dependencies in the same Dockerfile that creates the final
container. This may complicate the use of the same Dockerfile during
development, however.

### cr_deploy.sh configuration

`cr_deploy.sh` is what you'll call on your host machine, which will
run the Docker image for the deployer. The deployer Docker image has an
entrypoint that executes the python script deploy.py, which in turn,
executes ansible-playbook.

The Ansible tasks in `ansible/deploy.yml` reference files that you
can make available with Docker volume mounts. You can either modify
`cr_deploy.sh` to add or modify existing mounts, or configure the
files/directories with environment variables. The Ansible tasks also read
environment variables which you can set in `deploy.env` or
`deploy.<environment>.env`.

You can configure some settings in `cr_deploy.sh` with environment
variables if you want to avoid modifying it:

| Environment variable name |       Default value      | Description                                                                                    |
|---------------------------|:------------------------:|------------------------------------------------------------------------------------------------|
| DOCKER_CONTEXT_DIR        |     Current directory    | The absolute path of the Docker context directory                                              |
| DOCKERFILE_PATH           |        `Dockerfile`      | Path to the Dockerfile, relative to the Docker context                                                |
| CLOUDREACTOR_TASK_VERSION |           Empty          | A version number to report to CloudReactor. If empty, the latest git commit hash will be used. |
| PER_ENV_SETTINGS_FILE     |`deploy.<environment>.env`| Path to a dotenv file containing environment-specific settings                                 |
| USE_USER_AWS_CONFIG       |          `FALSE`         | Set to TRUE to use your AWS configuration in `$HOME/.aws` |
| AWS_PROFILE     |Empty| The name of the AWS profile to use, if `USE_USER_AWS_CONFIG` is `TRUE`. If not specified, the default profile will be used. |
| EXTRA_DOCKER_RUN_OPTIONS     |Empty| Additional [options](https://docs.docker.com/engine/reference/commandline/run/) to pass to `docker run`                                 |
| DEPLOY_COMMAND            |    `python deploy.py`    | The command to use when running the image. Defaults to `bash` when `DEBUG_MODE` is `TRUE`.
| EXTRA_ANSIBLE_OPTIONS     |           Empty          | If specified, the default `DEPLOY_COMMAND` will appended with `--ansible-args $EXTRA_ANSIBLE_OPTIONS`. These options will be passed to `ansible-playbook` inside the container. |
| DOCKER_IMAGE              	|`cloudreactor/aws-ecs-cloudreactor-deployer`	| The Docker image to run. Can be set to another name in case you extend the image to add build or deployment tools. 	|
| DOCKER_IMAGE_TAG           	|`1`	| The tag of the Docker image to run. Can also be set to pinned versions like `1.2.2`, compatible releases like `1.2`, or `latest`. |
| DEBUG_MODE                  | `FALSE` | If set to `TRUE`, docker will be run in interactive mode (`-ti`) and a bash shell will be started inside the container. |

If you want to avoid modifying `cr_deploy.sh`, you can create a script that
configures some settings with environment variables, then calls `cr_deploy.sh`.
See [`deploy_sample.sh`](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer/blob/main/deploy_sample.sh)
for an example.

The behavior of ansible-playbook can be modified with many command-line
options. To pass options to ansible-playbook, either:

1. Add `--ansible-args` to the end of the command-line for `cr_deploy.sh`,
followed by all the options you want to pass to ansible-playbook. For example,
to use secrets encrypted with ansible-vault and get the encryption password from
the command-line during deployment:


        ./cr_deploy.sh staging --ansible-args --ask-vault-pass

    Alternatively, you can use a password file:

        ./cr_deploy.sh staging --ansible-args --vault-password-file pw.txt

    The password file could be a plaintext file, or a script like this:

        #!/bin/bash
        echo `aws s3 cp s3://widgets-co/vault_pass.$DEPLOYMENT_ENVIRONMENT.txt -`

If you use a password file, make sure it is available in the Docker
context of the container. You can either put it in your Docker context
directory or add an additional mount option to the docker command-line.

2. Or, specify `EXTRA_ANSIBLE_OPTIONS`. For example, to specify the password
file:

        EXTRA_ANSIBLE_OPTIONS="--vault-password-file pw.txt" ./cr_deploy.sh staging

## More customization

You can customize the build even more by overriding any of the files in
the `ansible` directory of aws-ecs-cloudreactor-deployer
with you own version, by passing
a volume mount option to the Docker command line. For example, to override
`ansible.cfg` and `deploy.yml`, set the `EXTRA_DOCKER_RUN_OPTIONS` environment
variable before calling `cr_deploy.sh`:

    export EXTRA_DOCKER_RUN_OPTIONS="-v $PWD/ansible_overrides/ansible.cfg:/work/ansible.cfg -v $PWD/ansible_overrides/deploy.yml:/work/deploy.yml"

aws-ecs-cloudreactor-deploy uses templates contained in `/work/templates`
of its Docker image:

* The ECS task definition is created with the Jinja2 template
`deploy/templates/ecs_task_definition.json.j2`.
* The CloudReactor Task is created with the Jinja2 template
`deploy/templates/cloudreactor_task.yml.j2`. which produces a YAML
file that is converted to JSON before sending it CloudReactor.

These templates use settings from the files described above. If you need to
modify the templates, you can override the default templates similarly:

    export EXTRA_DOCKER_RUN_OPTIONS="-v $PWD/ansible_overrides/templates/ecs_task_definition.json.j2:/work/templates/ecs_task_definition.json.j2"

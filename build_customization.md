---
layout: default
title: Build customization of aws-ecs-cloudreactor-deploy
nav_order: 7
---
# Build customization of aws-ecs-cloudreactor-deploy

## Background

[aws-ecs-cloudreactor-deploy](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
is a Docker image that is able to deploy Tasks to AWS ECS and CloudReactor.
aws-ecs-cloudreactor-deploy uses
[Ansible](https://docs.ansible.com/ansible/latest/index.html)
to execute a playbook that includes steps to resolve the configuration
properties, before deploying to both AWS and CloudReactor.

Out of the box, it supports unencrypted secrets, secrets encrypted with
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html),
and optionally uploaded to AWS Secrets Manager.
It is geared toward code written in dynamic languages that do not require
compilation. However, you can customize the image to support other ways of
handling secrets and additional build steps such as compilation.

## Custom build steps

You can run custom build steps by adding steps to the following files in
`deploy_config/hooks`:

* `pre_build.yml`: run before the Docker image is built. Compilation and
asset processing can be run here. You can also login to Docker repositories
and/or upload secrets to Secrets Manager.
* `post_build.yml`: run after the Docker image has been uploaded. Executions
of "docker run" can be run here. For example, database migrations can be
run from the local deployment machine.
* `post_task_creation.yml`: run each time a Task is deployed to ECR and
CloudReactor. Execution of Tasks that were just deployed can be run here.

In these build steps, you can use the
[community.docker](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_container_module.html) and [community.aws](https://docs.ansible.com/ansible/latest/collections/community/aws/index.html) Ansible Galaxy plugins which are included
in the deployer image, to perform setup operations like:

* Creating/updating secrets in Secrets Manager
* Uploading files to S3
* Creating roles and seting permissions
* Sending messages via SNS

If you need to use libraries (e.g. compilers) not available in this image,
your custom build steps can either:

1) Use
[multi-stage Dockerfiles](https://docs.docker.com/develop/develop-images/multistage-build/)
as a way to build dependencies in the same Dockerfile that creates the final
container. This may complicate the use of the same Dockerfile during
development, however.

2) Use the `docker` command to build intermediate files (like JAR files or executables).
Use `docker build` to build images, `docker create` to
create containers, and finally, `docker cp` to copy files from containers
back to the host. When docker runs in the container, it will use the
host machine's docker service.

3) Use build tools installed in a custom deployer image. In this case, you'll
want to create a new image based on `cloudreactor/aws-ecs-cloudreactor-deployer`:

        FROM cloudreactor/aws-ecs-cloudreactor-deployer:2.0.1
        # Example: get the JDK to build JAR files
        RUN apt-get update && \
          apt-get -t stretch-backports install openjdk-11-jdk

        ...

    Then set the `DOCKER_IMAGE` environment variable to the name of your new
    image, or change the deployment command in `cr_deploy.sh` to use your
    new image instead of `cloudreactor/aws-ecs-cloudreactor-deployer`. Your
    ansible tasks can now use `javac`. If you create a Docker image for a
    specific language, we'd love to hear from you!

During your custom build steps, the following variables are available:

1. `work_dir` points to the directory in the container in which
the root directory of your project is mounted. This is `/work` for
command-line builds.
2. `deploy_config_dir` points to the directory in the container in which the
`deploy_config` directory is mounted. This is `/work/deploy_config` for
command-line builds.
3. `docker_context_dir` points to the directory on the host is the Docker context
directory. For command-line builds, this is the project root directory unless
overridden by `DOCKER_CONTEXT_DIR`. It is mounted in `/work/docker_context`
in the container.

You can find more helpful variables in the `vars` section of
[`ansible/vars/common.yml`](ansible/vars/common.yml).

## Setup deployment from the command-line

To enable deployment by running from a command-line prompt, copy `cr_deploy.sh`
to the root directory of your project. `cr_deploy.sh` will run the Docker
image for the deployer. It can be configured with the following
environment variables:

| Environment variable name |       Default value      | Description                                                                                    |
|---------------------------|:------------------------:|------------------------------------------------------------------------------------------------|
| DOCKER_CONTEXT_DIR        |     Current directory    | The absolute path of the Docker context directory                                              |
| DOCKERFILE_PATH           |        `Dockerfile`      | Path to the Dockerfile, relative to the Docker context                                                |
| CLOUDREACTOR_TASK_VERSION_SIGNATURE |       Empty          | A version number to report to CloudReactor. If empty, the latest git commit hash will be used if git is available. If git is not available, the current timestamp will be used. |
| CLOUDREACTOR_DEPLOY_API_KEY |         Empty         | The CloudReactor Deployment API key. Can be used instead of setting it in `deploy_config/vars/<environment>.yml`. |
| CONFIG_FILENAME_STEM      | The deployment environment | Use this setting if you store configuration in files that have a different name than the deployment environment they are for. For example, you can use the file `deploy_config/vars/staging-cmdline.yml` to store the settings for the `staging` deployment environment, if you set `CONFIG_FILENAME_STEM` to `"staging-cmdline"`. |
| PER_ENV_SETTINGS_FILE     |`deploy.<config filename stem>.env`| Path to a dotenv file containing environment-specific settings                                 |
| USE_USER_AWS_CONFIG       |          `FALSE`         | Set to TRUE to use your AWS configuration in `$HOME/.aws` |
| AWS_PROFILE     |Empty| The name of the AWS profile to use, if `USE_USER_AWS_CONFIG` is `TRUE`. If not specified, the default profile will be used. |
| PASS_AWS_ACCESS_KEY       |          `FALSE`         | Set to TRUE to use pass the  `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables to the deployer |
| EXTRA_DOCKER_RUN_OPTIONS     |Empty| Additional [options](https://docs.docker.com/engine/reference/commandline/run/) to pass to `docker run`                                 |
| EXTRA_ANSIBLE_OPTIONS     |           Empty          | If specified, the default `DEPLOY_COMMAND` will appended with `--ansible-args $EXTRA_ANSIBLE_OPTIONS`. These options will be passed to `ansible-playbook` inside the container. |
| ANSIBLE_VAULT_PASSWORD    |           Empty          | If specified, the password will be used to decrypt files encrypted by Ansible Vault |
| DOCKER_IMAGE              	|`cloudreactor/aws-ecs-cloudreactor-deployer`	| The Docker image to run. Can be set to another name in case you extend the image to add build or deployment tools. 	|
| DOCKER_IMAGE_TAG           	|`2`	| The tag of the Docker image to run. Can also be set to pinned versions like `2.0.1`, compatible releases like `2.0`, or `latest`. |
| DEBUG_MODE                  | `FALSE` | If set to `TRUE`, docker will be run in interactive mode (`-ti`) and a bash shell will be started inside the container. |
| DEPLOY_COMMAND            |    `python deploy.py`    | The command to use when running the image. Defaults to `bash` when `DEBUG_MODE` is `TRUE`. |

If possible, try to avoid modifying `cr_deploy.sh`, because this project
will frequently update it with options. Instead, create a wrapper script
that configures some settings with environment variables, then calls
`cr_deploy.sh`.
See [`deploy_sample.sh`](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer/blob/main/deploy_sample.sh)
for an example.

The deployer Docker image has an entrypoint that executes the python script
`deploy.py`, which in turn, executes ansible-playbook.

The Ansible tasks in `ansible/deploy.yml` reference files that you
can make available with Docker volume mounts. You can either modify
`cr_deploy.sh` to add or modify existing mounts, or configure the
files/directories with environment variables. The Ansible tasks also read
environment variables which you can set in `deploy.env` or
`deploy.<config filename stem>.env`. To start from a template, copy
`deploy.env.example` to `deploy.env` and
and fill in your AWS access key, access key secret, and default
region. The access key and secret would be for the AWS user you plan on using
to deploy with, as described in
[Deployer API Permissions](/deployer_api_permissions.html).

Also fill in the value of `CLOUDREACTOR_DEPLOY_API_KEY`. You created this
key (which has the `Developer` Access Level) in the
[Getting Started](/#create-a-cloudreactor-api-key) section.

### Ansible options

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

    The file `ansible/vault_pass_from_env.sh` may also be used so that the
    vault password can come from the environment variable `ANSIBLE_VAULT_PASS`:

        ./cr_deploy.sh staging --ansible-args --vault-password-file /work/vault_pass_from_env.sh

If you use a password file, make sure it is available in the Docker
context of the container. You can either put it in your Docker context
directory or add an additional mount option to the docker command-line.

2. Or, specify the `EXTRA_ANSIBLE_OPTIONS` environment variable. For example,
to specify the password file:

        EXTRA_ANSIBLE_OPTIONS="--vault-password-file pw.txt" ./cr_deploy.sh staging

### More customization

You can customize the build even more by overriding any of the files in
the `ansible` directory of aws-ecs-cloudreactor-deployer
with you own version, by passing
a volume mount option to the Docker command line. For example, to override
`ansible.cfg` and `deploy.yml`, set the `EXTRA_DOCKER_RUN_OPTIONS` environment
variable before calling `cr_deploy.sh`:

    export EXTRA_DOCKER_RUN_OPTIONS="-v $PWD/ansible_overrides/ansible.cfg:/work/ansible.cfg -v $PWD/ansible_overrides/deploy.yml:/work/deploy.yml"

* The ECS task definition is created with the Jinja2 template
`ansible/templates/ecs_task_definition.json.j2`.
* The CloudReactor Task is created with the Jinja2 template
`ansible/templates/cloudreactor_task.yml.j2`. which produces a YAML
file that is converted to JSON before sending it CloudReactor.

These templates use settings from the files described above. If you need to
modify the templates, you can override the default templates similarly:

    export EXTRA_DOCKER_RUN_OPTIONS="-v $PWD/ansible_overrides/templates/ecs_task_definition.json.j2:/work/templates/ecs_task_definition.json.j2"

### Deploying by command-line:

Once you are done with configuration, you can deploy:

    ./cr_deploy.sh <environment> [TASK_NAMES]

or in Windows:

    .\cr_deploy.cmd <environment> [TASK_NAMES]

where `TASK_NAMES` is an optional, comma-separated list of Tasks to deploy.
If `TASK_NAMES` is omitted, or set to `ALL`, all Tasks will be deployed.

If you wrote a wrapper over `cr_deploy.sh`, use that instead.

## Setup deployment via GitHub Action

{: #github-action }

This Docker image can also be used as a
[GitHub Action](https://github.com/marketplace/actions/cloudreactor-aws-ecs-deployer).
As an example, in a file named `.github/workflows/deploy.yml`, you could have
something like this to deploy to your staging environment after you commit to
the master branch:

    name: Deploy to AWS ECS and CloudReactor
    on:
      push:
        branches:
          - master
        paths-ignore:
          - '*.md'
          - 'docs/**'
      workflow_dispatch: # Allows deployment to be triggered manually
        inputs: {}
    jobs:
      deploy:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v2
        - name: Deploy to AWS ECS and CloudReactor
          uses: CloudReactor/aws-ecs-cloudreactor-deployer@v2.0.1
          with:
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ${{ secrets.AWS_REGION }}
            ansible-vault-password: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
            deployment-environment: staging
            cloudreactor-deploy-api-key: ${{ secrets.CLOUDREACTOR_DEPLOY_API_KEY }}
            log-level: DEBUG

In the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` GitHub secrets, you would set
the access key ID and secret access key for the AWS user that has the permissions
necessary to deploy to ECS, as described above. The `AWS_REGION` would store
the region, such as `us-west-1`, where you want deploy your Task(s).

You would populate the GitHub secret `CLOUDREACTOR_DEPLOY_API_KEY` with the
value of the `Deployment API key`, as described above.

The optional `ANSIBLE_VAULT_PASSWORD` GitHub secret would store the password
used to decrypt configuration files (such as
`deploy_config/vars/staging.yml`) that were encrypted with Ansible Vault.

See the deployer's [GitHub Action definition](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer/blob/main/action.yml) for a full list of options.

## More details

More details can be found in the
[deployer project's documentation](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer).

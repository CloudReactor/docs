---
layout: default
title: Secret management
nav_order: 6
---

# Secret management
{: .no_toc}

If you are using a CloudReactor quick start project, like
[cloudreactor-python-ecs-quickstart](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart),
you most likely added sensitive data to per-environment files, which you don't
want to commit in plaintext. The default configuration assumes you just keep
those secret files locally and don't commit them -- thus they are excluded in
.gitignore.

However, for sharing with a team or to have the secrets in source control
for backup/history reasons, it's better to check the secret files in, but
encrypted. Alternatively, you can get secrets from AWS at runtime.

Here, we'll cover the management of 2 types of secrets:
* Deployment secrets: these are secrets needed to deploy your Task, but are not required while the Task is running. For example, to deploy Tasks using CloudReactor, you add AWS keys to the file `deploy.env`, and your
CloudReactor Deploy API key to the file `deploy_config/vars/<environment>.yml` (where `<environment>` is the name of a Run Environment in CloudReactor).

* Runtime secrets: these are secrets that your Task uses while it is running.
Examples are database passwords and API keys for 3rd party services your Task
uses. These typically are injected as environment variables that your Task
reads. Using [proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper) to fetch secrets and populate the environment is
a convenient way inject secrets.

1. TOC
{:toc}

---

## Deployment Secrets

### Ansible Vault

One option for managing either deployment secrets is to use
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
which is well integrated with ansible. Ansible is used to deploy the
example projects, and it will transparently decrypt files encrypted with
ansible-vault when copying them.

In the example projects, to use ansible-vault to encrypt your deployment
secrets, first create a password (method from [howtogeek](https://www.howtogeek.com/howto/30184/10-ways-to-generate-a-random-password-from-the-command-line/))
in a directory outside your project. The example below use `$HOME/secrets`:

< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c64 > vault_pass.txt

Then change your directory to `deploy_config/vars` and run:

    ansible-vault encrypt --vault-password-file $HOME/secrets/vault_pass.txt [environment].yml

ansible-vault will prompt for a password, then encrypt the file. To edit it:

    ansible-vault edit --vault-password-file $HOME/secrets/vault_pass.txt [environment].yml

Next, create a file `deploy.sh` with the following contents:

    #!/usr/bin/env bash

    set -e

    export ANSIBLE_VAULT_PASSWORD=`cat ../secrets/vault_pass.txt`
    EXTRA_DOCKER_RUN_OPTIONS="-e ANSIBLE_VAULT_PASSWORD"

    ./cr_deploy.sh "$@"

What's going on under the hood? The deployer image contains a special script
(`ansible/vault_pass_from_env.sh`)
that will be used when `ANSIBLE_VAULT_PASSWORD` is defined. The script is used
as the Ansible Vault password file, and simply outputs the value of
`ANSIBLE_VAULT_PASSWORD`.

To deploy, just use your custom `deploy.sh` instead of `cr_deploy.sh`:

    ./deploy.sh <deployment_environment>

It may be more convenient to have deployer container to fetch the
Ansible Vault password, because it has access to your AWS environment and
tools like the AWS CLI. For example, to have the container fetch the password
from an AWS S3 object, create the bash script `deploy_config/vault_pass.sh`:

    #!/bin/bash
    echo `aws s3 cp s3://mygroup/settings/ansible/vault_pass.$DEPLOYMENT_ENVIRONMENT.txt -`

(`$DEPLOYMENT_ENVIRONMENT` will resolve to the deployment environment you pass
to `cr_deploy.sh`.)

Then your custom `deploy.sh` would like this:

    #!/usr/bin/env bash

    set -e

    export EXTRA_ANSIBLE_OPTIONS="--vault-password-file /work/deploy_config/vault_pass.sh"

    ./cr_deploy.sh "$@"

#### Using deployment secrets encrypted with Ansible Vault

Once you inject the Ansible Vault password into the deployer image,
Ansible will be able to read files encrypted with ansible-vault, and you
can reference variables defined in those files in any task or template
that it uses. For example you might encrypt the file
`deploy_config/vars/staging.yml` whose plaintext contents contain:

    cloudreactor:
      deploy_api_key: SOME_SECRET_API_KEY

    # App specific settings here, in this example, credentials to fetch
    # libraries from a private repository:
    artifactory:
      username: repo_user
      password: repo_password

The default deployment tasks will then be able to use the value of
`cloudreactor.deploy_api_key` when connecting to CloudReactor.

To inject secrets available for your Dockerfile to use, add something like
this to `deploy_config/vars/common.yml`:

    default_build_options:
      extra_docker_build_args: "--build-arg ARTIFACTORY_USER={{artifactory.username | quote}} --build-arg ARTIFACTORY_PASSWORD={{artifactory.password | quote}}"

Then in your `Dockerfile` you can use the build args:

    ...
    ARG ARTIFACTORY_USER
    ARG ARTIFACTORY_PASSWORD
    RUN ./gradlew shadowJar -Partifactory_user="${ARTIFACTORY_USER}" -Partifactory_password="${ARTIFACTORY_PASSWORD}"
    ...

---

### git-crypt

For deployment secrets, another option for encryption is
[git crypt](https://github.com/AGWA/git-crypt),
which encrypt secrets when they are committed to Git.
However, this leaves secrets unencrypted in the filesystem where the
repository is checked out. That can be advantage as it is easier
to edit and search secret files.
The disadvantage of using git-crypt is that if the machine or disk that
contains these secret files is compromised, those secrets can be exposed.

If you set up git-crypt, uncomment the lines in `.gitignore` that ignore
secret files, since they will be encrypted in the repository (see below).
Also uncomment the lines in `.gitattributes` that specify which files to encrypt.
Then just add secrets directly to any configuration file which you have
selected to be encrypted.

---

## Runtime secrets

If you are running your tasks in AWS, one option for runtime secrets
is to store them in AWS Secrets Manager. When your Task runs, it can use
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper]) to
fetch the secrets from Secrets Manager and inject them into your Tasks's
process environment. Or your Task can fetch the secrets itself.
If you use Secrets Manager, then you don't need to store
secrets at all in your source repository, only locations of secrets in
Secrets Manager. You can also use Ansible Vault or git-crypt to encrypt
runtime secrets in your repository, and use the deployer image to store them in
Secrets Manager as part of the deployment process.

### Storing secrets separately from deployment

If you don't want to store secrets as part of the deployment process,
you can store them in AWS Secrets Manager by using the AWS Management Console
or Terraform.

#### Using the AWS Management Console

To use the AWS Management Console, log into
[AWS Secrets Manager](https://aws.amazon.com/secrets-manager/). Create a secrets
value as plaintext, in .env, JSON, or YAML format.

For example, to use the .env format (recommended if you are using proc_wrapper),
you would create a secrets value named
"myorg/myapp/production/db.env" that contains the following environment
variables:

    PROC_WRAPPER_API_KEY="xxx"
    DB_HOST=db.example.com
    DB_PORT=5432
    DB_NAME=mydb
    DB_USERNAME=dbuser
    DB_PASSWORD=dbpassword

After storing the secret, copy the ARN of the secret. It should look like:

    arn:aws:secretsmanager:us-west-1:012345678901:secret:myorg/myapp/production/db.env-BHyuR

#### Using Terraform

Alternatively, if you want to keep your infrastructure definitions and secrets
in source control, you can
[use Terraform to set secret values](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret).

Due to the available functions in Terraform, it may be easier to store secrets
in JSON format. For example, in a `variables.tf` file:

    variable "task_secrets" {
      description = "Secrets for MyApp tasks"
      type = map(string)
    }

In the main `app.tf` file:

    resource "aws_secretsmanager_secret" "task_secrets" {
      name = "myorg/myapp/staging/config.json"
    }

    resource "aws_secretsmanager_secret_version" "task_secrets_value" {
      secret_id     = aws_secretsmanager_secret.task_secrets.id
      secret_string = jsonencode(var.task_secrets)
    }

And finally in a `staging_secrets.auto.tfvars` file:

    task_secrets = {
        PROC_WRAPPER_API_KEY = "xxx"

        DB_HOST = "db.example.com"
        DB_PORT = "5432"
        DB_NAME = "mydb"
        DB_USERNAME = "dbuser"
        DB_PASSWORD = "dbpassword"
    }

`staging_secrets.auto.tfvars` could be encypted using git-crypt or using
Ansible Vault if you use the
[Ansible Vault Terraform provider](https://meilleursagents.github.io/terraform-provider-ansiblevault/).

After you run `terraform plan` and `terraform apply`, copy the ARN for
aws_secretsmanager_secret.task_secrets which should look like:

    arn:aws:secretsmanager:us-west-1:012345678901:secret:myorg/myapp/production/db.config-BHyuR

### Storing secrets as part of the deployment process

Alternatively, you may want to manage secrets as part of the deployment process
so that no separate step to set secrets is required.
To automate secrets being uploaded to Secrets Manager during deployment,
you can use the
[CloudReactor deployer image](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer).
The example projects already use this image, and have the upload
step commented out in [`deploy_config/hooks/pre_build.yml`](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart/blob/ebb68102b45d32613316692b846e6bb107a1388a/deploy_config/pre_build.yml#L18):


    - name: Upload .env file to AWS Secrets Manager
      community.aws.aws_secret:
        name: '{{project_name}}/{{env}}/env'
        state: present
        secret_type: 'string'
        secret: "{{ lookup('file', '/work/deploy_config/files/.env.' + env)  }}"
      register: create_dotenv_secret_result

### Giving your Task permission to access Secrets Manager

Now you have to grant your Task permissions to access Secrets Manager at
runtime. You can do this either manually in the AWS Management Console or
with Terraform. The instructions below create an IAM Role with permissions,
but you can also create an IAM user and use access keys.

#### Granting permission to access secrets using the AWS Management Console

Go to the
[IAM Dashboard](https://console.aws.amazon.com/iam/home?region=us-west-2#)
in the AWS console and create a IAM Role that has permission to access your
secret. In ECS, this role will be the Task Role that your Task runs with.
Give the role a policy that looks like this:

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Action": [
            "secretsmanager:GetSecretValue"
        ],
        "Resource": [
            "arn:aws:secretsmanager:us-west-1:01234567901:secret:myorg/myapp/staging/*"
        ]
      }
    }

where you would substitute "us-west-1" with the region you stored your secret,
"012345678901" with your AWS account number, and "myorg/myapp/staging" with
the path that your secrets are stored in. You can also include additional
permissions that your Task can use to access other AWS resources.

If you are deploying your Task using ECS, the role should also have a
trust relationship so that ECS can assume it:

    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Sid": "",
          "Effect": "Allow",
          "Principal": {
            "Service": [
              "ecs.amazonaws.com",
              "ecs-tasks.amazonaws.com"
            ]
          },
          "Action": "sts:AssumeRole"
        }
      ]
    }

See the official AWS documentation
[IAM Roles for Tasks](https://docs.aws.amazon.com/AmazonECS/latest/userguide/task-iam-roles.html)
for more details.

### Granting permission to access secrets using Terraform

It's also possible to [setup the role using Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role). The project
[example_aws_infrastructure_terraform](https://github.com/CloudReactor/example_aws_infrastructure_terraform) shows how to setup a Task role with the
necessary permissions.

### Assigning the role to your Task

After either method above, record the ARN of the role you created,
which should look like:

    arn:aws:iam::012345678901:role/myapp-task-role-staging

If running in ECS, you'll want to use the role as the "Task Role". If you are
using the CloudReactor deployer image,
starting from, add the `role_arn` property in
`deploy_config/vars/staging.yml`:

    default_env_task_config:
      ...
      ecs:
        ...
        task:
          ...
          role_arn: "arn:aws:iam::012345678901:role/myapp-task-role-staging"

If running in EC2, ensure the EC2 instances you run your program in have the
instance role.
See the official AWS documentation [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
for details.

### Fetching secrets at runtime

If necessary, modify the source code of your Task to ensure it reads all secret
values from environment variables.

Now we can make those secrets available to your Tasks. You could have
your code fetch these secrets directly from AWS Secrets Manager and parse them. However, to avoid writing this code and depending on an AWS library, we
recommend using the
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper)
module fetch, parse, and populate secret values into your environment before
your program starts. That way your program doesn't need to fetch and parse the
secrets itself.

You can follow the instructions in the proc_wrapper
[README.md](https://github.com/CloudReactor/cloudreactor-procwrapper/blob/main/README.md) if setting up
your project yourself, or you can use an [example project](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart)
which uses the deployer image to deploy to AWS ECS Fargate and CloudReactor.

Let's assume you want to deploy a task that reads the database to the
"staging" CloudReactor Run Environment. Then, in the
`deploy_config/vars/staging.yml` file (if it doesn't exist, copy
from `deploy_config/vars/example.yml`), add the `env_locations` property:

      default_env_task_config:
        ...
        wrapper:
          ...
          env_locations:
            - arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/staging/db.env-BHyuR

Then the environment variables DB_HOST, DB_PORT, DB_USERNAME, and DB_PASSWORD
will be populated at runtime by the proc_wrapper module.

You can also remove the suffix (in the above example, `-BHyuR`) from your secret
ARNs if you want your Task to fetch the latest value of your secret, instead of
a pinned value.

If you don't want to use proc_wrapper to fetch secrets, you can fetch secrets
yourself. Some popular libraries for doing that include:

* [boto3](https://pypi.org/project/boto3/) for python
* [AWS SDK for Java](https://aws.amazon.com/sdk-for-java/)
* [AWS SDK for Go](https://aws.amazon.com/sdk-for-go/)

### Deploy

Finally, deploy your project. If you used the proc_wrapper module, it should be
able to fetch and parse your secrets, and inject them into your program's
environment.

---

## Cleanup

### Modifying .gitignore in example projects

Once you figure out which files to encrypt, comment out the lines in
`.gitignore` that ignore secret files, since you'll be checking them in encrypted:

    # Comment out to enable deployment secrets to be committed encrypted,
    # if deploying with Docker:
    # deploy.env
    # deploy.*.env

    # Comment out to enable deployment secrets to be committed encrypted
    # deploy_config/vars/*.yml

    # Comment out to enable runtime secrets to be committed encrypted
    # deploy_config/files/.env.*

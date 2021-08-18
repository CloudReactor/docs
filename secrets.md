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
* Deployment secrets: these are secrets needed to deploy your Task, but are not required while the task is running. For example, to deploy tasks using CloudReactor, you add AWS keys to the file `deploy.env`, and your CloudReactor API key to the file `deploy_config/vars/<environment>.yml` (where "<environment>" is the name of a Run Environment in CloudReactor).

* Runtime secrets: these are secrets that your Task uses while it is running.
Examples are database passwords and API keys for 3rd party services your Task
uses. In the example projects, the file `deploy_config/files/.env.<environment>`
contains runtime secrets. In the python example project, python tasks read this
file at runtime using the
[python-dotenv](https://github.com/theskumar/python-dotenv) library.

1. TOC
{:toc}

---

## Ansible Vault

One option for managing either deployment or runtime secrets is to use
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
which is well integrated with ansible. Ansible is used to deploy the
example projects, and it will transparently decrypt files encrypted with
ansible-vault when copying them.

In the example projects, to use ansible-vault to encrypt your deployment
secrets, change your directory to `/deploy_config/vars` and run:

    ansible-vault encrypt [environment].yml

ansible-vault will prompt for a password, then encrypt the file. To edit it:

    ansible-vault edit [environment].yml

Next, create a file `deploy.sh` with the following contents:

    #!/usr/bin/env bash

    set -e

    # Optional: if you want to allow the deployer image to use your AWS
    # configuration in ~/.aws
    # export USE_USER_AWS_CONFIG="TRUE"

    export EXTRA_ANSIBLE_OPTIONS="--vault-password-file /work/deploy_config/vault_pass.sh"
    # Optional: use to pin the version of the deployer
    # export DOCKER_IMAGE_TAG="1.4.0"

    ./cr_deploy.sh "$@"

Then create the bash script `deploy_config/vault_pass.sh` to read the Ansible vault
passphrase and send it to standard out. For example, to read from an AWS S3
bucket:

    #!/bin/bash
    echo `aws s3 cp s3://mygroup/settings/ansible/vault_pass.$DEPLOYMENT_ENVIRONMENT.txt -`

which would read the object `s3://mygroup/settings/ansible/vault_pass.staging.text`
if the deployment environment passed to `deploy.sh` is `staging`.

Alternatively, you can pass the filename of a file containing the passphrase
itself, ensuring that you volume mount it into the container's filesystem.
In that case, your `deploy.sh` would look like:

    #!/usr/bin/env bash

    if [ -z "$1" ]
      then
        echo "Usage: $0 <deployment> [task_names]"
        exit 1
      else
        export DEPLOYMENT_ENVIRONMENT=$1
    fi

    export EXTRA_DOCKER_RUN_OPTIONS="-v $HOME/vault/$DEPLOYMENT_ENVIRONMENT/vault_pass.txt:/work/deploy_config/vault_pass.txt"
    export EXTRA_ANSIBLE_OPTIONS="--vault-password-file /work/deploy_config/vault_pass.txt"

    ./cr_deploy.sh "$@"

### Using deployment secrets encrypted with Ansible Vault

Once you inject the Ansible Vault passphrase into the deployer image,
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

To inject the artifactory credentials, add this to `deploy_config/vars/common.yml`:

    default_build_options:
      extra_docker_build_args: "--build-arg ARTIFACTORY_USER={{artifactory.username | quote}} --build-arg ARTIFACTORY_PASSWORD={{artifactory.password | quote}}"

Then in your `Dockerfile` you can use the build args:

    ...
    ARG ARTIFACTORY_USER
    ARG ARTIFACTORY_PASSWORD
    RUN ./gradlew shadowJar -Partifactory_user="${ARTIFACTORY_USER}" -Partifactory_password="${ARTIFACTORY_PASSWORD}"
    ...

### Using runtime secrets encrypted with Ansible Vault

You can also use Ansible Vault to encrypt runtime secrets, by encrypting
`deploy_config/files/.env.[environment]` and having ansible copy the
file into your Docker context in `deploy_config/hooks/pre_build.yml`:

    - name: Copy runtime .env file read by application
      copy: |
        src=deploy_config/files/.env.{{env}}
        dest=docker_context/build/.env

Then your Dockerfile can copy the file into the image:

    ARG ENV_FILE_PATH=build/.env
    COPY ${ENV_FILE_PATH} .env

Then you can have the proc_wrapper module inject the environment variables
set in the .env file, by specifying the secret location in `common.yml`:

    default_task_config:
      ...
      wrapper:
        ...
        env_locations:
          - file:///home/appuser/app/.env


However, using Ansible Vault for runtime secrets has the
drawback that the secrets will be saved in plaintext in your container image.
For most applications, that is secure enough because [AWS ECR stores images
encrypted](https://aws.amazon.com/ecr/faqs/) and it is assumed the server
you deploy from (which builds and caches Docker images) is secure.

---

## git-crypt

For runtime or deployment secrets, another option for encryption is
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

## Runtime secrets with AWS Secrets Manager

If you are running your tasks in AWS, one option for runtime secrets
is to store them in AWS Secrets Manager. Then you don't need to store
secrets at all in your source repository, only locations of secrets.

Step 1: Log into [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/)
in the AWS Console. Create a secrets object as plaintext, in .env, json, or yaml
format.

For example, to use the .env format, you would create a secrets object named
"myorg/myapp/production/db.env" that contains the following environment
variables:

    DB_HOST=db.example.com
    DB_PORT=5432
    DB_NAME=mydb
    DB_USERNAME=dbuser
    DB_PASSWORD=dbpassword

After storing the secret, copy the ARN of the secret. It should look like:

    arn:aws:secretsmanager:us-west-1:012345678901:secret:myorg/myapp/production/db.env-BHyuR

If want to automate secrets being uploaded to Secrets Manager, two options are:

1. Use the [CloudReactor deployer image](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer), to upload (locally encypted) secrets to Secrets Manager during
deployment. The example projects already use this image, and have the upload
step commented out in [`deploy_config/hooks/pre_build.yml`](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart/blob/ebb68102b45d32613316692b846e6bb107a1388a/deploy_config/pre_build.yml#L18).
2. [Use Terraform to set secret values](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret)


Step 2: Next, go to the
[IAM Dashboard](https://console.aws.amazon.com/iam/home?region=us-west-2#)
in the AWS console and create a IAM Role that is has permission to access your secret.
It should have a policy that looks like this:

    {
      "Version": "2012-10-17",
      "Statement": {
        "Effect": "Allow",
        "Action": [
            "secretsmanager:GetSecretValue"
        ],
        "Resource": [
            "arn:aws:secretsmanager:us-west-1:01234567901:secret:myorg/myapp/production/*"
        ]
      }
    }

where you would substitute "us-west-1" with the region you stored your secret,
"012345678901" with your AWS account number, and "myorg/myapp/production" with
the path that your secrets are stored in.

If you are deploying your task using ECS, the role should also have a
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

It's also possible to [setup the role using Terraform](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role).

Once you've created the role, record the ARN which should look like:

    arn:aws:iam::012345678901:role/myapp-task-role-production

which we will use in step 4.

Step 3: Now we can make those secrets available to your Tasks. You could have
your code fetch these secrets directly from AWS Secrets Manager and parse them. However, to avoid writing this code and depending on an AWS library, we
recommend using the
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper)
module fetch, parse, and populate secret values into your environment before
your program starts. That way your program doesn't need to fetch and parse the
secrets itself.

You can follow the instructions in the proc_wrapper
[README.md](https://github.com/CloudReactor/cloudreactor-procwrapper/blob/main/README.md) if setting up
your project yourself, or you can use an [example project](https://github.com/CloudReactor/cloudreactor-python-ecs-quickstart)
which has deployment to ECS Fargate with CloudReactor management setup.

Let's assume you want to deploy a task that reads the database to the
"production" CloudReactor Run Environment. Then, in the
`deploy_config/vars/production.yml` file (if it doesn't exist, copy
from `deploy_config/vars/example.yml`), add the `env_locations` property:

      default_env_task_config:
        ...
        wrapper:
          ...
          env_locations:
            - arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/{{env}}/db.env-BHyuR

Then the environment variables DB_HOST, DB_PORT, DB_USERNAME, and DB_PASSWORD
will be populated at runtime by the proc_wrapper module.

You can also remove the suffix (in the above example, `-BHyuR`) from your secret
ARNs if you want your Task to fetch the latest value of your secret, instead of
a pinned value.

Step 4: Finally, ensure that your program runs with the IAM role you set up in
step 2.
If running in ECS, you'll want to use the role as the "Task Role". If you are starting from
an example project, add the `role_arn` property in
`deploy_config/vars/production.yml`:

    default_env_task_config:
      ...
      ecs:
        ...
        task:
          ...
          role_arn: "arn:aws:iam::012345678901:role/myapp-task-role-production"

If running in EC2, ensure the EC2 instances you run your program in have the instance role you created in step 2.
See the official AWS documentation [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
for details.

Step 5: Deploy your project and the proc_wrapper module should be able to fetch
and parse your secrets, and inject them into your program's environment. It's up
to your program to read the environment variables and configure things from
there.

---

## Modifying .gitignore in quick start projects

Once you figure out which files to encrypt, comment out the lines in
`.gitignore` that ignore secret files, since you'll be checking them in encrypted:

    # Comment out to enable deployment secrets to be committed encrypted,
    # if deploying with Docker:
    # deploy/deploy.env
    # deploy/deploy.*.env

    # Comment out to enable deployment secrets to be committed encrypted
    # deploy_config/vars/*.yml

    # Comment out to enable runtime secrets to be committed encrypted
    # deploy_config/files/.env.*

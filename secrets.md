---
layout: default
title: Secret management
nav_order: 6
---

# Secret management
{: .no_toc}

If you forked or cloned an example CloudReactor quick start project, you most likely added sensitive data to per-environment files, which you don't want to commit in plaintext. The default configuration assumes you just keep those secret files locally and don't commit them -- thus they are excluded in .gitignore.

However, for sharing with a team or to have the secrets in source control for backup/history reasons, it's better to check the secret files in, but encrypted. Alternatively, you can get secrets from AWS at runtime.

Here, we'll cover the management of 2 types of secrets:
* Deployment secrets: these are secrets needed to deploy your task, but are not required while the task is running. For example, to deploy tasks using CloudReactor, you add AWS keys to the file `deploy/docker_deploy.env`, and your CloudReactor API key to the file `deploy/vars/<environment>.yml` (where "<environment>" is the name of a Run Environment in CloudReactor).
* Runtime secrets: these are secrets that your task uses while it is running. Examples are database passwords and API keys for 3rd party services your task uses. In the example projects, the file `deploy/files/.env.<environment>` contains runtime secrets. In the python example project,
python tasks read this file at runtime using the [python-dotenv](https://github.com/theskumar/python-dotenv) library.

Three methods of managing secrets that we'll cover are:

1. TOC
{:toc}

---

## Ansible Vault

One option for managing either deployment or runtime secrets is to use
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
which is well integrated with ansible. Ansible is used to deploy the
example projects, and it will transparently decrypt files encrypted with
ansible-vault when copying them.

In the example projects, to use ansible-vault to encrypt your deployment secrets, change your directory to `/deploy/vars` and run:

    ansible-vault encrypt [environment].yml

ansible-vault will prompt for a password, then encrypt the file. To edit it:

    ansible-vault edit [environment].yml

Next, change the deployment script `deploy.sh` to get the encryption password,
either from user input, or an external file or script. Detailed instructions
are in `deploy.sh`. You can also modify `deploy/ansible.cfg` to specify an
external file tha contains the encryption password. See this
[tutorial](https://www.digitalocean.com/community/tutorials/how-to-use-vault-to-protect-sensitive-ansible-data-on-ubuntu-16-04) for more
details.

You can also use Ansible Vault to encrypt runtime secrets, by following the
steps above for `deploy/files/.env.[environment]`. However, this has the
drawback that the secrets will be in plaintext in your container image.
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

If you set up git-crypt, uncomment the lines in `.gitignore` that ignore
secret files, since they will be encrypted in the repository (see below).
Also uncomment the lines in `.gitattributes` that specify which files to encrypt.

The disadvantage of using git-crypt is that if the machine or disk that
contains these secret files is compromised, those secrets can be exposed.

---

## Runtime secrets with AWS Secrets Manager

If you are running your tasks in AWS, one option for runtime secrets
is to store them in AWS Secrets Manager.

Step 1: Log into [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) in
the AWS Console. Create a secrets object with key/value pairs for each resource
you want to store secrets for.

For example, you could create a secrets object named "myorg/myapp/production/db" that contains
the following keys:

    - host
    - port
    - db_name
    - username
    - password

After storing the secret, copy the ARN of the secret. It should look like:

    arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR

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

Once you've created the role, record the ARN which should look like:

    arn:aws:iam::012345678901:role/myapp-task-role-production

which we will use in step 4.

Step 3: Now we can make those secrets available to your tasks. You can have your code fetch these
secrets directly from AWS Secrets Manager, and parse them, or you can use the
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper)
module fetch, parse, and populate them into your environment before your program starts.
That way your program doesn't need to fetch and parse the secrets itself.

You can follow the instructions in the proc_wrapper
[README.md](https://github.com/CloudReactor/cloudreactor-procwrapper/blob/main/README.md) if setting up
your project yourself, or you can use an [example project](https://github.com/CloudReactor/cloudreactor-ecs-quickstart)
which has deployment to ECS Fargate with CloudReactor management setup.

Let's assume you are using an example project, and that you want to deploy a task that reads the
database to the "production" CloudReactor Run Environment.

Then, in the `deploy/vars/production.yml` file (if it doesn't exist, copy
from `deploy/vars/example.yml`), add the following block under "default_env_task_config"

    default_env_task_config:
      env:
        AWS_SM_DB_HOST_FOR_PROC_WRAPPER_TO_RESOLVE: "arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR|JP:$.host"
        AWS_SM_DB_PORT_FOR_PROC_WRAPPER_TO_RESOLVE: "arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR|JP:$.port"
        AWS_SM_DB_NAME_FOR_PROC_WRAPPER_TO_RESOLVE: "arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR|JP:$.db_name"
        AWS_SM_DB_USERNAME_FOR_PROC_WRAPPER_TO_RESOLVE: "arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR|JP:$.username"
        AWS_SM_DB_PASSWORD_FOR_PROC_WRAPPER_TO_RESOLVE: "arn:aws:secretsmanager:us-west-2:012345678901:secret:myorg/myapp/production/db-BHyuR|JP:$.password"

Then the environment variables DB_HOST, DB_PORT, DB_USERNAME, and DB_PASSWORD
will be populated at runtime.

You can also remove the suffix (in the above example, "-BHyuR") from your secret ARNs if you want your task
to fetch the latest value of your secret, instead of a fixed value.

Step 4: Finally, ensure that your program runs with the IAM role you set up in step 2.
If running in ECS, you'll want to use the role as the "Task Role". If you are starting from
an example project, uncomment/edit the following lines in `deploy/vars/production.yml`:

    default_env_task_config:
      command: "python src/task_1.py"
      ecs:
        task:
          role_arn: "arn:aws:iam::012345678901:role/myapp-task-role-production"

If running in EC2, ensure the EC2 instances you run your program in have the instance role you created in step 2.
See the official AWS documentation [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
for details.

Step 5: Deploy your project and the proc_wrapper module should be able to fetch and parse your secrets,
and inject them into your program's environment. It's up to your program to read the
environment variables and configure things from there.

---

## Modifying .gitignore

Once you figure out which files to encrypt, comment out the lines in
`.gitignore` that ignore secret files, since you'll be checking them in encrypted:

    # Comment out to enable deployment secrets to be committed encrypted,
    # if deploying with Docker:
    # deploy/docker_deploy.env
    # deploy/docker_deploy.*.env

    # Comment out to enable deployment secrets to be committed encrypted
    # deploy/vars/*.yml

    # Comment out to enable runtime secrets to be committed encrypted
    # deploy/files/.env.*

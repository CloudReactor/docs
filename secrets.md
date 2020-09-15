---
layout: default
title: Secret management
nav_order: 6
---

# Secret management
{: .no_toc}

If you followed the quick start guide, you most likely added sensitive data to per-environment files, which you don't want to commit in plaintext. The default configuration assumes you just keep those secret files locally and don't commit them -- thus they are excluded in .gitignore.

However, for sharing with a team or to have the secrets in source control for backup/history reasons, it's better to check the secret files in, but encrypted. Alternatively, you can get secrets from AWS at runtime.

Here, we'll cover the management of 2 types of secrets:
* Deployment secrets: these are secrets needed to deploy your task, but are not required while the task is running. For example, to deploy tasks using CloudReactor, you add AWS keys to the file `/deploy/docker_deploy.env`, and your CloudReactor API key to the file `/deploy/vars/[prod].yml` (where "prod" is the name of a run environment in CloudReactor).
* Runtime secrets: these are secrets that your task uses while it is running. Examples are database passwords and API keys for 3rd party services your task uses. In this example project, the file `deploy/files/.env.<environment>` contains runtime secrets. The python tasks read this file at runtime using the [python-dotenv](https://github.com/theskumar/python-dotenv) library.

Three methods of managing secrets that we'll cover are:

1. TOC
{:toc}

---

## Runtime secrets with AWS Secrets Manager

One option for runtime secrets is to store them in AWS Secrets Manager.

1. Log into [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) in
the AWS Console. Create a secrets object with key/value pairs for each resource
you want to store secrets for.

    For example, you could create a secrets object called "postgres" that contains
    the following keys:
    - dbname
    - username
    - password
    - host
    - port

    You'll also want to create a secrets object called "PROC_WRAPPER_API_KEY".
    This should be a plaintext (i.e not key/value pair) secret. The content
    of this secret should just be your CloudReactor API key.

2. Now we can make those secrets available to your tasks. Lets say you want
those secrets to be available to tasks deployed to your "prod" ECS cluster /
CloudReactor run environment.

    Then, in the `deploy/vars/prod.yml` file (if it doesn't exist, copy / rename
    from `deploy/vars/example.yml`), add the following block.

    Note that each entry in `secrets:` consists of a name and valueFrom, where
    the name is chosen by you, and valueFrom is the full ARN from AWS Secrets
    Manager.

    ```
    # /deploy/vars/[prod].yml -- or other .yml file
    default_env_task_config: &default_env_task_config
      ecs:
        extra_main_container_properties:
          secrets:
            - name: PROC_WRAPPER_API_KEY
              valueFrom: "arn:aws:secretsmanager:<region>:<account_id>:secret:CloudReactor/prod/common/cloudreactor_api_key-xxx123"
            - name: POSTGRES_SECRETS
              valueFrom: "arn:aws:secretsmanager:<region>:<account_id>:secret:CloudReactor/prod/common/postgres-xxx123"
    ```

    See `deploy/vars/common.yml` for an example: just remove comments and edit

3. Ensure your local (dev) secrets object has the same shape as your AWS
Secrets Manager secrets. This allows you to parse the secrets object in the 
same way in your task source code.

    Open `/deploy/files/.env.dev` and create an entry with the same name as
    you gave the secrets object in the previous step; for example,
    `POSTGRES_SECRETS`.

    Add settings for local development to this secret in a json format -- this
    will mimic the shape of the secrets in AWS Secrets Manager. For example:

    ```
    # /deploy/vars/.env.dev
    POSTGRES_SECRETS={"dbname": "mydatabasename", "username": "postgres",
    "password": "mysecurepassword",
    "host": "databaseIdentifier.abcde12345.us-east-1.rds.amazonaws.com",
    "port": "5432"}
    ```

4. Finally, in your task source code, e.g. "new_task.py":
    - import json and load_dotenv from dotenv:

    ```python
    import json
    dotenv import load_dotenv
    ```
    -  wherever you want to inject / use the secrets object:

    ```python
    postgres_secrets = os.environ.get('POSTGRES_SECRETS')
    # convert secrets string to dictionary
    pg_secrets_dict = json.loads(postgres_secrets)
    db_name = pg_secrets_dict['dbname']
    db_user = pg_secrets_dict['username']
    db_password = pg_secrets_dict['password']
    db_host = pg_secrets_dict['host']
    # set default port to 5432
    db_port = int(pg_secrets_dict.get('port', '5432'))
    ```
    - now your secrets, whether in local development or e.g. prod (as here),
    are available in the `db_name`, `db_user`, etc. variables.

---

## git-crypt

Another option for encryption is 
[git crypt](https://github.com/AGWA/git-crypt), 
which encrypt secrets when they are committed to Git. 
However, this leaves secrets unencrypted in the filesystem where the 
repository is checked out. That can be advantage as it is easier
to edit and search secret files.

If you set up git-crypt, uncomment the lines in `.gitignore` that ignore 
secret files, since they will be encrypted in the repository. Also uncomment
the lines in `.gitattributes` that specify which files to encrypt.

The disadvantage of using git-crypt is that if the machine or disk that
contains these secret files is compromised, those secrets can be exposed.

---

## Ansible Vault

One option for managing either deployment or runtime secrets is to use 
[Ansible Vault](https://docs.ansible.com/ansible/latest/user_guide/vault.html)
which is well integrated with ansible. Ansible is used to deploy this
example project, and it will transparently decrypt files encrypted with 
ansible-vault when copying them.

To use ansible-vault to encrypt your deployment secrets, change your directory to /deploy/vars and run:

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
steps above for `deploy/files/[environment].yml`. However, this has the 
drawback that the secrets will be in plaintext in your container image.
For most applications, that is secure enough because [AWS ECR stores images
encrypted](https://aws.amazon.com/ecr/faqs/) and it is assumed the server
you deploy from (which builds and caches Docker images) is secure.

Once you figure out which files to encrypte, uncomment the lines in 
`.gitignore` that ignore secret files, since you'll be checking them in encrypted.

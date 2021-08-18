---
layout: default
title: Troubleshooting
nav_order: 9
---

# Troubleshooting

## Debugging deployment issues

In a bash shell, run:

    ./deploy_shell.sh <environment>

In a Windows command prompt, run:

    deploy_shell.cmd <environment>

These commands will take you to a bash shell inside the deployer Docker container where you can re-run the deployment script with `./cr_deploy.sh` and inspect the files it produces in the `build/` directory.

## Common errors

* When deploying, you see

      AnsibleFilterError: |combine expects dictionaries, got None"}

This may be caused by defining a property like a task under `task_name_to_config`
in `deploy/vars/common.yml`:

    task_name_to_config:
       some_task:
       another_task:
         schedule: cron(9 15 * * ? *)

`some_task` is missing a dictionary value so the corrected version is:

    task_name_to_config:
       some_task: {}
       another_task:
         schedule: cron(9 15 * * ? *)


* If you are using Native Deployment on Mac OS X, and see the message:

      Abort trap: 6

  follow the steps on [https://dbaontap.com/2019/11/11/python-abort-trap-6-fix-after-catalina-update/](https://dbaontap.com/2019/11/11/python-abort-trap-6-fix-after-catalina-update/) to fix, possibly, replacing 1.0.2t with 1.0.2s or whatever you have in `/usr/local/Cellar/openssl/1.0.2s/lib`.

* If you are using Native Deployment on Mac OS X, and see the error `ERROR:root:code for hash md5 was not found.`, you may need to
run the command

      brew switch openssl 1.0.2s

    See [StackOverflow](https://stackoverflow.com/questions/59269208/errorrootcode-for-hash-md5-was-not-found-when-using-any-hg-mercurial-command) for more details

## I need help!

If you encounter any other problems, feel free to reach out to us at support@cloudreactor.io!

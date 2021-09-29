---
layout: default
title: Home
nav_order: 1
---
# Easy task and workflow monitoring & automation
{: .no_toc }
{: .fs-10 }


Monitor, manage, and orchestrate on-demand tasks, scheduled tasks, and
long-running services with CloudReactor's easy-to-use dashboard.
Deploy tasks with a single command to AWS ECS Fargate.

Use this one-page guide to get up and running with key CloudReactor features.
See additional pages in left nav bar for more features and advanced topics.
{: .fs-4 .fw-300 }

[Get started](#getting-started){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 } [Learn more about CloudReactor](/cloudreactor.html){:target="_blank"}{: .btn .fs-5 .mb-4 .mb-md-0 }

## Table of contents
{: .no_toc .text-delta .mt-6 }

1. TOC
{:toc}

---

## Getting Started

### Decide on an integration level

First, decide on how much effort you're willing to invest in return for
the features you want. The following table shows the tradeoffs:


| Integration Level | Monitored |         Managed        | Requires Deployment Changes | Requires CloudReactor access grant  |
|-------------------|:---------:|:----------------------:|:---------------------------:|:-----------------------------------:|
| Offline           |     No    |           No           |              Minimal             |                  No                 |
| Passive           |    Yes    |        Stop only       |           Minimal           |                  No                 |
| Simple            |    Yes    | Standard, for Fargate Tasks |           Minimal           |                 Yes                 |
| Full              |    Yes    | Enhanced, for Fargate Tasks |             Yes             |                 Yes                 |

All integration levels gain these features provided by
[proc_wrapper](https://github.com/CloudReactor/cloudreactor-procwrapper):

* Retries and time limits on wrapped processes
* Secret injection from AWS Secrets Manager and extraction into the
process environment

The `Monitored` column in the table above refers to the ability
to see the current status of a Task and receiving notifications
when the Task has a problem. proc_wrapper reports heartbeats
to CloudReactor when it runs your code, and you can view the last
heartbeat time in the CloudReactor dashboard. Your process can also
report status information like success counts, failure counts,
status messages, etc. which and are displayed in the CloudReactor dashboard.
You can also configure CloudReactor to send notifications if your Task
fails, takes too long to run, exits early, or if your Task isn't run on schedule.
Monitored Tasks also have their execution history stored in CloudReactor
so you can see what happened in previous executions and view trends.
To enable monitoring, the following are required:

* A free CloudReactor account
* Minimal changes to your deployment process to set some environment
variables, and to wrap your code with proc_wrapper

The *Passive* integration level provides monitoring
with a minimal amount of configuration and deployment changes. It is also
compatible with any execution method that uses a command-line in Linux,
Windows, or any operating system with python 3.6 or higher installed.

The *Simple* integration level allows CloudReactor to start Tasks that are run
in AWS ECS, and orchestrate them in Workflows. To enable these capabilities,
you need to install the
[CloudReactor CloudFormation Template](https://github.com/CloudReactor/aws-role-template).
The template can be installed manually or by using
the [AWS Setup Wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard)
which guides you through the steps. The wizard can also setup all AWS
infrastructure needed to run tasks in ECS Fargate.

In the guide below, we'll assume you chose the *Passive* or *Simple*
integration level, as they require only minimal changes to your application.

### Create a CloudReactor account

First, sign up for a
[CloudReactor account](https://dash.cloudreactor.io/signup).
No credit card is required, and there is a free plan available.

### Setup a Run Environment

If you chose the *Passive* integration level, the next step is to
create a Run Environment. Navigate to
[Run Environments](https://dash.cloudreactor.io/run_environments)
and select the `Create new Run Environment ...` button.
Give your Run Environment a name, leave the
`Enable CloudReactor to manage Tasks in AWS ECS` checkbox unchecked
since you are using the *Passive* option, then hit the `Save` button.

If, instead, you chose the *Simple* integration level,
follow the instructions for
[granting CloudReactor Task management access](/cloudreactor_access.html),
then proceed with the steps below.

### Create a CloudReactor API Key
{: #create-api-key }

Now, create an API key your Task will use to authenticate with CloudReactor.
Select your username in the upper right corner of the dashboard,
then in the popup menu, select `API Keys`. On the following page, hit the
button labeled `Add a new API key ...`. On the following page,

1. Give your API key a name.
2. Choose the Run Environment you just created. You may also choose `Any`
if you want to reuse the API key in different Run Environments.
3. Select `Developer` Access Level of as your program will create a Task
in CloudReactor the first time it is run.
4. Hit the `Save` button.
5. You will be taken back a list of your API keys. On the row with the key
you just created, use the copy button to copy the API key to your clipboard.

### Set environment variables

Assuming your code is started via a command line, proc_wrapper reads its
configuration from environment variables. The minimum set for an auto-created,
passive Task is:

* `PROC_WRAPPER_TASK_NAME` - set to whatever you want to name the Task in
CloudReactor
* `PROC_WRAPPER_API_KEY` - set to the value of the API key you created above
* `PROC_WRAPPER_AUTO_CREATE_TASK=TRUE`
* `PROC_WRAPPER_AUTO_CREATE_TASK_RUN_ENVIRONMENT_NAME` - set to the name of the Run Environment you created above
* `PROC_WRAPPER_TASK_IS_PASSIVE=TRUE`
* `PROC_WRAPPER_SEND_RUNTIME_METADATA=TRUE`

Arrange to have these environment variables set before the command line is
executed.

### Install the wrapper and modify your command line

Finally, just prepend a call to the CloudReactor proc_wrapper standalone
executable or python module to your existing command line. For example,
if your code was started with the command line:

```
    java -jar app.jar
```

you would instead use the command line:

```
    ./proc_wrapper java -jar app.jar
```

after downloading the standalone executable *proc_wrapper* from the
[releases](https://github.com/CloudReactor/cloudreactor-procwrapper/releases)
page. Standalone executables are available for both 64-bit Linux and
Windows.

Or, if you have python 3.6 or higher installed in your environment, use the
command line:

```
    python -m proc_wrapper java -jar app.jar
```

after installing the
[cloudreactor-procwrapper](https://pypi.org/project/cloudreactor-procwrapper/)
python module.

### View the Task in CloudReactor

Now, after the first time your Task executes, it will show up in the
CloudReactor dashboard! While your Task is running, you can stop it using the
Stop button in the dashboard.

You'll be able see:

1. The current status of your Task (running or stopped), and the last time
it sent a heartbeat
2. A history of previous executions of your Task, including start times,
durations, and exit codes

### Set up notifications (optional)

Now that your Task is monitored by CloudReactor, you can set up notifications
via PagerDuty or email, that trigger when your Task fails, succeeds,
times out, stops sending heartbeats, or doesn't run as scheduled.
See the [notifications guide](notifications.html) for instructions.

---

## Improvements

* To avoid leaking secrets (passwords, API keys, etc.), see the guide on
[secret management](/secrets.md)
* To allow CloudReactor to start Tasks on your behalf, thus enabling
manual starting in the dashboard, and Workflows, follow the
[full integration level](full_integration.html)
instructions

---

## Contact us

Hopefully, this guide has helped you add monitoring to your tasks, with
a minimum of changes. Feel free to reach out to us at support@cloudreactor.io
to upgrade to a paid account with more limits, or if you have any questions
or issues!

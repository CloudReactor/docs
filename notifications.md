---
layout: default
title: Notifications
nav_order: 8
---

# Notifications
{: .no_toc}

You can set up an Alert Method if you want to be notified when Task Executions
succeed, fail, timeout, stop sending heartbeats, or don't start as scheduled.
Additionally, for long-running services, you can configure CloudReactor to
send notifications if the number of Task Executions providing the service
goes below a minimum concurrency threshold.

Currently, we support two notification types:

* PagerDuty events. [PagerDuty](https://pagerduty.com) is powerful because it supports many ways
of forwarding events once it receives them, for example sending an SMS message, sending an email, or sending a message to a Slack channel.
* Email messages

To have CloudReactor send Task and Workflow events to PagerDuty, follow these general steps:

1. TOC
{:toc}

---

## Create a PagerDuty Profile

First, create a PagerDuty Profile that contains configuration on how to
connect to your PagerDuty account.

1. If you haven't already, create an account with PagerDuty
2. Login to your PagerDuty account
3. Choose the menu option `Configuration ... Service Directory`. On that page
select the `+ New Service` button to add a new Service.
4. Name your Service anything you desire. Under Integration Settings ... Integration Type, select "Use our API directly" and ensure that
`Events API v2` is selected. If desired, adjust the Incident Settings.
Finally, select the `Add Service` button at the page.
5. On the next page you'll see your newly created Service along with an Integration Key. Copy the Integration Key for use in CloudReactor.
6. In the [CloudReactor](https://dash.cloudreactor.io/) website,
select your username in the top right corner to reveal a dropdown menu. Select `PagerDuty profiles`.
7. On the next page, select the `Add PagerDuty Profile` button.
8. You'll be taken to a form that lets you create the profile. Enter a
name, and optionally a description. In the `Integration key` field,  paste in the Integration Key you copied from PagerDuty in step 5. For the
`Default event severity` field, choose the PagerDuty event severity to
use for events with no explicit severity. By default, this is `Error`.
9. Select the `Save` button to save the profile

## Create an Email Notification Profile

Alternatively, you can get notifications via email. In that case:

1. In the [CloudReactor](https://dash.cloudreactor.io/) website,
select your username in the top right corner to reveal a dropdown menu.
Select `Email Notification profiles`.
2. On the next page, select the `Add Email Notification Profile` button.
3. You'll be taken to a form that lets you create the profile. Enter a
name, and optionally a description.
4. Optionally, choose a Run Environment to limit usage of the profile to
Tasks running with the selected Run Environment
5. Add email addresses to the To, CC, and BCC fields
9. Select the `Save` button to save the profile

## Create an Alert Method

Next, create an Alert Method that will link to the PagerDuty Profile or
Email Notification Profile you just created.

1. In the [CloudReactor](https://dash.cloudreactor.io/) website,
select your username in the top right corner to reveal a dropdown menu. Select `Alert methods`.
2. On the next page, select the `Add Alert Method` button.
3. You'll be taken to a form that lets you create the Alert Method. Enter a
name, and optionally a description.
4. Leave the Enabled checkbox checked to ensure this Alert Method is fired
when events occur.
5. Check the boxes as desired to be notified on success, failure, and/or
timeout
6. Choose the desired event severity when:
    * Scheduled Task or Workflow executions don't start;
    * Tasks do not send heartbeats on time; and
    * A service goes down

For PagerDuty notifications:
1. Select the PagerDuty notification method
2. Select the PagerDuty Profile you created above.

For Email notifications:
1. Select the Email notification method
2. Select the Email Notification Profile you created above.

Finally, select the `Save` button to save the Alert Method

## Add the Alert Method to one or more Tasks or Workflows

Next, you just need to associate your Tasks or Workflow to the Alert Method
you created.

### Setting Alert Methods for Tasks

For Tasks, you can set the Alert Methods in the CloudReactor
dashboard, or during deployment.

To use the dashboard:

1. Go the list of your Tasks by going to the home page of CloudReactor
2. Click on the Task you want to set the Alert Method on
3. Click the `Alerts` tab
4. Check the checkboxes next to the Alert Methods you want to associate with the
Task
5. Hit the `Save changes` button

If you use the
[aws-ecs-cloudreactor-deployer](https://github.com/CloudReactor/aws-ecs-cloudreactor-deployer)
Docker image to deploy your project, you can also set the Alert Methods of
your Tasks during deployment. Normally, you will want to set different
Alert Methods for each Run Environment in your deployment environment
configuration file, for example `deploy/staging.yml`. In that file you can
set the alert methods for the whole deployment environment in
`default_env_task_config.alert_methods`, or per Task in
`task_name_to_env_config.<task_name>.alert_methods`.  The
`alert_methods` property should be set to a list of the name(s) of
the Alert Methods you created above.

### Setting the Alert Method for Workflows

For Workflows, you can set the Alert Methods on the CloudReactor
website:

1. Go the list of your Workflows by selecting `Workflows` in the top
navigation bar.
2. Click on the Workflow you want to set the Alert Method on
3. On the Workflow detail page, click the `Settings` tab
4. For the `Alert Methods` field, check the checkboxes next to the
Alert Methods you want to associate with the Workflow

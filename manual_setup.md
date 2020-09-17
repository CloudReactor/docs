---
layout: default
title: Manual setup
nav_order: 10
---
# Setting up AWS and CloudReactor manually

You can use this guide if, for some reason, you don't want to use the [CloudReactor & AWS Setup Wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard) (but really, you should, it's faster and easier).

Still want to go ahead with manual setup?

Broadly speaking, we'll need to:
- [Set up a cluster in AWS ECS](#set-up-a-cluster-in-aws-ecs) (this is where your tasks will be deployed to)
- [Set up AWS role permissions to allow CloudReactor to stop, start and schedule ECS tasks](#set-aws-role-permissions-to-allow-cloudreactor-to-stop-start-and-schedule-ecs-tasks)
- [Configure CloudReactor with AWS ECS and AWS role settings](#configure-cloudreactor-with-aws-ecs-and-aws-role-settings)

Configuration will require some keys and other parameters to be entered. We'll note anything you need to record in  <span style="color: red">red</span> -- open up a text file to hold those variables as we go along.

---

### Set up a cluster in AWS ECS
AWS provides a wizard that creates an ECS cluster in just a few steps. This is appropriate if you want to get started quickly. The wizard can optionally create a new VPC and new public subnets on that VPN -- but cannot create private subnets.

Note that you may use other methods like [CloudFormation templates](https://github.com/aws-samples/ecs-refarch-cloudformation) or [Terraform templates](https://github.com/turnerlabs/terraform-ecs-fargate).

We will continue with the AWS wizard. Note that when running the wizard, your user needs to have at least the permissions listed under ["Amazon ECS First Run Wizard Permissions" here](https://docs.aws.amazon.com/AmazonECS/latest/userguide/security_iam_id-based-policy-examples.html#first-run-permissions).

The steps to run the wizard are:

1. Go to https://aws.amazon.com/ecs/getting-started/
2. Click the `ECS console walkthrough` button (log into AWS if necessary)
3. Change the region to your default AWS region
4. Click the `Get started` button
5. Choose the `nginx` container image and click the `Next` button
6. On the next page, the defaults are sufficient, so hit `Next` again
7. On the next page, name your cluster the desired name of your deployment environment -- for example `staging`. If you have an existing VPC and subnets you want to use to run your tasks, you can select them here. Otherwise, the console will create a new VPC and subnets for you.
After entering your desired cluster name, hit `Next` again.
8. On the next and final page, review your settings and hit the `Create` button. You'll see the status of the created resources on the next page. <span style="color: red">If you didn't choose existing subnets, record the subnet IDs</span>

AWS will create:
- A cluster named as you chose on step 7 above.
- A VPC named `ECS [cluster name] - VPC`
- 2 subnets in the VPC named `ECS [cluster name] - Public Subnet 1` and `ECS [cluster name] - Public Subnet 2`.
You can see these in VPC .. Subnets. Note that if you used the wizard to create a new VPC and subnets for you, these subnets will be public; if you want to use [private subnets](docs/networking.md), you'll have to create your own. If you haven't already, <span style="color: red">record the Subnet IDs</span>
- A security group named `ECS staging - ECS Security Group` in the VPC. You can find it in `VPC .. Security Groups`. <span style="color: red">Record the Security Group ID</span>
- Once you've recorded the Subnet IDs and Security Group IDs, under "ECS resource creation", you'll see `Cluster [the name of the cluster you created]`. Clicking this link will take you to the cluster's details page; <span style="color: red">record the `Cluster ARN`</span> you see here.


{: .mt-5}
- [x] ECS cluster created!

---

### Set AWS role permissions to allow CloudReactor to stop, start and schedule ECS tasks
To have CloudReactor manage your tasks in your AWS environment, you'll need to give CloudReactor permissions in AWS to run tasks, schedule tasks, create services, and trigger Workflows by deploying the [CloudReactor AWS CloudFormation template](https://github.com/CloudReactor/aws-role-template), named `cloudreactor-aws-role-template.json`.

Follow the instructions in the [README.md](https://github.com/CloudReactor/aws-role-template/blob/master/README.md#allowing-cloudreactor-to-manage-your-tasks), in the section "Allowing CloudReactor to manage your tasks".

<span style="color: red">Be sure to record the ```ExternalID```, ```CloudreactorRoleARN```, ```TaskExecutionRoleARN```, ```WorkflowStarterARN```, and ```WorkflowStarterAccessKey``` values.</span>


{: .mt-5}
- [x] AWS role permissions created for CloudReactor!


---

### Configure CloudReactor with AWS ECS and AWS role settings
Sign up for a CloudReactor account at [https://dash.cloudreactor.io/signup](https://dash.cloudreactor.io/signup), and login.

We'll create a Run Environment in CloudReactor. A Run Environment contains settings that tell CloudReactor how to run tasks in AWS.

1. Click on "Run Environments", then "Add Environment"
2. Name your environment (e.g. "staging", "production"). You may want to keep the name in all lowercase letters without spaces or symbols besides "-" and "_", so that filenames and command-lines you'll use later will be sane. <span style="color: red">Note the exact name of your Run Environment</span>, as you'll need this later.
3. Fill in your AWS account ID and default region. Your AWS account ID is a 12-digit number that you can find by clicking "Support" then "Support Center". For default region, select the region that you want CloudReactor to run tasks / workflows in (e.g.`us-west-2`).
4. For `Assumable Role ARN` fill in the value of `CloudreactorRoleARN` from the output of the CloudFormation stack.
5. For `External ID`, use the same External ID you entered when you created the CloudFormation stack.
6. For `Workflow Starter Lambda ARN`, fill in the value of `WorkflowStarterARN` from the output of the CloudFormation stack.
7. For `Workflow Starter Access Key`, fill in the value of `WorkflowStarterAccessKey` from the output of the CloudFormation stack.
8. Add the subnets and security group created by the ECS getting started wizard above
9. Under AWS ECS Settings, choose a `Default Launch Type` of `Fargate` and check FARGATE under Supported Launch Types.
10. For `Default Cluster ARN`, fill in the `Cluster ARN` of the ECS cluster you created above
11. For `Default Execution Role` and `Default Task Role`, fill in the value of `TaskExecutionRoleARN` from the output of the CloudFormation stack.
12. Click on the `Save` button


{: .mt-5}
- [x] CloudReactor configured to run tasks in ECS!

---

### Next steps

At this point, AWS has the required infrastructure to run tasks on ECS, and CloudReactor and AWS can talk to each other.

Now, we can [deploy tasks and have them managed by CloudReactor](/index.md).
---
layout: default
title: Networking
nav_order: 9
---
# Fargate Networking

Your tasks will need to be able to access the internet in order to pull its Docker image from ECR, and to communicate with CloudReactor (e.g. stop / start, sending metrics). Internet access will also be required to access external resources, such as Snowflake or an API.

There are a couple of common configurations for enabling internet access:
- Deploy tasks to a private subnet(s), connected to one or more public subnets. The private subnets can access the internet via a NAT gateway in the public subnets. This is recommended as a best practice.
- Deploy tasks to a public subnet, and provide the task with a publicly accessible IP address

## Setting up networking using the CloudReactor AWS Setup Wizard

The [CloudReactor AWS Setup Wizard](https://github.com/CloudReactor/cloudreactor-aws-setup-wizard) can automatically create a VPC, between 1-3 private / public subnet pairs, with a NAT gateway to enable internet access from the private subnets, and security groups. It's probably the simplest way to get set up quickly. It can also help you configure your AWS environment so that CloudReactor
can manage your tasks.

## Setting up networking manually

See [https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-public-private-vpc.html](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/create-public-private-vpc.html) for step-by-step instructions on creating private and public subnets.

Follow the AWS VPC wizard if you don’t have a VPC setup at all; this will create the private & public subnets and a NAT gateway for you.

Ultimately, you may want to review the following steps to confirm everything is set up as required (and to set up any required missing infrastructure):
- Create an Elastic IP: AWS VPC dashboard > "Elastic IPs" > "Allocate new address".
- Create a NAT gateway (if none exist): AWS VPC dashboard > “NAT gateways” > "Create NAT Gateway" > select the public subnet(s) and Elastic IP address created in the previous step.
- Create an internet gateway (if none exist): AWS VPC dashboard > “Internet gateway” > “Create Internet Gateway”.
- Ensure your route tables are setup correctly. AWS VPC dashboard > “Route tables”. You should have one route table set up and assigned to public subnets, and a second assigned to private subnets
  - Your public subnet route table should have a destination “0.0.0.0/0” and target "igw-xxxxx" (where igw-xxxxx is an AWS internet gateway)
  - Your private subnet route table should have a destination “0.0.0.0/0” and target "nat-xxxxx" (where nat-xxxxx is an AWS NAT Gateway, such as just created)
  - Note that both subnets’ route tables will have a target “local”: this is added by default and enables communication within the VPC.
  - You can assign route tables to subnets from this dashboard, or via the AWS VPC dashboard > "Subnets", selecting the subnet and editing the route table associated with it.

### Deploying a task to a specific subnet

Once subnets are created and configured successfully, go to the [CloudReactor dashboard](https://dash.cloudreactor.io/run_environments) to add them to a Run Environment. Select a Run Environment, and under "default subnets", add the private subnets that you just created.

Or, if you only created public subnets and want to deploy your tasks there, add the public subnet IDs instead. Note that in this instance, you'll need to assign your tasks with a public IP address. To do this in an example project, open
`/deploy/vars/common.yml` and uncomment the line `assign_public_ip: True`.

Now, all tasks deployed to that Run Environment will use the default subnets set above.

You can optionally over-ride the default subnets setting if required (if unsure, feel free to ignore!). In priority order (highest to lowest):
- via `/deploy/vars/[run_environment].yml`: for example, a task called `new_task` to be deployed to the `production` environment could have a specific subnet over-ride by adding a block for the task in `production.yml`. See `/deploy/vars/example.yml` for an example.
- via `/deploy/vars/common.yml`: again, you could add a configuration block here that includes a subnet definition.

## Security Groups

You must assign your Fargate tasks one or more security groups. To run in Fargate and communicate with CloudReactor, each task must be assigned a security group that allows all outbound access (0.0.0.0.0/0) over TCP. Inbound access is not required unless your task handles outside requests. Generally, the security group you use should be assigned to the VPC that hosts the subnets the task will run in.

The ECS Getting Started wizard will create a default security group attached to the VPC it creates. The default security group is fine to use with CloudReactor managed tasks.

## More info

For a good overview of networking in Fargate, see [Fargate Networking 101](https://cloudonaut.io/fargate-networking-101/).

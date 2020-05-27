---
layout: default
title: Networking
nav_order: 7
---
# Networking

The ECS Getting Started wizard referenced in the [Getting Started guide](./#getting-started) creates public subnets which can be reached from outside your VPC. However, best practice for security's sake is to run your tasks on private subnets whenever possible.

## Creating Private Subnets

If you want to use private subnets, create them if you don't already have existing ones, and ensure that each subnet has a [NAT gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) attached to it, so that it can pull Docker images from ECR and communicate with CloudReactor.  

For a good overview of networking in Fargate, see [Fargate Networking 101](https://cloudonaut.io/fargate-networking-101/).
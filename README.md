# ansible-aws-provision

This repository is intended as a replacement for people who have historically
relied on [Vagrant](https://www.vagrantup.com/) to test IaC code on Virtual
machines.

# Motivation

There are three main reasons why creating a replacement for Vagrant became important. the first is that Vagrant is clearly in the process of dying a slow death:

* Hashicorp, the developer of Vagrant, is not spending much development effort on it.
* Containers have taken over the role of Vagrant in many cases.
* Most Vagrant plugins are not maintained at all, or are poorly maintained and require workarounds and tricks to be useful.
* Introduction of ARM64-based systems (e.g. Macbook M1) make it impossible to rely on the Virtualbox provider, which is the only one that can be reasonably expected to work with minimal issues.

When you have to work with baremetal or virtual machines containers are not an
option. And if Vagrant is not an option, then your remaining options all suck.
This repository attempts to be a "good enough" solution to replace Vagrant.

# How does the script work?

This script works in three distinct phases, just like Vagrant:

1. Provision the VM
1. Wait for SSH to become responsive
1. Run provisioners
    1. Copy files over
    1. Run commands

If any of the provioning steps fail then the new EC2 instance is automatically
destroyed.

# The AWS EC2 inventory

This repository contains and AWS EC2 inventory called
[aws_ec2.yml](aws_ec2.yml). It is not enabled ansible.cfg by default.

# Prerequisites

## Software packages

You need to have Ansible, boto3 and botocore installed. On RHEL derivatives like Fedora you can use dnf:

    dnf install ansible python3-boto3 python3-botocore

On other systems you can use pip:

    pip3 install ansible boto3 botocore

## AWS setup

You need to have the following environment variables set:

```
AWS_DEFAULT_REGION=<region>
AWS_SECRET_ACCESS_KEY=<secret-access-key>
AWS_ACCESS_KEY_ID=<access-key-id>
```

The AWS access keys should belong to a user that has the necessary privileges
to create and destroy AWS EC2 instances.

# Setting this up in your project

We recommend cloning this repository as a Git submodule in your project's Git. For example:

    cd myproject
    git submodule add https://github.com/Puppet-Finland/ansible-aws-provision.git

Then create two configuration files in the main project directory (*not* under
ansible-aws-provision):

* *aap-project.yml*: variables related to your software/automation project
* *aap-site.yml*: variables related to your AWS environment

If you use ansible-aws-provision in a public project then you should only version *aap-project.yml*.
There are sample configuration files under the [samples](samples) directory.

# Usage

Once everything is prepared you should be able to just

    cd ansible-aws-provision
    ansible-galaxy collection install -n -f -p collections -r collections/requirements.yml
    ansible-playbook provision.yml

If provisioning failed, you can debug using

    ansible-playbook -vv provision.yml

If you provisioning worked fine in Vagrant, it should, in general, work fine in ansible-aws-provision.

# Support for automatic EC2 instance shutdown

Your EC2 costs will go through the roof easily for no reason if you forget
stuff running in there. Therefore we recommend setting up
[EC2 instance shutdown automation](https://www.puppeteers.net/blog/stop-ec2-instances-automatically-with-terraform)
to cut costs.

This Ansible code is only concerned with spinning up temporary and disposable
EC2 instances. It does not support shutting those instances down automatically.
However, it does add tags that help you identify EC2 instances that need to be
shut down, say, every night. In particular each EC2 instance gets a
"tostop=true" tag automatically. That tag is the default tag for
[terraform-aws-lambda-scheduler-stop-start](https://github.com/diodonfrost/terraform-aws-lambda-scheduler-stop-start),
a Terraform/Opentofu module that sets up everything needed to shut down EC2
instances automatically using AWS Lambda and Eventbridge. Here's a sample of
how to use the module to shut down EC2 instances with a matching tag at 19:30
UTC every night:

```
#
# Automatically stop EC2 instances every night
#
# https://github.com/diodonfrost/terraform-aws-lambda-scheduler-stop-start
#
module "stop_ec2_instance" {
  source                         = "github.com/diodonfrost/terraform-aws-lambda-scheduler-stop-start?ref=3.5.0"
  name                           = "ec2_stop"
  # See https://docs.aws.amazon.com/lambda/latest/dg/services-cloudwatchevents-expressions.html
  cloudwatch_schedule_expression = "cron(30 19 * * ? *)"
  schedule_action                = "stop"
  autoscaling_schedule           = "false"
  documentdb_schedule            = "false"
  ec2_schedule                   = "true"
  ecs_schedule                   = "false"
  rds_schedule                   = "false"
  redshift_schedule              = "false"
  cloudwatch_alarm_schedule      = "false"
  scheduler_tag                  = {
    key   = "tostop"
    value = "true"
  }
}
```

We use this code with Opentofu 1.6.0 and it works without any issues.

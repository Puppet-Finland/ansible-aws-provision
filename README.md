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

```
    cd myproject
    git submodule add https://github.com/Puppet-Finland/ansible-aws-provision.git
```

Then create a configuration file, *ansible-aws-provision.yml*, in the main project directory (not under ansible-aws-provision). Here's an example from puppet-redmine:

```
# SSH keypair name as seen by AWS
keypair_name: provision

# Region to deploy the EC2 instance to
region: eu-central-1

# Subnet to use for the EC2 instance
subnet_name: production-primary-public

# Security group to attach to the EC2 instance
security_group_name: production-standard

# VPC to use for the EC2 instance. Must contain the subnet as well or EC2
# instance creation will fail.
vpc_id: vpc-00d74e9278823eca1

# Ubuntu 22.04 x86_64
ami_id: ami-0faab6bdbac9486fb

# Value of the "Name" tag for the instance. In other words, the human-readable
# name of the EC2 instance that will be created.
instance_name: redmine

# Instance type
instance_type: t3.medium

# SSH user that has passwordless sudo privileges. Typically the default user of
# the AMI provider.
login_user: ubuntu

# Copy these files. Note that paths are evaluated from the ansible-aws-provision
# subdirectory, so remember to add a "..".
#
# Try to cherry-pick what you transfer: the playbook uses ansible.builtin.copy
# which is very slow when transfering lots of data (e.g. .git directories).
#
files:
  "../data": "/tmp/redmine"
  "../hiera.yaml": "/tmp/redmine"
  "../manifests": "/tmp/redmine"
  "../templates": "/tmp/redmine"
  "../Puppetfile": "/tmp/redmine"
  "../vagrant": "/tmp/redmine"

# Run these arbitrary provisioning commands on the EC2 instance. These 
commands:
  - "/tmp/redmine/vagrant/common.sh"
  - "/opt/puppetlabs/bin/puppet apply /tmp/redmine/vagrant/redmine.pp --modulepath=/tmp/redmine/modules"
```

The next step is to enter the ansible-aws-provision directory and install the Ansible AWS
module:

    cd ansible-aws-provision
    ansible-galaxy collection install -n -f -p collections -r collections/requirements.yml

# Usage

Once everything is prepared you should be able to just

    cd ansible-aws-provision
    ansible-playbook provision.yml

If provisioning failed, you can debug using

    ansible-playbook -vv provision.yml

If you provisioning worked fine in Vagrant, it should, in general, work fine in ansible-aws-provision.


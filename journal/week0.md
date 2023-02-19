# Week 0 — Billing and Architecture

## Required Homeworks/Tasks

### Install & Verify AWS CLI 

#### Install AWS CLI

We can install the AWS CLI by updating our **.gitpod.yml** file to include the installation task:
 
```
tasks:
  - name: aws-cli
    env:
      AWS_CLI_AUTO_PROMPT: on-partial
    init: |
      cd /workspace
      curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
      unzip awscliv2.zip
      sudo ./aws/install
      cd $THEIA_WORKSPACE_ROOT
```
#### Verify AWS CLI on Gitpod

**Add verification Image**

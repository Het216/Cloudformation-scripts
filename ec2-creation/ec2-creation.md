# EC2 Instance with CloudFormation

This template creates a simple Ubuntu EC2 instance using AWS CloudFormation. The main file is [ec2-creation.yaml](ec2-creation.yaml), and this document explains what each part does, why it is used, and how to deploy the stack from scratch.

## What This Template Does

The template creates:

1. A security group that allows SSH access on port 22.
2. An EC2 instance running Ubuntu 22.04.
3. Stack outputs for the instance ID and public IP address.

It is a minimal example, which makes it useful for learning how CloudFormation works and how to launch a basic EC2 server consistently.

## Prerequisites

Before deploying, you need:

1. An AWS account.
2. AWS CLI installed and configured.
3. A valid EC2 key pair already created in the same AWS region.
4. Permission to create EC2, security group, and CloudFormation resources.

## Template Overview

The file [ec2-creation.yaml](ec2-creation.yaml) is a CloudFormation template. CloudFormation lets you define infrastructure in code so you can create, update, and delete resources in a repeatable way.

### `AWSTemplateFormatVersion`

This tells CloudFormation which template format version is being used. It is mostly informational, but it is a standard part of the template header.

### `Description`

This is the short description shown in the AWS Console. It helps you identify the stack quickly.

## Resources Section

All infrastructure in this template is defined under `Resources`.

### Security Group: `EC2SecurityGroup`

This resource creates a security group, which acts like a virtual firewall for the EC2 instance.

Why it is used:

1. It controls which inbound traffic can reach the server.
2. It protects the instance from unwanted network access.
3. It is required so SSH can work from your machine to the EC2 instance.

#### `GroupDescription`

This is a human-readable description of the security group. Here it explains that the group allows SSH access.

#### `SecurityGroupIngress`

This defines the inbound traffic rule.

The rule in this template allows:

1. Protocol: `tcp`
2. Port: `22`
3. Source: `0.0.0.0/0`

That means anyone on the internet can try to connect by SSH. This is fine for learning, but for real use it is safer to restrict the source to your own IP address.

## EC2 Instance: `MyEC2Instance`

This resource creates the actual virtual machine.

Why it is used:

1. It gives you a real Linux server to connect to.
2. It demonstrates how CloudFormation can provision compute resources.
3. It can be reused later for web apps, testing, or learning SSH and Linux administration.

### `ImageId`

This uses AWS Systems Manager Parameter Store to fetch the latest Ubuntu 22.04 AMI automatically.

Why this matters:

1. You do not need to hardcode an AMI ID.
2. The template can always use a current Ubuntu image.
3. It reduces manual maintenance when AWS publishes new AMIs.

### `InstanceType`

`t3.micro` is the EC2 size used in the template.

Why it is used:

1. It is small and low-cost.
2. It is enough for basic learning and testing.
3. It is a common free-tier-friendly choice, depending on account eligibility.

### `KeyName`

This points to an existing EC2 key pair named `testing`.

Why it is used:

1. It allows SSH login to the instance.
2. AWS uses the public key part of the key pair to secure access.
3. Without a valid key pair, you cannot easily connect to the instance through SSH.

Important: the key pair must already exist in the same AWS region where you deploy the stack.

### `SecurityGroupIds`

This attaches the security group created in the same stack to the EC2 instance.

Why it is used:

1. The instance needs network rules to allow SSH.
2. Referencing the created security group keeps the stack self-contained.
3. `!Ref EC2SecurityGroup` returns the ID of the security group.

### `Tags`

This adds metadata to the instance.

Why it is used:

1. Tags make resources easier to identify in the AWS Console.
2. The `Name` tag is the most visible label for an EC2 instance.
3. Tags are useful for management, billing, and automation.

The template sets the Name tag to `testing`.

## Outputs Section

Outputs show useful values after the stack is created.

### `InstanceId`

This output returns the EC2 instance ID.

Why it is useful:

1. You can copy the ID without searching through the console.
2. It helps when you want to connect the instance to other AWS services later.
3. It confirms that the instance was created successfully.

### `PublicIP`

This output returns the public IP address of the EC2 instance.

Why it is useful:

1. You need it for SSH connection.
2. It helps you confirm that the instance received network access.
3. It is the fastest way to reach the server after deployment.

## How the `!Ref` and `!GetAtt` Functions Work

The template uses two common CloudFormation functions:

### `!Ref`

`!Ref` returns a resource value. For this template:

1. `!Ref EC2SecurityGroup` returns the security group ID.
2. `!Ref MyEC2Instance` returns the instance ID.

### `!GetAtt`

`!GetAtt` fetches a specific attribute from a resource.

In this template, `!GetAtt MyEC2Instance.PublicIp` returns the public IP address assigned to the instance.

## Step-by-Step Deployment

### 1. Make sure AWS CLI is configured

If you have not configured the CLI yet, run:

```bash
aws configure
```

Enter your AWS access key, secret access key, region, and output format.

### 2. Confirm the key pair exists

The value in `KeyName` is `testing`, so that key pair must exist in the target region.

If it does not exist, create a key pair in the EC2 console first and download the `.pem` file.

### 3. Validate the template

Run:

```bash
aws cloudformation validate-template --template-body file://ec2-creation.yaml
```

This checks that the template syntax is valid.

### 4. Create the stack

Run:

```bash
aws cloudformation deploy ^
	--template-file ec2-creation.yaml ^
	--stack-name ec2-creation-stack ^
	--capabilities CAPABILITY_IAM
```

If you use PowerShell, you can run it on one line or use PowerShell line continuation.

### 5. Check the outputs

After deployment, run:

```bash
aws cloudformation describe-stacks --stack-name ec2-creation-stack
```

Look for the `InstanceId` and `PublicIP` outputs.

## Connecting to the Instance

Use the public IP from the stack output and your `.pem` key file to SSH into the server.

Example:

```bash
ssh -i testing.pem ubuntu@<public-ip>
```

The username for Ubuntu AMIs is usually `ubuntu`.

## Why This Template Is Useful

This template is a clean learning example because it shows:

1. How to create a security group.
2. How to launch an EC2 instance.
3. How to use a parameter store AMI reference.
4. How to expose useful stack outputs.

It is also a good starting point if you want to expand later with user data, EBS volumes, or application software installation.

## Cleanup

When you are done, delete the stack so you stop paying for the instance:

```bash
aws cloudformation delete-stack --stack-name ec2-creation-stack
```

Then wait for deletion to complete:

```bash
aws cloudformation wait stack-delete-complete --stack-name ec2-creation-stack
```

## Suggested Improvements

1. Restrict SSH access to your own IP address instead of `0.0.0.0/0`.
2. Add a parameter for the key pair name instead of hardcoding `testing`.
3. Add a `UserData` script if you want the instance to install software automatically.
4. Add more tags for project, environment, and owner.


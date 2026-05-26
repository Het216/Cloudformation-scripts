# CloudFormation Templates

Examples of CloudFormation templates for EC2-related tasks: EC2 monitoring (Slack alerts via SNS + Lambda) and a simple EC2 instance creation example.

Overview
--------

This repository contains CloudFormation templates and example parameter files for common EC2 workflows: monitoring an existing EC2 instance and notifying Slack, plus a minimal EC2 instance creation template.

Quick links
-----------

- Template: [slack-sns-alerts.yaml](slack-sns-alerts.yaml)
- Example parameters: [parmaters.json](parmaters.json)

Features
--------

- Creates an SNS topic, Lambda function (Node.js) and its IAM role
- Creates a CloudWatch alarm that monitors an EC2 instance's CPU
- Sends formatted alerts to a Slack incoming webhook

Prerequisites
-------------

- AWS account and AWS CLI v2 configured
- An existing EC2 instance ID to monitor
- A Slack incoming webhook URL
- A local AWS CLI profile with permissions to create CloudFormation, IAM, Lambda, SNS, and CloudWatch resources

Basic usage
-----------

Validate template:

```bash
aws cloudformation validate-template --template-body file://slack-sns-alerts.yaml
```

Deploy stack (example):

```bash
aws cloudformation deploy \\
  --template-file slack-sns-alerts.yaml \\
  --stack-name ec2-monitoring-slack-alerts \\
  --parameter-overrides file://parmaters.json \\
  --capabilities CAPABILITY_NAMED_IAM
```

Notes
-----

- The template uses `NoEcho` for the Slack webhook parameter to keep it hidden in the console.
- On Windows use PowerShell continuation or run the command on a single line.

Files
-----

- `slack-sns-alerts.yaml` — main CloudFormation template
- `parmaters.json` — sample parameter file (rename to `parameters.json` if desired)

Troubleshooting & cleanup
-------------------------

If deployment fails, check IAM permissions, the provided EC2 instance ID, and Lambda logs.

To delete the stack and clean up resources:

```bash
aws cloudformation delete-stack --stack-name ec2-monitoring-slack-alerts
aws cloudformation wait stack-delete-complete --stack-name ec2-monitoring-slack-alerts
```

Contributing
------------

Pull requests are welcome. For changes to the template, validate with `aws cloudformation validate-template` before submitting.

License
-------

This repository has no license file; add one if you plan to publish or share the project.


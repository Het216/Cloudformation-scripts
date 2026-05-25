# EC2 Monitoring Slack Alerts

This project deploys a CloudFormation stack that watches an EC2 instance CPU alarm and sends alarm notifications to a Slack channel through an SNS topic and a Lambda function.

The template file is [slack-sns-alerts.yaml](slack-sns-alerts.yaml). The parameter sample file is [parmaters.json](parmaters.json). If you want a cleaner name, you can rename that file to `parameters.json`, but the commands below use the current filename.

## What This Stack Does

The stack creates four main pieces:

1. An SNS topic that CloudWatch alarms publish to.
2. A Lambda function subscribed to that SNS topic.
3. An IAM role for the Lambda function so it can run and write logs.
4. A CloudWatch alarm that monitors EC2 CPU utilization and notifies SNS when the threshold is breached or resolved.

The Lambda function receives the SNS message, converts the CloudWatch alarm payload into a Slack-friendly message, and sends it to your Slack incoming webhook.

## Before You Start

You need:

1. An AWS account.
2. Access to the AWS Management Console.
3. The AWS CLI installed on your machine.
4. A Slack workspace and an incoming webhook URL.
5. An existing EC2 instance ID to monitor.

This template monitors CPU for an existing EC2 instance. It does not create the EC2 instance.

## 1. Install the AWS CLI

If AWS CLI is not installed yet, install version 2 from AWS official documentation.

After installation, confirm it works:

```bash
aws --version
```

You should see a version string that starts with `aws-cli/2`.

## 2. Create IAM Access for the AWS CLI

If you do not already have credentials for CLI use, create an IAM user first. This is the classic setup for local CLI access.

### 2.1 Create an IAM user

In the AWS Console:

1. Open IAM.
2. Go to Users.
3. Choose Create user.
4. Enter a user name such as `cloudformation-cli-user`.
5. Enable access for command line and programmatic access by creating an access key later.

### 2.2 Grant permissions

For learning or personal use, you can attach a managed policy such as:

1. `AdministratorAccess` for full access, or
2. A narrower policy that allows CloudFormation, IAM role creation, Lambda, SNS, CloudWatch, and Logs.

For production, use the least privilege policy that still allows stack creation.

### 2.3 Create an access key

After the IAM user is created:

1. Open the user.
2. Go to Security credentials.
3. Create an access key.
4. Save the Access key ID and Secret access key securely.

Do not commit these credentials to source control.

## 3. Configure the AWS CLI

Run:

```bash
aws configure
```

Enter:

```text
AWS Access Key ID:     your-access-key-id
AWS Secret Access Key: your-secret-access-key
Default region name:   us-east-1
Default output format: json
```

If you use multiple AWS accounts, you can also create a named profile:

```bash
aws configure --profile ec2-alerts
```

Then use `--profile ec2-alerts` in the deployment commands below.

### 3.1 Verify the CLI identity

Check which account the CLI is using:

```bash
aws sts get-caller-identity
```

This should return your account ID, user ID, and ARN.

## 4. Create the Slack Webhook

In Slack:

1. Create or open the target workspace.
2. Create an incoming webhook for the channel you want alerts in.
3. Copy the webhook URL.

The template stores this value in the `SlackWebhookUrl` CloudFormation parameter. The template marks it `NoEcho: true`, which prevents the value from being shown in stack events and console output.

## 5. Explain the Template

The file [slack-sns-alerts.yaml](slack-sns-alerts.yaml) is a CloudFormation template. CloudFormation is infrastructure as code: it lets you define AWS resources in a repeatable file and deploy them with a single command.

### 5.1 Template header

`AWSTemplateFormatVersion` declares the CloudFormation format version. It is mostly informational.

`Description` is the label that appears in the CloudFormation console so you can identify the stack easily.

### 5.2 Parameters section

The template uses two parameters:

`SlackWebhookUrl`
: The Slack webhook endpoint used by the Lambda function. `NoEcho: true` hides it in console displays.

`EC2InstanceId`
: The EC2 instance to monitor. The type `AWS::EC2::Instance::Id` forces a valid EC2 instance ID format.

Why parameters matter:

1. The same template can be reused for different environments.
2. Sensitive values stay outside the template.
3. The instance being monitored can change without editing code.

### 5.3 Resources section

#### SNS topic: `SlackAlertsTopic`

This SNS topic is the message hub.

Why it exists:

1. CloudWatch alarms can publish to SNS directly.
2. SNS can fan out notifications to multiple subscribers later if you add more integrations.
3. It decouples the alarm from the notification target.

#### IAM role: `SlackAlertLambdaRole`

This is the execution role for the Lambda function.

Why it exists:

1. Lambda needs permission to start and run.
2. Lambda needs permission to write logs into CloudWatch Logs.
3. The role is the security boundary for what the function can do.

The template attaches `AWSLambdaBasicExecutionRole`, which is the standard managed policy for Lambda logging.

#### Lambda function: `SlackAlertLambda`

This function receives SNS notifications and posts formatted messages to Slack.

Why it exists:

1. SNS messages from CloudWatch are not Slack-ready by themselves.
2. The function transforms the alarm payload into a nicer alert format.
3. It sends the Slack webhook request securely from AWS.

Important Lambda settings:

`Runtime: nodejs20.x`
: The function uses Node.js 20.

`Handler: index.handler`
: CloudFormation inline code is treated like an `index.js` file exporting a `handler` function.

`Timeout: 15`
: Gives the function enough time to receive the SNS event, format the message, and retry if needed.

`Environment`
: Stores `SLACK_WEBHOOK_URL` as an environment variable so the code can read it at runtime.

#### Log group: `SlackAlertLogGroup`

This controls the Lambda log group and retention.

Why it exists:

1. Logs help diagnose delivery failures and formatting issues.
2. Retention is set to 14 days so logs do not grow forever.
3. It makes log retention explicit instead of leaving it to the default behavior.

#### SNS subscription: `SlackAlertsSubscription`

This connects the SNS topic to the Lambda function.

Why it exists:

1. SNS needs a subscriber to forward messages.
2. The subscription tells SNS to invoke the Lambda function whenever a message is published.
3. It is the bridge between alarm events and the formatter function.

#### Lambda invoke permission: `LambdaInvokePermission`

This resource allows SNS to invoke the Lambda function.

Why it exists:

1. Lambda invocations are denied by default.
2. SNS must be explicitly allowed to call the function.
3. `SourceArn` restricts the permission to only this SNS topic.

#### CloudWatch alarm: `HighCPUAlarm`

This is the alarm that watches EC2 CPU utilization.

Why it exists:

1. It detects sustained CPU pressure on the instance.
2. It sends messages when the metric crosses the threshold.
3. It can notify both when the alarm enters ALARM and when it returns to OK.

Key settings:

`Namespace: AWS/EC2`
: The EC2 service metric namespace.

`MetricName: CPUUtilization`
: The CPU metric to evaluate.

`Statistic: Average`
: Uses average CPU utilization for each period.

`Period: 300`
: Evaluates data in 5-minute windows.

`EvaluationPeriods: 3`
: Needs 3 periods before deciding the alarm state.

`DatapointsToAlarm: 2`
: Requires 2 breaching datapoints out of the 3 periods.

`Threshold: 80`
: CPU above 80% is considered a breach.

`ComparisonOperator: GreaterThanThreshold`
: Fires when CPU goes above the threshold.

`TreatMissingData: ignore`
: Missing metrics are ignored instead of causing noisy state changes.

`AlarmActions` and `OKActions`
: Both point to the SNS topic so Slack gets notified on both alarm and recovery.

### 5.4 Lambda code behavior

The inline Node.js code does the following:

1. Reads the Slack webhook URL from the environment.
2. Extracts the AWS region from the SNS topic ARN.
3. Reads the CloudWatch alarm JSON message from the SNS event.
4. Builds a Slack attachment payload.
5. Skips non-important transitions such as `INSUFFICIENT_DATA -> OK`.
6. Sends the payload to Slack with a POST request.
7. Retries failed sends up to 3 times.

This logic exists because CloudWatch alarm events are raw service messages. The Lambda turns them into something readable in Slack.

## 6. Prepare the Parameter File

The file [parmaters.json](parmaters.json) is used as a CloudFormation parameter file.

It should contain the values for:

1. `SlackWebhookUrl`
2. `EC2InstanceId`

Example structure:

```json
[
	{
		"ParameterKey": "SlackWebhookUrl",
		"ParameterValue": "https://hooks.slack.com/services/..."
	},
	{
		"ParameterKey": "EC2InstanceId",
		"ParameterValue": "i-0123456789abcdef0"
	}
]
```

## 7. Validate the Template

Before deploying, validate the template syntax:

```bash
aws cloudformation validate-template --template-body file://slack-sns-alerts.yaml
```

This checks the template structure, not whether all runtime permissions or resource names will succeed.

## 8. Deploy the Stack

You can deploy with `aws cloudformation deploy`.

### 8.1 Deploy using the parameter file

```bash
aws cloudformation deploy ^
	--template-file slack-sns-alerts.yaml ^
	--stack-name ec2-monitoring-slack-alerts ^
	--parameter-overrides file://parmaters.json ^
	--capabilities CAPABILITY_NAMED_IAM
```

If you use a named profile:

```bash
aws cloudformation deploy ^
	--profile ec2-alerts ^
	--template-file slack-sns-alerts.yaml ^
	--stack-name ec2-monitoring-slack-alerts ^
	--parameter-overrides file://parmaters.json ^
	--capabilities CAPABILITY_NAMED_IAM
```

### 8.2 Why `CAPABILITY_NAMED_IAM` is required

The template creates named IAM resources such as the Lambda execution role. CloudFormation requires explicit confirmation before it creates IAM resources, because those resources can affect account security.

### 8.3 Important note about Windows shells

The caret character `^` is used for line continuation in Windows Command Prompt. If you run these commands in PowerShell, use backticks or write them on one line.

PowerShell example:

```powershell
aws cloudformation deploy `
	--template-file slack-sns-alerts.yaml `
	--stack-name ec2-monitoring-slack-alerts `
	--parameter-overrides file://parmaters.json `
	--capabilities CAPABILITY_NAMED_IAM
```

## 9. Check the Deployment

After deployment, verify the stack:

```bash
aws cloudformation describe-stacks --stack-name ec2-monitoring-slack-alerts
```

You can also list the outputs:

```bash
aws cloudformation describe-stacks --stack-name ec2-monitoring-slack-alerts --query "Stacks[0].Outputs"
```

## 10. Test the Alarm

To test the end-to-end flow, you need the EC2 instance to generate sustained CPU usage above the threshold.

Ways to do that include:

1. Running a temporary CPU load test on the instance.
2. Lowering the threshold temporarily for test purposes.
3. Waiting for a real CPU spike if one is acceptable.

When the alarm enters `ALARM`, CloudWatch publishes to SNS, SNS invokes Lambda, Lambda formats the message, and Slack receives the notification.

When the alarm returns to `OK`, the same path sends a recovery message.

## 11. Troubleshooting

If deployment fails:

1. Check that the EC2 instance ID is valid in the selected region.
2. Confirm the Slack webhook URL is correct.
3. Verify your CLI credentials have permission to create IAM, Lambda, SNS, CloudWatch, and Logs resources.
4. Make sure you included `CAPABILITY_NAMED_IAM`.

If Slack does not receive alerts:

1. Check the Lambda CloudWatch Logs.
2. Confirm the SNS subscription exists.
3. Confirm the CloudWatch alarm actually moved into `ALARM` or back to `OK`.
4. Confirm the webhook URL is still active.

If the alarm exists but never fires:

1. Verify the instance is producing CPU metrics.
2. Confirm the alarm region matches the instance region.
3. Check that the threshold and evaluation periods are appropriate.

## 12. Cleanup

To remove the stack and all resources it created:

```bash
aws cloudformation delete-stack --stack-name ec2-monitoring-slack-alerts
```

Then wait for deletion to finish:

```bash
aws cloudformation wait stack-delete-complete --stack-name ec2-monitoring-slack-alerts
```

## Suggested Improvements

1. Rename [parmaters.json](parmaters.json) to `parameters.json`.
2. Move the Lambda code into a separate file or S3 object instead of inline `ZipFile`.
3. Replace the IAM user workflow with IAM Identity Center if your organization uses it.
4. Add SNS email or additional subscribers if you want backup alerting.


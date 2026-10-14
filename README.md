# Security Monitoring in AWS: CloudWatch Logs, Login Failure Alerts, and NAT Gateway Traffic

A hands-on AWS Builder Lab where I built a full monitoring pipeline: forwarding EC2 security logs to CloudWatch, detecting repeated authentication failures with a metric filter and alarm, and watching for unusual outbound traffic through a NAT gateway, all with email alerts via SNS.

## Scenario
A simulated company ran a WordPress-style web app and database server inside a VPC, with a Network Load Balancer handling external traffic. Because the database server held sensitive data, I needed to monitor its local security logs for repeated login failures and keep an eye on unusual outbound network traffic.

## What I did

### 1. Collected EC2 security logs with the CloudWatch agent
- Connected to the **Database Server** EC2 instance (Amazon Linux 2023) via Systems Manager Session Manager, no SSH needed
- Installed the Amazon CloudWatch agent with `sudo dnf install -y amazon-cloudwatch-agent`
- Ran the configuration wizard and set it up to monitor `/var/log/secure`, sending it to log group `database_server_security_logs` with log stream `{instance_id}` and 1-day retention
- Started the agent and verified its status showed `"status": "running"` and `"configstatus": "configured"`
- Generated test data by running `su dbdev` with an incorrect password four times, then confirmed the failed attempts appeared in the CloudWatch log stream

### 2. Built a metric filter for authentication failures
- Created a metric filter on `database_server_security_logs` matching the pattern `"authentication failure"`
- Tested the pattern against the log stream to confirm it caught the failures I'd just generated
- Configured it as **database server authentication failures**, metric namespace `authentication failures`, value `1` per match, default `0`

### 3. Created a CloudWatch alarm with SNS notifications
- Alarm on the custom metric: Sum statistic, 5-minute evaluation period, threshold **greater than 2**
- Missing-data treatment set to "Treat missing data as good (not breaching threshold)"
- Created an SNS topic, **CloudWatch_alarm_notifications**, with an email subscription (confirmed via the AWS confirmation email)
- Named the alarm **database server authentication failures alarm**

### 4. Monitored NAT gateway outbound traffic
- Created a second alarm on the NAT gateway's `BytesOutToDestination` metric: Sum statistic, 15-minute period, threshold **greater than 2,000,000 bytes**
- Notification routed to the same SNS topic, named **NAT gateway outgoing traffic alarm**
- Generated test traffic by uploading a ~2.3 MB file to the lab's S3 test bucket from the database server's Session Manager terminal: `aws s3 cp ~/SampleData.zip s3://$TestBucket/`
- Confirmed the alarm entered ALARM state and the SNS email fired

## Key takeaways
- Collecting logs and detecting threats are two different steps — a metric filter is what turns a raw log pattern into a measurable number an alarm can actually evaluate
- The gap between "logs exist" and "someone gets alerted" is filled by exactly three pieces: a metric filter, an alarm threshold, and an SNS subscription someone has actually confirmed
- "Treat missing data as good" avoids false alarms from gaps in telemetry, but it can also quietly mask an agent that's stopped reporting — worth a separate health check for critical pipelines
- High outbound traffic isn't inherently malicious; thresholds need to be tuned against what's actually normal for the environment, not a lab default
- An alert is the start of an investigation, not the conclusion of one — someone still has to receive it, investigate, and decide what it means

## Tools
Amazon CloudWatch (Logs, metric filters, alarms), Amazon SNS, Amazon EC2, AWS Systems Manager Session Manager, Amazon VPC / NAT Gateway, Amazon S3, Amazon Linux 2023

---
*Completed as an AWS Builder Lab (Security Monitoring).*

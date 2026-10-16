# Build a Security Monitoring System

**Author:** Hayder Ben Abdelhamid  
**Email:** haider.abdelhamid@gmail.com  
**Role:** Cybersecurity Engineering Student  

---

## Introducing Today's Project!

In this project, I demonstrated how to set up a real-time monitoring and alerting system in AWS using CloudTrail, CloudWatch, and Amazon SNS. The goal was to understand how cloud security and monitoring services integrate while building a functional pipeline that delivers instant email alerts upon sensitive data access.

### Tools and Concepts
* **AWS Services:** Secrets Manager, CloudTrail, CloudWatch Logs, CloudWatch Alarms, Amazon SNS, IAM, S3.
* **Key Concepts:** Secret vaulting, audit logging, log metric filters, alarm thresholds, event-driven notification architecture.

### Project Reflection
This project took approximately 3 hours. The most challenging part was troubleshooting why email notifications weren't delivering during our initial test—debugging asynchronous event pipelines without immediate error logs can be tricky. The most rewarding part was comparing CloudTrail vs. CloudWatch notifications, which provided deep insight into why custom metric filters and alarms are necessary for targeted alerting.

---

## Create a Secret

AWS Secrets Manager is used to store sensitive data (like database credentials, API keys, and configuration tokens) securely so they aren't hardcoded into source code.

For this project, I created a secret named `TopSecretInfo` containing a string value.

```json
{
  "coffee_requirement": "3 coffees a day to function"
}
```

---

## Set Up CloudTrail

CloudTrail is a governance and auditing service that tracks API activity and events across an AWS account. 

I configured my trail to capture **Management events**, because retrieving a secret falls under management operations. 

```bash
# Verify management event logging configuration for the trail
aws cloudtrail describe-trails --trail-name-list MySecurityTrail
```

### Read vs. Write Activity
* **Read API Activity:** Accessing or viewing a resource.
* **Write API Activity:** Creating, updating, or deleting a resource. 

For this project, I tracked management activity where retrieving a secret (`GetSecretValue`) is logged under account activity.

---

## Verifying CloudTrail

I retrieved the secret in two ways:
1. Via the **AWS Secrets Manager Console** by clicking *Retrieve secret value*.
2. Via the **AWS CLI** using CloudShell.

```bash
# AWS CLI command used to retrieve the secret value and generate a CloudTrail event
aws secretsmanager get-secret-value --secret-id TopSecretInfo
```

Reviewing the **CloudTrail Event History** confirmed that a `GetSecretValue` event was successfully logged regardless of whether the retrieval was initiated via the console or the CLI.

---

## CloudWatch Metrics

CloudWatch centralizes log data and enables advanced analysis, metric creation, and alarms. While CloudTrail Event History stores events for 90 days, CloudWatch Logs allow long-term retention and custom filtering.

I created a **CloudWatch Metric Filter** to parse the log group and increment a custom metric by `1` every time the `GetSecretValue` event occurred.

```json
{
  "$.eventName": "GetSecretValue",
  "$.eventSource": "secretsmanager.amazonaws.com"
}
```

---

## CloudWatch Alarm & SNS Topic

A CloudWatch alarm monitors metrics and triggers state changes when specific thresholds are breached.

* **Metric Threshold:** Configured to trigger when the **SUM** of `GetSecretValue` events exceeds `1` over a 5-minute evaluation period.
* **Amazon SNS Topic:** Created a broadcast channel (`SecretAccessAlerts-Topic`) connected to an email endpoint. 

```bash
# Create an SNS topic for security alerts
aws sns create-topic --name SecretAccessAlerts-Topic
```

AWS requires manual email subscription confirmation to prevent unsolicited notifications.

---

## Troubleshooting Notification Errors

During my first test run, accessing the secret did not trigger an email notification. 

**Investigation Steps:**
1. Verified that CloudTrail was successfully capturing the `GetSecretValue` event.
2. Confirmed that CloudTrail logs were streaming into CloudWatch.
3. Inspected the metric filter to ensure it wasn't dropping valid JSON paths.
4. Checked the CloudWatch Alarm evaluation logic.

**Root Cause & Fix:**
The alarm was incorrectly configured to calculate the *average* number of secret accesses instead of the **SUM** over the evaluation period. Once updated to use the `SUM` statistic, the alarm correctly tracked event frequency.

---

## Success!

To validate the completed pipeline, I retrieved the secret one final time. 

* **Result:** An email notification was delivered to my inbox within 1–2 minutes.
* **CloudWatch Status:** The alarm transitioned into the `ALARM` state.

```json
{
  "AlarmName": "SecretAccessAlarm",
  "StateValue": "ALARM",
  "Reason": "Threshold Crossed: 1 datapoint [1.0] was greater than or equal to the threshold (1)."
}
```

---

## Comparing CloudWatch with CloudTrail Notifications

In a project extension, I enabled direct SNS notification delivery inside CloudTrail to evaluate CloudTrail notifications vs. CloudWatch Alarms.

* **Observation:** My inbox was quickly flooded with raw notifications every time log files were delivered to the S3 bucket.
* **Conclusion:** CloudTrail SNS notifications alert you that new log files exist, whereas CloudWatch Metric Filters and Alarms allow for precise, targeted alerting on specific security events (like unauthorized secret access) without unnecessary noise.

---

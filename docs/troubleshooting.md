# 🔎 Troubleshooting & Root Cause Analysis

## AWS Security Monitoring System

## Problem Statement

During the initial end-to-end test, retrieving the test secret `TopSecretInfo` did not produce the expected email notification. The pipeline was intended to record `GetSecretValue` in CloudTrail, forward events to CloudWatch Logs, publish a custom metric, evaluate a CloudWatch alarm, and notify an email subscriber through SNS.

**Expected:** Secret retrieval → CloudTrail event → CloudWatch Logs → metric filter → alarm → SNS email.

**Observed:** The secret was retrieved and monitoring components were configured, but the expected email was not received during the initial test.

## Diagnostic Approach

The investigation tested each stage independently rather than changing multiple settings at once.

| Stage | Diagnostic action | Finding |
|---|---|---|
| CloudTrail | Filter Event History for `secretsmanager.amazonaws.com` and locate `GetSecretValue` | Retrieval events were recorded |
| CloudWatch Logs | Inspect log group `nextwork-secretsmanager-loggroup` and its streams | CloudTrail records were present |
| Metric filter | Test `"GetSecretValue"` against sample event data | Matching events were identified |
| CloudWatch alarm | Review statistic, period, threshold, and alarm actions | Original evaluation settings were reconsidered |
| SNS subscription | Confirm email subscription to `SecurityAlarms` | Subscription confirmation verified |
| SNS delivery | Publish a test message directly to SNS | Email delivery succeeded |
| Final end-to-end test | Retrieve the secret again after alarm adjustment | Alarm and email notification succeeded |

## 1. CloudTrail Verification

1. Open **CloudTrail → Event history**.
2. Filter by event source `secretsmanager.amazonaws.com`.
3. Locate `GetSecretValue` events associated with the test retrieval.
4. Inspect the event timestamp, user identity, region, and requested resource.

**Finding:** CloudTrail captured the expected API activity. `GetSecretValue` is a read management event.

## 2. CloudWatch Logs Verification

1. Open the trail `secrets-manager-trail` and review its CloudWatch Logs integration.
2. Confirm the configured destination log group is `nextwork-secretsmanager-loggroup`.
3. Check the CloudTrail log delivery IAM role.
4. Inspect CloudWatch log streams for the expected events.

**Finding:** Events were present in CloudWatch Logs. S3 delivery and CloudWatch Logs delivery must be verified independently.

## 3. Metric Filter Verification

**Filter pattern:**

```text
"GetSecretValue"
```

**Metric configuration:**

```text
Namespace: SecurityMetrics
Metric: Secret is accessed
Value on match: 1
Default value: 0
```

The filter was tested against sample events containing `GetSecretValue`, `DescribeSecret`, `GetResourcePolicy`, `ListSecretVersionIds`, and `CreateSecret`.

**Finding:** The test pattern identified the `GetSecretValue` sample event and excluded the unrelated operations.

**Limitation:** The pattern is a text match and is not restricted to one secret or even explicitly to Secrets Manager's event source.

## 4. Alarm Evaluation

### Initial Settings

| Setting | Value |
|---|---|
| Alarm | `Secret is accessed` |
| Statistic | Average |
| Period | 5 minutes |
| Threshold | >= 1 |
| Notification topic | `SecurityAlarms` |

To test the alarm action independently from the metric, a manual alarm state change can be used in an authorized lab:

```bash
aws cloudwatch set-alarm-state \
  --alarm-name "Secret is accessed" \
  --state-value ALARM \
  --state-reason "Manually triggered for testing"
```

A manual state change tests alarm actions; it does **not** prove the metric filter or metric-based threshold is functioning.

### Configuration Change

| Setting | Before | After |
|---|---|---|
| Statistic | Average | Sum |
| Period | 5 minutes | 1 minute |
| Threshold | >= 1 | >= 1 |

**Rationale:** `Sum` is a clearer fit for counting occurrences because each matching event contributes `1` to the metric. The shorter period changes the evaluation window.

**Root-cause qualification:** The original end-to-end test failed and the final test succeeded after these changes, but the available evidence does not establish that `Average` alone caused the failure. An average of published values equal to `1` can also satisfy a threshold of `>= 1`; missing data, timing, and alarm state must also be considered.

## 5. SNS Subscription and Delivery

1. Open **Amazon SNS → Topics → SecurityAlarms**.
2. Verify the email subscription is **Confirmed**, not pending confirmation.
3. Publish a benign test message directly to the topic.
4. Confirm that the message reaches the subscribed inbox.

**Finding:** The SNS delivery test succeeded, helping isolate the issue from basic email delivery.

## 6. Final End-to-End Test

1. Retrieve the test secret from AWS Secrets Manager.
2. Verify a new `GetSecretValue` event in CloudTrail.
3. Confirm CloudWatch Logs receives the event.
4. Review the metric and alarm state.
5. Verify receipt of the SNS email notification.

**Result: PASS.** The alarm entered `ALARM` and an email was delivered following the configuration adjustment.

## Results Summary

| Test | Result |
|---|---|
| CloudTrail event logging | PASS |
| CloudWatch Logs ingestion | PASS |
| Metric filter sample matching | PASS |
| Initial end-to-end email notification | FAIL |
| SNS direct delivery | PASS |
| Final end-to-end email notification | PASS |

## Lessons Learned

- Test logging, detection, alarm evaluation, and notification delivery as separate components.
- Select metric statistics based on the signal being measured.
- Check alarm history and missing-data settings when expected transitions do not occur.
- Verify SNS email subscription confirmation.
- Expect asynchronous delivery and processing delays.
- CloudWatch alarms generally notify on state transitions, not on each event while already in `ALARM`.

## Suggested Improvements

- Replace the text filter with a structured event filter.
- Monitor a specific secret and expected access identities.
- Add dashboards and alert enrichment.
- Record exact timestamps and alarm history during future tests to support stronger root-cause conclusions.

## References

- [CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [CloudWatch metric filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
- [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)

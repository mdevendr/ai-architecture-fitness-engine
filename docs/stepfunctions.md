# Step Functions — Architecture Signal Aggregator & Scoring Orchestrator

## 1. Purpose of Step Functions
Step Functions acts as the control plane for the Architecture Fitness Engine. It does not analyze or score architecture; instead, it prepares, validates, groups, governs, and orchestrates signals before invoking the Lambda AI Orchestrator and Amazon Bedrock.

Responsibilities:
- Receive batched signals from SQS
- Parse, normalize, and validate signals
- Group signals by resource
- Optionally group by workload
- Load governance/threshold configuration from S3
- Enforce cooldowns and idempotency
- Determine if scoring is required
- Build clean evaluation context
- Invoke Lambda AI Orchestrator
- Apply governance to AI output
- Write authoritative records to S3 (Score & Findings)

## 2. Signal Lifecycle

### 2.1 In SQS (Persistent)
- Messages remain until Step Functions deletes them
- Step Functions only makes them invisible temporarily (visibility timeout)
- SQS provides durability and ordering

### 2.2 In Step Functions (Ephemeral)
- Signals exist only in JSON input/output during execution
- Step Functions stores nothing persistently

### 2.3 Hand-off to Lambda AI Orchestrator
- Step Functions sends only clean evaluation context
- Raw noisy signals are never forwarded

## 3. Input Schema (From SQS)
Each SQS message follows this canonical structure:

signal_type: ""
resource_id: ""
resource_type: ""
aws_service: ""
event_time: ""
severity: ""
signal_payload: {}
source_event: {}

## 4. How Step Functions Aggregates Signals

### 4.1 Parse and Validate
- Ensure required fields exist
- Drop malformed signals

### 4.2 Group by Resource
Example structure:
resource_id: "lambda:payment-processor"
signals:
  - signal_type: "THROTTLE"
  - signal_type: "LATENCY_SPIKE"

### 4.3 Combine Into Workload View (Optional)
If naming conventions or mapping tables exist, Step Functions groups resources into workloads.

### 4.4 Apply Threshold Governance
Step Functions loads governance/threshold config from S3:
s3://governance_xxxxxx/threshold-config.json

It then enforces:
- Drift severity thresholds
- Cost anomaly impact thresholds
- Latency/error thresholds
- Throttle thresholds
- Minimum signal count
- Scoring cooldown window

### 4.5 Build Evaluation Payload
Example:
workload_id: "payments"
resources:
  - resource_id: "lambda:payment-processor"
    resource_type: "lambda"
    signals: [...]
severity: "high"
trend: {...}
evaluation_window: "<timestamp>"

This is the payload sent to Lambda.

## 5. When Step Functions Triggers Scoring
Scoring occurs only when thresholds or governance rules are met. Trigger conditions include:
- High-severity anomalies
- Config drift confirmed
- Cost spikes exceeding threshold
- Retry storms
- Performance degradation
- Minimum signal count met
- Cooldown window expired
- Trend analysis indicates deterioration
- Meaningful change since last score

## 6. Idempotency
Step Functions prevents duplicate scoring using:
- Stable scoring window keys
- Last evaluated timestamp
- Resource signal hash comparison
- SQS visibility timeout logic
- Drop-if-identical rule

## 7. SQS Deletion and DLQ
Step Functions does not move messages to DLQ. SQS handles DLQ routing automatically when:
- Message fails > maxReceiveCount
- Message expires after retention

## 8. Why Step Functions Is Required
Provides:
- Deterministic orchestration
- Structured aggregation
- Threshold enforcement
- Trend evaluation
- Multi-resource correlation
- Safe prompt preparation
- Retry logic
- Consistent storage format
- Auditability

## 9. Flow Summary
SQS -> Step Functions (parse/aggregate/validate) -> Lambda AI Orchestrator -> Bedrock -> Step Functions (governance) -> S3 (Score & Findings) -> SNS/Jira/Teams.

## 10. What Step Functions Does NOT Do
- Infer architecture issues
- Interpret anomalies
- Detect anti-patterns
- Generate recommendations
These belong exclusively to Bedrock via the Lambda AI Orchestrator.

## Final Summary
Step Functions is the aggregator, governor, and dispatcher for the Architecture Fitness Engine. It ensures structured, consistent, safe evaluations before invoking AI reasoning. Without Step Functions, insights would be noisy, duplicated, unactionable, and non-deterministic.

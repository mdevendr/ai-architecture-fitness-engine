# EventBridge Pipes (Quick Summary)

- Serverless connector that moves events from a source to a target without needing Lambda.
- Provides filtering, transformation, and enrichment of incoming events.
- Sources:  
  - [EventBridge Rules, CloudWatch, SQS, DynamoDB Streams, Kinesis]
- Targets:  
  - [SQS FIFO, Step Functions, EventBridge Bus, Lambda, API Destinations]
- In this architecture, the Pipe:
  - Uses a Filter to drop noise  
  - Uses an Input Transformer to convert events into a canonical architecture signal format  
  - Sends the normalized event to SQS FIFO for ordered, deduped processing before Step Functions consumes it.


---

# What EventBridge Pipes Receive (Input)

Each Pipe receives **raw filtered events** from EventBridge Rules.

Examples of incoming event formats:
- AWS Config compliance changes  
- Cost Explorer anomaly alerts  
- CloudWatch metric alarms  
- DynamoDB throttle spikes  
- Compute Optimizer recommendations  
- Trusted Advisor high-risk items  

These events all differ in structure.  
Pipes transform them into a common schema.

---

# What EventBridge Pipes Output (Normalized Architecture Signal)

Every Pipe transforms events into a **canonical signal envelope**:

```json
{
  "signal_type": "",
  "resource_id": "",
  "resource_type": "",
  "aws_service": "",
  "event_time": "",
  "severity": "",
  "signal_payload": {},
  "source_event": {}
}
```

This is what Step Functions, DynamoDB, QuickSight, and Bedrock will consume.

---

# Transformation Logic

EventBridge Pipes apply **Input Transformers** to:
1. Extract relevant fields
2. Normalize all event formats
3. Add derived fields (severity, signal_type)
4. Compress noisy fields
5. Preserve the full original event for auditability

Each rule type has its own mapping logic.

---

# Canonical Transformations Per Signal Category

Below are the transformations used for each signal type.  
Every example shows **input → output**.

---

## 1. AWS Config Drift

### Input Example:
```json
{
  "source": "aws.config",
  "detail": {
    "configRuleName": "multi-az-enabled",
    "newEvaluationResult": { "complianceType": "NON_COMPLIANT" }
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "CONFIG_DRIFT",
  "resource_id": "<config resourceId>",
  "resource_type": "<config resourceType>",
  "aws_service": "Config",
  "event_time": "<resultRecordedTime>",
  "severity": "HIGH",
  "signal_payload": {
    "config_rule": "<configRuleName>",
    "compliance_status": "<NON_COMPLIANT>"
  },
  "source_event": <original-event>
}
```

---

## 2. Cost Anomaly Signals

### Input Example:
```json
{
  "source": "aws.ce",
  "detail": {
    "anomalyScore": 92,
    "service": "AmazonEC2",
    "impact": { "totalImpact": 134.22 }
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "COST_ANOMALY",
  "resource_id": "ACCOUNT",
  "resource_type": "AWS::CostExplorer",
  "aws_service": "CostExplorer",
  "event_time": "<time>",
  "severity": "HIGH",
  "signal_payload": {
    "service": "<detail.service>",
    "impact": "<totalImpact>",
    "anomaly_score": "<anomalyScore>"
  },
  "source_event": <original-event>
}
```

---

## 3. CloudWatch Performance Degradation

### Input Example:
```json
{
  "source": "aws.cloudwatch",
  "detail": {
    "metricName": "5XXError",
    "value": 321,
    "threshold": 100
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "PERFORMANCE_DEGRADATION",
  "resource_id": "<dimension-identifier>",
  "resource_type": "AWS::ApplicationMetric",
  "aws_service": "CloudWatch",
  "event_time": "<time>",
  "severity": "MEDIUM",
  "signal_payload": {
    "metric_name": "5XXError",
    "metric_value": 321,
    "threshold": 100
  },
  "source_event": <original-event>
}
```

---

## 4. DynamoDB Throttles / Hot Partitions

### Input Example:
```json
{
  "source": "aws.cloudwatch",
  "detail": {
    "metricName": "ConsumedWriteCapacityUnits",
    "value": 10240
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "DDB_THROTTLE",
  "resource_id": "<table-name>",
  "resource_type": "AWS::DynamoDB::Table",
  "aws_service": "DynamoDB",
  "event_time": "<time>",
  "severity": "HIGH",
  "signal_payload": {
    "metric_name": "ConsumedWriteCapacityUnits",
    "value": 10240
  },
  "source_event": <original-event>
}
```

---

## 5. Compute Optimizer Recommendations

### Input Example:
```json
{
  "source": "aws.compute-optimizer",
  "detail": {
    "resourceArn": "arn:aws:ec2:...",
    "finding": "Overprovisioned",
    "recommendation": "m5.large"
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "RIGHTSIZING_RECOMMENDATION",
  "resource_id": "<resourceArn>",
  "resource_type": "<derived-type>",
  "aws_service": "ComputeOptimizer",
  "event_time": "<time>",
  "severity": "LOW",
  "signal_payload": {
    "finding": "Overprovisioned",
    "recommendation": "m5.large"
  },
  "source_event": <original-event>
}
```

---

## 6. Trusted Advisor High-Risk Findings

### Input Example:
```json
{
  "source": "aws.trustedadvisor",
  "detail": {
    "checkId": "xyz123",
    "status": "WARN"
  }
}
```

### Pipe Output:
```json
{
  "signal_type": "TA_RISK",
  "resource_id": "<resource>",
  "resource_type": "<AWS-resource-type>",
  "aws_service": "TrustedAdvisor",
  "event_time": "<time>",
  "severity": "HIGH",
  "signal_payload": {
    "check_id": "xyz123",
    "status": "WARN"
  },
  "source_event": <original-event>
}
```

---

# Why Transformation Is Required

EventBridge Pipes standardize all signals into one format so downstream components can:
- Score workloads consistently  
- Store results in DynamoDB predictably  
- Pass structured inputs to Bedrock  
- Build QuickSight dashboards  
- Auto-create Jira tasks  

Without normalization, each service’s unique event format would break the scoring pipeline.

---

# Final Output Schema

All Pipes must output the following exact JSON schema:

```json
{
  "signal_type": "",
  "resource_id": "",
  "resource_type": "",
  "aws_service": "",
  "event_time": "",
  "severity": "",
  "signal_payload": {},
  "source_event": {}
}
```

This ensures reliability across the entire architecture fitness engine.


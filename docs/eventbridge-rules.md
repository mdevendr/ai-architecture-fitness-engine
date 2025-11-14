# EventBridge Rules — Cloud Signal Filter

The Cloud Signal Filter captures ONLY architecturally meaningful signals.
Everything else is ignored.

These six categories represent over 90% of architecture drift and modernization triggers:

1. Config Drift (AWS Config)
2. Cost Anomalies (Cost Explorer)
3. Latency / Errors / Throttles (CloudWatch)
4. DynamoDB Hot Partitions / Storage Hotspots
5. Compute Optimizer Recommendations
6. Trusted Advisor High-Risk Findings

---

# 1. Config Drift Signals
Events that indicate architecture compliance drift.

## Rule: Config Compliance Change

```json
{
  "source": ["aws.config"],
  "detail-type": ["Config Rules Compliance Change"]
}
```

Examples filtered in:
- Multi-AZ removed
- Encryption disabled
- Public access enabled
- DR controls drift
- IAM policy drift

---

# 2. Cost Anomaly Signals

## Rule: Cost Anomaly Detected

```json
{
  "source": ["aws.ce"],
  "detail-type": ["Cost Anomaly Detection Alert"]
}
```

Examples filtered in:
- Sudden cost spikes
- Unexpected scaling
- Over-provisioned compute or storage
- Idle but expensive resources

---

# 3. CloudWatch Performance Degradation Signals

## Rule: High Error Rate

```json
{
  "source": ["aws.cloudwatch"],
  "detail": { "metricName": ["5XXError"] }
}
```

## Rule: High Latency Spike

```json
{
  "source": ["aws.cloudwatch"],
  "detail": { "metricName": ["Latency"] }
}
```

## Rule: Throttling Detected

```json
{
  "source": ["aws.cloudwatch"],
  "detail.metricName": ["Throttles"]
}
```

Examples filtered in:
- API failures
- Slow downstream dependencies
- Bottlenecks
- Saturation and throttling

---

# 4. DynamoDB / Storage Hotspots

## Rule: DynamoDB Capacity Stress

```json
{
  "source": ["aws.cloudwatch"],
  "detail.metricName": [
    "ConsumedReadCapacityUnits",
    "ConsumedWriteCapacityUnits"
  ]
}
```

Examples filtered in:
- Hot partitions
- Hash key imbalance
- Sudden IO spikes

---

# 5. Compute Optimizer Recommendations

## Rule: Recommendation Generated

```json
{
  "source": ["aws.compute-optimizer"],
  "detail-type": ["RecommendationGenerated"]
}
```

Examples filtered in:
- Over-provisioned EC2 instances
- Under-provisioned Lambda memory
- EBS throughput mismatch
- ECS compute imbalance

---

# 6. Trusted Advisor High-Risk Findings

## Rule: High-Risk TA Check Updated

```json
{
  "source": ["aws.trustedadvisor"],
  "detail-type": ["Trusted Advisor Check Item Refresh Notification"]
}
```

Examples filtered in:
- Fault tolerance issues
- Security exposures
- Performance degradation
- Cost inefficiency risks

---

# Summary

These rules ensure the engine reacts only to meaningful architecture signals:

- Architecture drift
- Resilience issues
- Performance bottlenecks
- Cost inefficiencies
- Scalability failures
- Modernization triggers

Operational noise is excluded by design.

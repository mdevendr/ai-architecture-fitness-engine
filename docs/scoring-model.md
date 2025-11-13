# Scoring Model

This document defines how the AI-driven engine computes architecture fitness and modernization scores across multiple dimensions.

## 1. Cost Efficiency (0–100)
Evaluates:
- Right-sizing alignment (CPU, memory)
- Idle resource percentage
- Storage lifecycle optimisation
- Data transfer patterns
- Reserved vs On-Demand efficiency

Signals:
- CloudWatch utilization metrics
- Cost Explorer granular usage
- EC2/ASG right-sizing metrics

## 2. Reliability & Resilience (0–100)
Factors:
- Multi-AZ / Multi-Region readiness
- DR posture (RPO/RTO signals)
- Error rate trends
- Retry/backoff behaviour
- Throttling patterns

Signals:
- CloudWatch alarms
- Service quotas
- Config resource relationships

## 3. Scalability (0–100)
Evaluates:
- Autoscaling policies
- Concurrency behaviour
- Queue depth & lag
- Burst-handling capability

Signals:
- Lambda concurrency metrics
- ALB surge queue
- Kinesis shard utilization

## 4. Maintainability (0–100)
Evaluates:
- Architecture complexity
- Modularity
- Operational overhead
- Pattern alignment (EDA, serverless, event-driven)

Signals:
- Deployment frequency
- Change failure rate
- Config drift

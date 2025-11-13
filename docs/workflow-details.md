# Workflow Details

This document explains each workflow stage used in the Architecture Fitness Engine.

## 1. Cloud Signals Layer
Uses:
- CloudWatch Metrics
- AWS Config
- DynamoDB Streams
- SQS metrics
- Lambda concurrency signals

Generates:
- High-value architecture events

## 2. EventBridge Pipes (Filter + Transform)
- Filters high-volume signals
- Removes noise
- Transforms event payloads for processing

## 3. Signal Buffer (SQS)
Used for:
- Durability
- Burst protection
- Decoupling fast producers from slow consumers

## 4. EventBridge Scheduler
Triggers daily or hourly architecture-wide evaluation.

## 5. Step Functions (Evaluator & Orchestrator)
Coordinates:
- Signal intake
- Dynamic thresholds
- AI invocation
- Score storage

## 6. Lambda (AI Orchestrator)
Prepares:
- Resource metadata
- Metrics
- Architecture summary

Calls:
- Amazon Bedrock for scoring & recommendations

## 7. DynamoDB
Stores:
- Evaluation history
- Score snapshots
- Metadata for each workload

## 8. S3
Stores:
- Reports
- Diagrams
- Audit exports

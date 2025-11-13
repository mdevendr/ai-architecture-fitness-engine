# Modernization Pattern Recommendations

The AI engine recommends patterns based on operational, performance, and architectural signals.

## 1. Serverless Migration
Recommended when:
- EC2 idle > 60%
- Spiky workloads
- Intermittent traffic

## 2. SQS Buffer Pattern
Triggered when:
- Throttling detected
- API burst failures
- Async boundary required

## 3. CQRS / Event Sourcing
Triggered when:
- Mixed read/write load
- High contention
- Auditability required

## 4. Strangler Fig Migration
Recommended when:
- Legacy monolith with high coupling
- Microservice extraction possible

## 5. DAX for DynamoDB
Triggered when:
- High read amplification
- Repeated hot-key read traffic
- Cacheable responses

## 6. Mesh / Sidecar
Triggered when:
- High service-to-service communication
- Need for mTLS, retries, or observability

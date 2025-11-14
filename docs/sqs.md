# SQS FIFO (Durability, DLQ Behaviour, Architectural Role, Costing)

## Durability
- SQS FIFO stores messages redundantly across multiple AZs.
- Guarantees at-least-once delivery with no message loss unless message TTL expires.
- Supports retention up to 14 days.
- Provides ordered and deduped buffering for all architecture signals.
- Ensures cost anomalies, drift, throttles, and performance signals are not lost during burst events.

## Why SQS FIFO Is Used in This Architecture
- Decouples event ingestion (EventBridge Pipes) from scoring (Step Functions) so each scales independently.
- Smooths bursty architecture signals into a predictable flow for Step Functions.
- Maintains strict ordering for workload-specific signals, ensuring consistent scoring.
- Provides durable multi-AZ buffering so downstream failures cannot cause data loss.
- Isolates poison messages via DLQ without blocking the main queue.
- Allows future consumers (analytics, dashboards, audit jobs) to reuse the same signal stream.
- Avoids thousands of Step Functions executions during spikes, drastically reducing cost.

## DLQ Behaviour (When Messages Go to DLQ)
A message enters the DLQ when:
1. Step Functions fails to process the message after `maxReceiveCount` retries.
2. The signal is malformed or structurally invalid.
3. The Bedrock scoring workflow repeatedly returns errors.
4. The message becomes a poison event (fails on every attempt).
5. The message exceeds the SQS retention period (e.g., 7 days).

## What Is Stored in the DLQ
The full normalized architecture signal:
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

## Cost Considerations
- SQS FIFO costs are based mainly on API requests (send, receive, delete).
- No compute cost, making it extremely cheap even under heavy event floods.
- Batching significantly reduces receive operations.
- DLQ adds cost only per message written.
- Eliminates expensive Lambda-based filtering and reduces Step Functions invocations.
- Typical estate example: 200 workloads, ~4,000 signals/day → SQS cost usually under $1/month.


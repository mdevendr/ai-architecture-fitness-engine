# Lambda AI Orchestrator

## 1. Overview

The **Lambda AI Orchestrator** is the thin translation layer between **Step Functions** and **Amazon Bedrock** in the Architecture Fitness & Modernization Engine.

It does **not** perform any business logic, scoring, or governance itself. Its responsibilities are:

- Accept a **clean, aggregated evaluation context** from Step Functions.
- Construct a **Bedrock-ready prompt** from that context.
- Call the configured **Bedrock model** using `InvokeModel` (or `InvokeModelWithResponseStream`).
- Validate and normalize the model response into a strict JSON schema.
- Return the normalized result back to Step Functions.

All governance, threshold decisions, and orchestration stay in **Step Functions**.  
All architectural reasoning and recommendations come from **Bedrock**.

---

## 2. Responsibilities

### 2.1 What Lambda Does

- Receives evaluation context from Step Functions.
- Optionally enriches input with static configuration (e.g., from S3 prompt templates).
- Builds a **system prompt** and **user prompt** for Bedrock.
- Calls the Bedrock model with `InvokeModel`.
- Parses and validates the LLM response.
- Normalizes the response into a predictable JSON structure.
- Returns the structured result to Step Functions.

### 2.2 What Lambda Does NOT Do

- Does **not** read from SQS directly.
- Does **not** write results to S3.
- Does **not** send notifications (SNS/Teams/Jira).
- Does **not** decide when to score.
- Does **not** implement governance or thresholds.
- Does **not** perform long-running workflows.

Those responsibilities are owned by:

- Step Functions (governance/orchestration).
- S3 (Score & Findings).
- SNS / external integrations (notification).
- Bedrock (reasoning and recommendations).

---

## 3. Input Contract (From Step Functions)

Lambda receives a **single JSON document** from Step Functions.  
The exact shape can be tuned, but it should follow this structure:

```json
{
  "evaluation_window": "2025-02-13T10:00:00Z",
  "scoring_mode": "reactive",
  "workload_id": "payments",
  "severity_context": "HIGH",
  "thresholds": {
    "latency_high_ms": 2000,
    "error_rate_threshold": 0.05,
    "throttle_threshold": 10,
    "cost_spike_percent": 30
  },
  "resources": [
    {
      "resource_id": "lambda:payment-processor",
      "resource_type": "AWS::Lambda::Function",
      "aggregated_signals": [
        {
          "signal_type": "PERFORMANCE_DEGRADATION",
          "severity": "HIGH",
          "signal_payload": {
            "latency_p95_ms": 4200,
            "baseline_latency_p95_ms": 800
          }
        },
        {
          "signal_type": "THROTTLE",
          "severity": "MEDIUM",
          "signal_payload": {
            "throttle_count": 37
          }
        }
      ]
    }
  ],
  "trend": {
    "latency_direction": "increasing",
    "error_rate_direction": "flat"
  }
}
```

Key properties:

- **Step Functions** is responsible for:
  - Aggregation and grouping.
  - Loading thresholds from S3.
  - Deciding that scoring is required.
- **Lambda** treats the payload as **read-only context**.

---

## 4. Prompt Construction

The Lambda function converts the input JSON into a **Bedrock prompt**. A typical pattern:

### 4.1 System Prompt (Stable, Predefined)

Example:

> You are an AWS Architecture Fitness and Modernization Reasoning Engine.  
> You evaluate cloud workloads based on architecture signals (drift, cost, performance, throttles, and scalability indicators).  
>  
> Follow these rules strictly:  
> - Use ONLY the data provided in the input JSON.  
> - Respect all thresholds provided.  
> - Do NOT invent new resources or signals.  
> - Respond in strict JSON format.  
> - Provide modernization suggestions using AWS-native patterns.  

This system prompt can be stored inline in code or loaded from S3 as a static template.

### 4.2 User Prompt (Dynamic, Derived from Context)

Lambda interpolates the evaluation context into a user message, e.g.:

> Evaluate the following architecture context for fitness and modernization potential.  
>  
> Evaluation window: 2025-02-13T10:00:00Z  
> Scoring mode: reactive  
> Workload ID: payments  
> Severity context: HIGH  
>  
> Thresholds:  
> - latency_high_ms: 2000  
> - error_rate_threshold: 0.05  
> - throttle_threshold: 10  
> - cost_spike_percent: 30  
>  
> Resources and signals (JSON):  
> `<resources JSON as string>`  
>  
> Use the thresholds to determine which signals are violations, which are warnings, and which are informational.  
> Then generate a fitness score, risk score, and modernization recommendations.

### 4.3 Payload to Bedrock

Lambda then wraps system + user messages into the model-specific request body required by the Bedrock model (e.g., Claude).

---

## 5. Bedrock Invocation

Lambda calls Bedrock over the **Bedrock Runtime** API.

### 5.1 IAM Requirements

The Lambda execution role must have:

- `bedrock:InvokeModel` (and/or `bedrock:InvokeModelWithResponseStream`)  
  scoped to:
  - the specific region
  - the specific model(s) used by the engine

Lambda should **not** have broad access to other AWS services beyond:

- Reading optional prompt/config templates from S3 (if used).
- Calling Bedrock.
- Writing logs to CloudWatch.

### 5.2 Example Pseudocode

```python
import boto3
import json
import os

bedrock = boto3.client("bedrock-runtime")

def handler(event, context):
    # 1. Receive context from Step Functions
    evaluation_context = event

    # 2. Build system and user prompts
    system_prompt = build_system_prompt()
    user_prompt = build_user_prompt(evaluation_context)

    # 3. Prepare Bedrock request
    body = {
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt}
        ],
        "max_tokens": 1024,
        "temperature": 0.2
    }

    response = bedrock.invoke_model(
        modelId=os.environ["MODEL_ID"],
        body=json.dumps(body)
    )

    # 4. Parse and validate model response
    result = parse_model_response(response["body"])

    # 5. Return normalized result to Step Functions
    return result
```

(The actual implementation will depend on the chosen model and SDK.)

---

## 6. Output Contract (To Step Functions)

Lambda returns a **strict, normalized JSON object** back to Step Functions. Example:

```json
{
  "status": "SUCCESS",
  "fitness_score": 61,
  "risk_score": 78,
  "issues_detected": [
    {
      "resource_id": "lambda:payment-processor",
      "signal_types": ["PERFORMANCE_DEGRADATION", "THROTTLE"],
      "summary": "Function shows sustained high latency and repeated throttling.",
      "severity": "HIGH"
    }
  ],
  "modernization_recommendations": [
    {
      "category": "resilience",
      "pattern": "Introduce SQS buffering",
      "rationale": "High latency and throttles indicate tight coupling and insufficient concurrency headroom."
    },
    {
      "category": "performance",
      "pattern": "Right-size Lambda (more memory)",
      "rationale": "Higher memory will increase CPU and likely reduce execution time and tail latency."
    }
  ],
  "narrative_summary": "The payments workload is currently at medium fitness with significant resilience and performance risks...",
  "raw_model_response": {}
}
```

Step Functions is responsible for:

- Validating this JSON.
- Persisting it to S3.
- Triggering SNS or Jira workflows.

---

## 7. Error Handling and Timeouts

The Lambda Orchestrator must handle:

- Bedrock API timeouts.
- Model parsing errors.
- Invalid/non-JSON model responses.
- Partial responses.

Recommended patterns:

- Use short Lambda timeouts (e.g., 5–10 seconds).
- Wrap Bedrock calls in try/except.
- Return explicit failure shapes:

```json
{
  "status": "ERROR",
  "error_type": "BEDROCK_TIMEOUT",
  "message": "Bedrock did not respond in time."
}
```

Step Functions can then decide:

- Whether to retry.
- Whether to mark the evaluation as failed.
- Whether to send the raw signals to DLQ or manual review.

---

## 8. Security and Compliance

Key assurances:

- Lambda runs inside a **private subnet** when calling Bedrock via VPC endpoints (if configured).
- No raw customer data is written to logs.
- Only aggregated, architecture-level signals are sent to Bedrock.
- The execution role is least-privileged:
  - Bedrock invoke rights.
  - Optional read-only S3 access for templates.
  - CloudWatch logging.

---

## 9. Summary

The Lambda AI Orchestrator is intentionally **thin** and **stateless**:

- It does not own governance or thresholds.
- It does not store or persist evaluations.
- It does not orchestrate multi-step flows.

Its sole purpose is to:

1. Turn Step Functions' evaluation context into a high-quality prompt.
2. Ask Bedrock to reason about architecture fitness and modernization.
3. Return a structured, predictable JSON result back to Step Functions.

This keeps the design **simple**, **auditable**, and **easy to evolve** if you change models, vendors, or prompt strategies in the future.

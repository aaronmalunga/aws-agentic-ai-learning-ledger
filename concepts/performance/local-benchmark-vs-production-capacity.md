# Local Benchmark vs Production Capacity

## Why This Lesson Exists

Reliora measured its local ticket runtime before live AWS verification.

The result is useful performance evidence, but it must not be confused with deployed AWS capacity.

---

# 1. Experiment Scope

The experiment was:

`local-ticket-runtime-v1`

It executed `LambdaBugReportRuntime.handle` locally and in-process using a synthetic workload.

The benchmark measured:

- 100,000 total requests;
- concurrency levels of 1, 2, 4, and 8;
- throughput;
- mean latency;
- p50 latency;
- p95 latency;
- p99 latency;
- failures.

Evidence classification:

`EXPERIMENTAL`

Evidence maturity:

`E3 — Local Synthetic Measured`

---

# 2. Observed Results

Approximate local throughput was:

| Requested concurrency | Requests per second |
|---:|---:|
| 1 | 10,670 |
| 2 | 11,023 |
| 4 | 9,683 |
| 8 | 6,900 |

All 100,000 measured requests completed successfully within the defined local experiment.

These numbers describe the local execution boundary only.

---

# 3. What Was Not Measured

The experiment explicitly excluded:

- AWS Lambda infrastructure;
- Lambda cold starts;
- Lambda scheduling and scaling;
- Amazon Bedrock AgentCore;
- AgentCore Gateway;
- DynamoDB network and service latency;
- AWS throttling;
- cloud cost;
- external network latency.

Therefore:

`local runtime throughput != deployed AWS throughput`

---

# 4. Concurrency Is Not Automatically Scalability

Throughput did not increase continuously as concurrency increased.

Concurrency 2 produced slightly higher throughput than concurrency 1, while concurrency 4 and 8 produced lower throughput.

This supports an important engineering lesson:

> More concurrency does not automatically produce more throughput.

Possible causes such as scheduling overhead, contention, interpreter behaviour, or measurement effects remain hypotheses until separately tested.

---

# 5. Measurement Boundaries Matter

The latency measurement covered runtime handler execution inside each worker.

Executor queue wait was excluded.

Therefore the measured latency is not the same as end-to-end customer latency.

A performance number should always be reported together with the boundary that produced it.

---

# 6. Safe Claim vs Unsafe Claim

Unsafe claim:

> Reliora supports more than 10,000 requests per second.

That would generalize local evidence into unsupported production capacity.

Evidence-bounded claim:

> In a controlled local synthetic benchmark, Reliora's in-process ticket runtime completed 100,000 requests without observed failures and measured approximately 6.9k to 11.0k requests per second across the tested concurrency levels. AWS services, networking, AgentCore, Gateway, DynamoDB, and cloud scaling were outside the measurement boundary.

---

# 7. Engineering Rule

The boundary of measurement defines the boundary of the claim.

Local benchmark evidence is valuable because it can establish a reproducible baseline before cloud testing.

It does not replace target-environment load testing or end-to-end operational evidence.

## Interview Explanation

> I separated local performance evidence from production scalability claims. Reliora's local ticket-runtime benchmark executed 100,000 synthetic in-process requests and measured roughly 6.9k to 11k requests per second across the tested concurrency levels with no observed failures. However, the benchmark excluded Lambda cold starts, AgentCore, Gateway, DynamoDB latency, networking, throttling, and cloud scaling. I therefore classify it as E3 local synthetic measured evidence rather than AWS capacity evidence. It gives me a reproducible local baseline, while a controlled cloud workload test would be required before making deployed scalability claims.

# Idempotency and Retry Safety

## Why This Lesson Exists

Tool-using AI agents interact with systems that can fail partially.

A request may be sent successfully while the caller never receives the response.

For example:

```text
Reliora sends create-ticket request
        ↓
backend creates ticket
        ↓
network timeout occurs
        ↓
Reliora does not receive success response
```

From Reliora's point of view, the result may now be:

```text
unknown
```

A natural recovery strategy is to retry.

But without additional protection:

```text
retry
    ↓
backend creates another ticket
```

The customer may end up with two tickets for one logical request.

Reliora therefore treats retry safety as a first-class reliability requirement.

The central concept is:

```text
idempotency
```

This document explains:

- what idempotency means
- why retries happen
- why retries can create duplicate side effects
- operation identity
- idempotency keys
- duplicate detection
- cached/reused results
- client-generated vs server-generated IDs
- timeout ambiguity
- exactly-once myths
- database constraints
- retry strategy
- observability
- testing
- how idempotency applies to AI agents

---

# 1. What Is Idempotency?

An operation is idempotent when performing the same logical operation more than once has the same intended effect as performing it once.

Conceptually:

```text
operation
    ↓
execute once
    ↓
desired effect

same logical operation
    ↓
execute again
    ↓
no additional unintended effect
```

---

# 2. Simple Example

Suppose the desired state is:

```text
account notification preference = enabled
```

Setting it to:

```text
enabled
```

twice may still produce the same final state:

```text
enabled
```

This operation can naturally be idempotent.

---

# 3. Non-Idempotent Example

Consider:

```text
create a new support ticket
```

Each execution may produce a new database record:

```text
first call
→ ticket A

second call
→ ticket B
```

Even if both requests contain identical information, the side effect happened twice.

This operation is not naturally idempotent.

---

# 4. Why AI Agents Need Idempotency

AI agents often call external tools.

Examples include:

```text
create support ticket
issue refund
send email
update order
cancel subscription
create reservation
submit application
write database record
```

These actions may produce material side effects.

If an agent retries the same action, duplication can cause real problems.

---

# 5. Retries Are Normal

Retries are not automatically evidence that the system is badly designed.

Distributed systems experience failures such as:

```text
network timeout
temporary service error
rate limit
connection reset
Lambda timeout
gateway timeout
temporary database issue
```

Retrying can improve reliability.

The problem is:

> A retry must not accidentally repeat a side effect that already succeeded.

---

# 6. The Classic Ambiguous Timeout

Consider this sequence:

```text
1. Reliora sends CREATE_TICKET
2. Backend receives request
3. Backend writes ticket
4. Backend starts sending response
5. Network connection breaks
6. Reliora receives timeout
```

What does Reliora know?

It knows:

```text
no success response was received
```

It does not know:

```text
whether the backend committed the write
```

This is an ambiguous outcome.

---

# 7. Why Timeout Does Not Mean Failure

A timeout means:

> The caller did not receive a response within the allowed time.

It does not necessarily mean:

> The requested operation did not happen.

The backend may have completed the action.

---

# 8. Three Possible Outcomes

After a timeout:

```text
Possibility A:
request never reached backend
→ no side effect

Possibility B:
backend received request but failed
→ no side effect

Possibility C:
backend succeeded but response was lost
→ side effect exists
```

The caller may not know which one occurred.

This uncertainty makes naive retries dangerous.

---

# 9. Naive Retry

Without idempotency:

```text
CREATE_TICKET
    ↓
ticket A created
    ↓
response lost
    ↓
retry CREATE_TICKET
    ↓
ticket B created
```

One customer request created:

```text
2 tickets
```

This violates Reliora's reliability objective.

---

# 10. Reliora Target

One of Reliora's reliability targets is:

```text
duplicate retry side effects:
0
```

This is currently a:

```text
TARGET
```

until it is implemented and evaluated.

The requirement means:

> Repeating the same logical ticket-creation operation should not create an additional ticket.

---

# 11. Operation Identity

To recognize a retry, the system first needs to know:

> Is this the same logical operation as before?

That requires an operation identity.

Conceptually:

```text
operation_id
```

or:

```text
idempotency_key
```

---

# 12. Idempotency Key

An idempotency key is a stable identifier attached to retries of the same logical operation.

For example:

```text
operation_id = bug-7f319...
```

The first request uses:

```text
operation_id = X
```

If the caller retries the same logical request, it uses:

```text
operation_id = X
```

again.

It must not generate a new key for every network attempt.

---

# 13. Correct Retry Identity

```text
Logical operation:
Create ticket for this completed bug-report workflow

Attempt 1:
operation_id = ABC123

Attempt 2:
operation_id = ABC123

Attempt 3:
operation_id = ABC123
```

The operation identity remains stable.

---

# 14. Incorrect Retry Identity

```text
Attempt 1:
operation_id = ABC123

Attempt 2:
operation_id = DEF456
```

The backend cannot easily know that these attempts represent the same logical operation.

It may create two tickets.

---

# 15. Attempt ID vs Operation ID

These should be distinguished.

```text
operation_id
→ identifies the logical business action

attempt_id
→ identifies one execution attempt
```

Example:

```text
operation_id = ORDER-REFUND-123

attempt 1:
attempt_id = A

attempt 2:
attempt_id = B
```

This allows observability to show multiple attempts without losing the identity of the underlying action.

---

# 16. First Request

A simplified idempotent flow is:

```text
Request:
operation_id = ABC123
        ↓
backend checks store
        ↓
ABC123 not found
        ↓
perform operation
        ↓
store result under ABC123
        ↓
return result
```

---

# 17. Retry

If the same request is retried:

```text
Request:
operation_id = ABC123
        ↓
backend checks store
        ↓
ABC123 already processed
        ↓
do NOT create second ticket
        ↓
return original result
```

The duplicate attempt becomes safe.

---

# 18. Reusing the Original Result

A strong idempotency system often stores:

```text
operation status
result
resource ID
request fingerprint
timestamp
```

For example:

```text
operation_id:
ABC123

status:
SUCCEEDED

ticket_id:
TICKET-42
```

A retry can return:

```text
TICKET-42
```

instead of creating:

```text
TICKET-43
```

---

# 19. Why Returning the Same Result Matters

The goal is not only:

```text
do not create duplicate
```

It is also helpful for the caller to learn:

```text
the original operation succeeded
```

This resolves uncertainty after a timeout.

---

# 20. Reliora Ticket Example

Desired flow:

```text
Customer completes bug details
        ↓
Reliora generates stable operation_id
        ↓
TicketProvider.create(...)
        ↓
backend checks operation_id
        ↓
ticket created once
        ↓
ticket ID stored with operation
        ↓
backend returns real ticket ID
```

If retry occurs:

```text
same operation_id
        ↓
existing success found
        ↓
same ticket ID returned
```

---

# 21. The Ticket ID Is Not the Idempotency Key

These are different concepts.

```text
idempotency key
→ identifies the request before execution

ticket ID
→ identifies the created resource after execution
```

Before the ticket exists, the application cannot rely on the ticket ID as the operation identity.

---

# 22. Client-Generated Operation ID

One common pattern is for the caller to generate the idempotency key before sending the request.

For example:

```text
Reliora
→ creates operation_id

backend
→ enforces uniqueness for operation_id
```

This works well because retries can reuse the same identifier.

---

# 23. Why Random UUID Per Attempt Is Not Enough

Suppose every call does:

```python
operation_id = uuid4()
```

immediately before sending the request.

Then:

```text
first attempt
→ ID A

retry
→ ID B
```

The identifiers are unique, but they do not provide retry idempotency.

Uniqueness and idempotency are different.

---

# 24. Uniqueness vs Idempotency

```text
Uniqueness:
Every attempt has a different ID.

Idempotency:
Every attempt for the same logical operation shares an ID.
```

For retry safety, we need the second property.

---

# 25. Stable Operation Boundary

A key design question is:

> What exactly counts as the same logical operation?

For Reliora, it might be:

```text
one completed bug-report submission
```

rather than:

```text
one HTTP request
```

Operation identity should reflect business semantics, not transport mechanics.

---

# 26. Transport Attempt vs Business Action

A network request is an attempt.

A business action is the thing the user intended.

For example:

```text
User intent:
Create one bug ticket

Transport:
HTTP request #1
HTTP request #2
HTTP request #3
```

All three transport attempts may belong to one logical operation.

---

# 27. Idempotency at the Correct Layer

If idempotency is implemented only in the UI, another caller could bypass it.

If implemented only in the LLM prompt:

```text
Do not create duplicates.
```

the model may still retry.

The enforcement should exist near the side-effect boundary.

---

# 28. Backend Enforcement

A stronger design is:

```text
application sends operation_id
        ↓
backend persists operation record
        ↓
backend enforces uniqueness
```

This protects the side effect regardless of whether the caller is:

```text
LLM
API
UI
retry worker
integration test
```

---

# 29. Database Uniqueness

A database can help enforce idempotency using a unique key.

Conceptually:

```text
operation_id
→ unique
```

If two concurrent requests attempt to create the same operation record:

```text
only one can own the key
```

This reduces race conditions.

---

# 30. Why Check-Then-Create Alone Can Be Unsafe

Naive logic:

```text
if operation_id not found:
    create ticket
```

can fail under concurrency.

Two requests may execute simultaneously:

```text
Request A:
checks → not found

Request B:
checks → not found

Request A:
creates ticket

Request B:
creates ticket
```

Both passed the check before either write became visible.

---

# 31. Race Condition

This is a race condition.

A race condition occurs when correctness depends on timing between concurrent operations.

Idempotency should therefore use atomic or transactional controls where appropriate.

---

# 32. Atomic Operation

An atomic operation behaves as one indivisible unit from the perspective of competing operations.

For example:

```text
create operation record only if operation_id does not already exist
```

should ideally happen atomically.

---

# 33. DynamoDB Example Concept

Reliora is expected to use DynamoDB for some backend state.

A DynamoDB conditional write could conceptually enforce:

```text
create item only if operation_id does not already exist
```

This is stronger than:

```text
read first
then write later
```

because the uniqueness decision happens with the write.

---

# 34. Conditional Writes

Conceptually:

```text
Put operation ABC123
IF ABC123 does not already exist
```

If another request already created it:

```text
conditional write fails
```

The application can then fetch/reuse the existing result.

---

# 35. Why This Is Useful

It handles situations where:

```text
two retries arrive nearly simultaneously
```

without relying only on application timing.

---

# 36. In-Progress Operations

A retry may occur while the first operation is still running.

The idempotency record might contain:

```text
status = IN_PROGRESS
```

The retry now needs a defined policy.

Possible responses include:

```text
wait
return operation still in progress
poll later
retry after delay
```

It should not automatically start another side effect.

---

# 37. Example State Model

An idempotency record may move through:

```text
NEW
 ↓
IN_PROGRESS
 ↓
SUCCEEDED
```

or:

```text
NEW
 ↓
IN_PROGRESS
 ↓
FAILED
```

The exact design depends on the operation.

---

# 38. Why `FAILED` Needs Care

If an operation is marked:

```text
FAILED
```

we need to know:

> Did the business side effect definitely fail?

If the backend state is uncertain, blindly allowing another execution may still produce a duplicate.

Failure state should distinguish:

```text
confirmed no side effect
```

from:

```text
outcome unknown
```

where necessary.

---

# 39. Unknown Outcome

A useful explicit state may be:

```text
UNKNOWN
```

or equivalent.

This communicates:

> We cannot safely prove whether the operation committed.

This can trigger:

```text
reconciliation
lookup
human review
safe retry protocol
```

instead of guessing.

---

# 40. Truthful State Matters

Distributed systems should not convert:

```text
unknown
```

into:

```text
failed
```

or:

```text
succeeded
```

without evidence.

Reliora should preserve uncertainty when necessary.

---

# 41. Idempotency Is Not Exactly-Once Delivery

A common phrase is:

```text
exactly once
```

But distributed systems make true exactly-once execution difficult.

More practical designs often use:

```text
at-least-once delivery
+
idempotent processing
```

to achieve effectively-once business effects.

---

# 42. At-Most-Once

```text
at-most-once
```

means an operation is attempted no more than once.

This avoids duplicates but can lose actions if the attempt fails.

---

# 43. At-Least-Once

```text
at-least-once
```

means the system may retry until it believes the action was processed.

This improves completion probability but can create duplicates unless processing is idempotent.

---

# 44. Effectively-Once Outcome

A common practical goal is:

```text
requests may be delivered multiple times
```

but:

```text
business side effect occurs once
```

Idempotency helps provide this behaviour.

---

# 45. Idempotency Does Not Mean "Never Retry"

The purpose is not to prevent retries.

It is to make retries safer.

Conceptually:

```text
failure
→ retry allowed
→ same operation identity
→ no duplicate side effect
```

---

# 46. Retry Policy

A retry policy defines:

```text
which errors are retryable
how many times to retry
how long to wait
whether to use backoff
whether to add jitter
when to stop
```

Idempotency and retry policy work together.

---

# 47. Not Every Error Should Be Retried

For example:

```text
timeout
→ possibly retryable

HTTP 503 / temporary unavailable
→ possibly retryable

invalid request schema
→ not retryable until corrected

authorization denied
→ not retryable automatically

policy blocked
→ not retryable automatically
```

Retrying permanent errors only wastes resources.

---

# 48. Transient vs Permanent Failure

A useful distinction is:

```text
Transient failure
→ condition may resolve

Permanent failure
→ same request will keep failing until something changes
```

Retries are primarily intended for transient failures.

---

# 49. Exponential Backoff

Instead of retrying immediately:

```text
retry
retry
retry
retry
```

a system may wait progressively longer.

Conceptually:

```text
attempt 1
wait 1 second

attempt 2
wait 2 seconds

attempt 3
wait 4 seconds

attempt 4
wait 8 seconds
```

This is exponential backoff.

---

# 50. Why Backoff Helps

If a service is temporarily overloaded, immediate repeated retries can worsen the overload.

Backoff reduces pressure.

---

# 51. Jitter

Jitter adds randomness to retry timing.

Instead of every client retrying at exactly:

```text
2 seconds
```

they retry across a small time range.

This reduces synchronized retry storms.

---

# 52. Retry Storm

Suppose thousands of requests fail simultaneously.

Without jitter:

```text
all clients wait 2 seconds
        ↓
all retry simultaneously
        ↓
service overloaded again
```

Jitter spreads requests over time.

---

# 53. AI Agents Can Create Retry Storms Too

Agent workflows may automatically retry:

```text
model calls
retrieval calls
tool calls
```

Bounded retry behaviour is therefore important for:

```text
reliability
cost
latency
service protection
```

---

# 54. Retry Budget

A retry budget limits how much retry activity is allowed.

For example:

```text
maximum 3 attempts
```

or:

```text
maximum total retry duration = 10 seconds
```

This prevents infinite retry loops.

---

# 55. Idempotency and Cost

Without idempotency, duplicate operations may create:

```text
duplicate Lambda calls
duplicate database writes
duplicate downstream API charges
duplicate customer actions
```

Retry safety therefore also supports FinOps.

---

# 56. Idempotency and User Experience

Duplicate support tickets can confuse customers.

For example:

```text
Ticket 1273 created
Ticket 1274 created
```

for one issue.

The customer may receive:

```text
two confirmation emails
two support threads
two status updates
```

Reliability problems become visible product problems.

---

# 57. Idempotency and Operations

Duplicate tickets also affect support operations:

```text
duplicate workload
conflicting agents
incorrect metrics
inflated ticket volume
```

The problem extends beyond technical correctness.

---

# 58. Idempotency and Analytics

Duplicate records can corrupt business metrics.

For example:

```text
real bug reports = 100
duplicate retries = 20
database records = 120
```

Analytics may incorrectly conclude:

```text
120 unique incidents
```

Operational data quality therefore depends on retry safety.

---

# 59. Idempotency and Financial Actions

The stakes become much higher for operations such as:

```text
refund
payment
credit
transfer
```

A duplicate execution can create direct financial loss.

This is why idempotency is a common payment-system principle.

---

# 60. Stripe-Style Mental Model

Payment APIs commonly demonstrate the pattern:

```text
same idempotency key
+
same logical request
→ same logical result
```

Reliora applies the same principle to agent side effects.

---

# 61. Request Fingerprinting

A system may store not only:

```text
operation_id
```

but also a fingerprint of the request parameters.

Why?

To detect misuse such as:

```text
same idempotency key
but different operation content
```

---

# 62. Example Conflict

First request:

```text
operation_id = ABC123
description = checkout error
```

Second request:

```text
operation_id = ABC123
description = payment fraud
```

These may not represent the same logical operation.

The backend should not silently return the first result without noticing the conflict.

---

# 63. Request Hash

A request fingerprint can be calculated from canonicalized operation parameters.

Conceptually:

```text
canonical request
      ↓
SHA-256
      ↓
request fingerprint
```

The stored idempotency record can include this value.

---

# 64. Retry Validation

On retry:

```text
operation_id matches
        ↓
request fingerprint matches?
       / \
     yes  no
     ↓     ↓
reuse   conflict/error
result
```

This prevents accidental key reuse across different operations.

---

# 65. Canonicalization Matters Again

If request fingerprints use hashes, logically identical requests should have consistent canonical representations.

For structured data, this may involve:

```text
stable field ordering
normalized encoding
defined serialization
```

The line-ending hash lesson therefore connects conceptually to idempotency.

---

# 66. Idempotency TTL

Stored idempotency records do not necessarily need to live forever.

A system may define a retention period.

For example:

```text
retain operation records for N hours/days
```

The correct duration depends on:

```text
retry window
business workflow
audit requirements
cost
compliance
```

---

# 67. Why TTL Must Be Justified

If the record expires too soon:

```text
late retry
→ operation key missing
→ duplicate side effect possible
```

If records live forever:

```text
unnecessary storage
retention burden
privacy concerns
```

Retention is an architectural decision.

---

# 68. Idempotency and PII

The idempotency store should not automatically retain entire user prompts.

Often it only needs:

```text
operation ID
request fingerprint
status
result reference
timestamps
```

Data minimization still applies.

---

# 69. Do Not Use Sensitive Data as the Idempotency Key

A key such as:

```text
customer-email + bug-description
```

may expose personal or sensitive information.

A generated opaque operation ID is usually safer.

---

# 70. Logging Idempotency

Useful telemetry could include:

```text
operation_id
attempt_id
idempotency_status
tool_result
latency
```

For example:

```text
idempotency_status = NEW
```

or:

```text
idempotency_status = REPLAY
```

This makes retry behaviour observable.

---

# 71. Avoid Logging Excessive Payloads

Operational logs should not include unnecessary:

```text
customer PII
full bug descriptions
tokens
credentials
```

just because idempotency is being traced.

Structured metadata is preferable.

---

# 72. Example Trace

```text
operation_id = ABC123
attempt_id = 1
idempotency = NEW
tool_call = STARTED
tool_result = TIMEOUT
```

Retry:

```text
operation_id = ABC123
attempt_id = 2
idempotency = REPLAY
stored_status = SUCCEEDED
ticket_id = TICKET-42
```

This explains what happened without creating a second ticket.

---

# 73. Idempotency Metrics

Future operational metrics may include:

```text
idempotency replay count
duplicate-prevention count
idempotency conflicts
in-progress retry count
operation failure count
```

These can help validate the control in real execution.

---

# 74. Testing Idempotency

Reliora should not claim retry safety merely because the architecture includes an idempotency key.

It must be tested.

A basic integration test could simulate:

```text
same operation
sent twice
```

and verify:

```text
one ticket created
same ticket ID returned
```

---

# 75. Test Case

Conceptually:

```text
operation_id = X

first call
→ ticket_id = 123

second call
→ ticket_id = 123

database tickets created
→ exactly 1
```

This is stronger evidence than inspecting code alone.

---

# 76. Concurrent Retry Test

A stronger test sends multiple attempts concurrently:

```text
attempt A ─┐
attempt B ─┼→ same operation_id
attempt C ─┘
```

Expected:

```text
one side effect
all attempts resolve consistently
```

This tests race-condition protection.

---

# 77. Timeout Test

A future fault-injection test could simulate:

```text
backend succeeds
response is lost
```

Then retry with the same key.

Expected:

```text
no duplicate ticket
original result recovered
```

This directly tests the classic ambiguous outcome.

---

# 78. Failure-Before-Commit Test

Simulate:

```text
request received
backend fails before ticket write
```

Retry should be allowed to execute the operation successfully.

This checks that idempotency does not incorrectly freeze a failed operation forever.

---

# 79. Failure-After-Commit Test

Simulate:

```text
ticket committed
response lost
```

Retry should reuse the original successful result.

This is one of the most important cases.

---

# 80. Key-Reuse Conflict Test

Use:

```text
same operation_id
```

with:

```text
different request payload
```

Expected:

```text
conflict
```

rather than silently reusing an unrelated result.

---

# 81. Expired-Key Test

If TTL is implemented, test what happens after expiration.

The behaviour must match documented policy.

For sensitive operations, a late retry after expiry may require:

```text
new confirmation
reconciliation
manual review
```

rather than automatic execution.

---

# 82. Idempotency Is a Business Contract

The question is not merely:

> Did our database avoid duplicate rows?

The stronger question is:

> Did one user-intended operation produce one intended business effect?

That is the true idempotency boundary.

---

# 83. Duplicate Database Row vs Duplicate Business Effect

A system might avoid duplicate ticket rows but still send:

```text
two emails
two notifications
two downstream events
```

Then the overall operation is not fully retry-safe.

The complete side-effect chain must be considered.

---

# 84. Multi-Step Operations

Suppose ticket creation does:

```text
1. write ticket
2. publish event
3. send email
```

Idempotency becomes more complex.

A retry must not accidentally create:

```text
duplicate event
duplicate email
```

even if the ticket row is unique.

---

# 85. Distributed Transaction Difficulty

Multiple systems may not support one global transaction.

This is why distributed workflows often need patterns such as:

```text
outbox
deduplication
event IDs
workflow state
compensation
```

Reliora should introduce these only when requirements justify them.

---

# 86. Do Not Over-Engineer P1

For Reliora's current ticket-creation workflow, the first implementation should use the simplest control that satisfies the requirement.

Likely:

```text
stable operation ID
+
conditional backend persistence
+
result reuse
+
tests
```

rather than introducing a complex distributed transaction platform prematurely.

---

# 87. Boring Technology Principle

A small DynamoDB idempotency record may solve the requirement better than adding:

```text
Kafka
Redis
complex workflow engines
distributed locks
```

The goal is the reliability property, not infrastructure prestige.

---

# 88. Idempotency vs Deduplication

These concepts are related but not identical.

```text
Deduplication
→ identify repeated data/events

Idempotency
→ repeated execution produces no additional unintended effect
```

Deduplication can be one mechanism used to implement idempotency.

---

# 89. Content-Based Deduplication

A system might compare:

```text
same customer
same bug description
same time window
```

and decide two requests look similar.

This is weaker than an explicit operation ID.

Two legitimate separate incidents may look identical.

---

# 90. Explicit Intent Is Better

If the application knows:

```text
this is a retry of operation X
```

that is stronger than guessing from similar content.

Operation identity should therefore be explicit where possible.

---

# 91. Why an LLM Should Not Decide Duplicate Identity Alone

An LLM might say:

```text
These two bug reports look like the same issue.
```

That semantic judgment can be useful.

But it should not be the sole enforcement mechanism for retry idempotency.

Retry identity should come from system state.

---

# 92. Semantic Duplicate Detection Is a Different Feature

Later, Reliora could theoretically detect:

```text
two different customer submissions that describe the same underlying bug
```

That is semantic deduplication.

It is not the same as:

```text
retrying one operation
```

These should remain separate concepts.

---

# 93. Idempotency and Human Handoff

Suppose tool execution is uncertain.

Instead of unsafe repeated execution, the system may route to human support with:

```text
operation ID
current status
known backend state
```

This allows manual resolution without duplicate action.

---

# 94. Idempotency and Recovery

A robust system should be able to answer:

```text
What happened to operation X?
```

Possible answers:

```text
never started
in progress
succeeded
failed safely
outcome unknown
```

This becomes part of operational recovery.

---

# 95. Reconciliation

Reconciliation checks system state after uncertainty.

For example:

```text
timeout occurred
        ↓
query ticket store using operation_id
        ↓
ticket found
        ↓
recover success result
```

This is stronger than blindly retrying.

---

# 96. Retry Before Reconciliation?

Depending on the system, the best sequence may be:

```text
timeout
    ↓
check idempotency state
    ↓
if succeeded:
return original result

if safe to retry:
retry

if unknown:
reconcile or handoff
```

The exact policy must be designed.

---

# 97. Idempotency and Observability

An idempotent system is easier to operate if engineers can trace:

```text
logical operation
```

across:

```text
multiple attempts
tool call
database write
response
```

This is why `operation_id` can also serve as a correlation identifier.

---

# 98. Correlation ID vs Idempotency Key

These can be the same value in simple systems, but conceptually they differ.

```text
Correlation ID
→ traces related activity

Idempotency key
→ prevents duplicate business effect
```

One identifier may sometimes serve both roles, but the semantics should remain clear.

---

# 99. Idempotency and Event Processing

If Reliora later uses asynchronous events, consumers may receive the same event more than once.

A consumer can store:

```text
event_id
```

and ignore/reuse previously processed events.

This is the same general principle applied to event-driven architecture.

---

# 100. Delivery Guarantees Are Not Business Guarantees

A messaging system may advertise delivery semantics.

But application-level idempotency is still important because the business operation may involve additional systems.

Reliability cannot be outsourced entirely to transport guarantees.

---

# 101. Idempotency and LLM Replanning

An agent may replan after an error.

For example:

```text
tool call timed out
        ↓
LLM reasons:
"Try creating the ticket again."
```

If tool calls are protected by operation identity, this retry can be safe.

Without it, agent replanning can duplicate side effects.

---

# 102. The Model Should Not Generate a New Operation ID on Every Replan

Operation identity should live in deterministic application state.

The LLM should not casually replace it during reasoning.

---

# 103. Workflow State Owns Operation Identity

A stronger design is:

```text
application creates operation_id
        ↓
stores it in workflow state
        ↓
all tool attempts reuse it
```

The model can see it only if necessary.

Often it does not need to.

---

# 104. Do Not Expose Operation IDs to Customers Unnecessarily

An internal idempotency key is usually implementation metadata.

The customer may only need:

```text
ticket ID
```

after success.

This supports separation between:

```text
internal control identifiers
```

and:

```text
customer-facing identifiers
```

---

# 105. Security of Idempotency Keys

An idempotency key should generally be difficult to guess when it exposes operational APIs.

Otherwise attackers might intentionally collide with another operation.

Random UUIDs are commonly suitable as opaque identifiers when generated once per logical operation.

---

# 106. Authorization Still Applies on Retry

A retry with a valid idempotency key should not bypass authorization.

For example:

```text
operation_id is valid
```

does not automatically mean:

```text
caller is authorized to retrieve the original result
```

Security boundaries remain.

---

# 107. Same Key Does Not Equal Permission

Idempotency answers:

> Is this the same operation?

Authorization answers:

> May this caller perform or observe it?

These remain separate controls.

---

# 108. Idempotency Storage Failure

What if the idempotency store itself is unavailable?

For a sensitive write, automatically proceeding without protection may be unsafe.

A fail-closed policy may be appropriate:

```text
cannot verify idempotency
→ do not perform protected side effect
```

depending on risk.

---

# 109. Availability Trade-Off

Failing closed can reduce availability.

But for some actions:

```text
temporary inability to create ticket
```

may be preferable to:

```text
creating unknown duplicates
```

This is a trade-off that should be explicit.

---

# 110. Reliability Is About Correct Trade-Offs

The goal is not:

```text
always succeed
```

The goal is:

```text
behave predictably under failure
preserve correctness
communicate uncertainty
recover safely
```

Sometimes safe failure is better than uncontrolled success.

---

# 111. Evidence for Idempotency

Once implemented, Reliora should preserve evidence such as:

```text
number of retry cases
number of side effects
operation IDs
returned ticket IDs
test outcome
```

Example future claim:

```text
EXPERIMENTAL:
Across 20 controlled duplicate-retry cases,
20 logical operations produced 20 tickets,
with 0 duplicate ticket creations.
```

This is stronger than:

```text
idempotency implemented
```

---

# 112. Denominator Matters Again

If we test:

```text
20 cases
```

say:

```text
0 duplicates across 20 tested retry scenarios
```

Do not say:

```text
duplicates are impossible
```

Evidence scope remains important.

---

# 113. Idempotency Failure Should Become a Regression Case

If a duplicate bug is ever discovered:

```text
preserve failing scenario
        ↓
create regression test
        ↓
fix implementation
        ↓
keep test permanently
```

Known distributed-systems failures should become institutional memory just like AI behavioural failures.

---

# 114. Idempotency Is Not an AI-Specific Concept

This concept comes from distributed systems, APIs, payments, and backend engineering.

Reliora needs it because AI agents are now interacting with those systems.

Strong agent engineering therefore requires conventional systems knowledge.

---

# 115. AI Engineering Is Systems Engineering

A tool-using agent combines:

```text
LLM behaviour
API semantics
distributed failures
security
identity
databases
observability
cost
```

Understanding only prompt engineering is insufficient for reliable agentic systems.

---

# 116. A Complete Reliora Retry Flow

Conceptually:

```text
Customer completes bug report
        ↓
application creates operation_id = X
        ↓
validate all required fields
        ↓
authorize ticket creation
        ↓
check idempotency record X
        ↓
not present
        ↓
atomically claim X
        ↓
create ticket
        ↓
store ticket ID with X
        ↓
response lost
        ↓
retry with X
        ↓
existing successful record found
        ↓
return original ticket ID
        ↓
no duplicate ticket
```

This is the behaviour we ultimately want to validate.

---

# 117. Important Lessons

1. Retries are normal in distributed systems.
2. A timeout does not prove that a side effect failed.
3. Blind retries can create duplicate business actions.
4. Idempotency makes repeated attempts of the same logical operation safe.
5. The same logical operation must reuse the same idempotency key.
6. Generating a new UUID for every attempt provides uniqueness, not idempotency.
7. Operation IDs and attempt IDs represent different concepts.
8. The operation identity should reflect business intent rather than network requests.
9. The backend should enforce idempotency near the side-effect boundary.
10. Atomic or conditional writes help prevent concurrent race conditions.
11. Check-then-create logic alone may fail under concurrency.
12. Successful idempotency replays should generally return the original result.
13. Ticket IDs and idempotency keys are different identifiers.
14. Request fingerprints can detect reuse of one key with different request content.
15. In-progress, failed, successful, and unknown operation states may require different handling.
16. Unknown outcomes should not be falsely reported as success or failure.
17. At-least-once retries plus idempotent processing can provide effectively-once business effects.
18. Not every error should be retried.
19. Exponential backoff, jitter, and retry limits reduce retry storms.
20. Idempotency improves reliability, user experience, data quality, and cost control.
21. Idempotency must consider the entire business side effect, not only one database row.
22. Semantic duplicate detection is different from retry idempotency.
23. Authorization must still be enforced on idempotent retries.
24. The idempotency store itself becomes part of the reliability boundary.
25. Retry safety should be demonstrated through controlled integration and fault-injection tests rather than claimed from architecture alone.

---

## Interview Explanation

> I treat idempotency as an operation-level contract rather than simply generating unique request IDs. For a Reliora ticket-creation workflow, the application creates one stable operation ID for the customer's logical submission and reuses it across retries. The backend atomically records that operation, performs the side effect once, and stores the resulting ticket ID. If a timeout causes the caller to retry after the original write succeeded, the backend recognizes the same operation and returns the original result instead of creating a duplicate ticket. I would verify this with duplicate, concurrent, and timeout-after-commit test cases rather than treating the presence of an idempotency key as proof that retry safety works.


---

# Implementation Update - Durable DynamoDB Idempotency

**Date:** 2026-09-03

**Reliora commit:** `24d544a - feat: establish durable AWS ticket execution foundation`

The earlier sections describe the idempotency behaviour Reliora requires.

That behaviour is now partially implemented for AWS through a DynamoDB-backed execution adapter.

## Local Reference Behaviour

The original local implementation established the required semantics using:

```text
InMemoryIdempotencyRegistry
InMemoryTicketResultStore
TicketProvider
```

The execution sequence was conceptually:

```text
check operation
-> execute provider
-> save authoritative result
```

This was useful for establishing and testing application behaviour.

However, correctness inside one Python process does not automatically provide distributed correctness.

## Why the Local Sequence Was Not Enough for AWS

In a distributed environment, a sequence such as:

```text
check
-> create
-> save
```

contains failure windows.

For example:

```text
worker A checks operation X
worker B checks operation X
both observe X as absent
both attempt the side effect
```

A process could also fail between individual persistence operations.

Therefore the AWS implementation could not simply replace the in-memory dictionaries with independent DynamoDB calls.

The behavioural contract had to remain the same while the persistence mechanics changed.

## Durable AWS Execution Boundary

Reliora now has:

```text
TicketExecutionPort
        |
        +-> local TicketExecutionCoordinator
        |
        +-> DynamoDBTicketExecutionCoordinator
```

The application service depends on the execution capability rather than on one persistence implementation.

The DynamoDB implementation stores two distinct forms of state:

```text
operations table
-> logical operation identity
-> request fingerprint
-> authoritative ticket reference

tickets table
-> authoritative ticket business record
```

This keeps retry-control state distinct from ticket business state.

## Atomic New-Operation Creation

For a new operation, the AWS adapter uses DynamoDB `TransactWriteItems`.

Conceptually:

```text
operation_id = O1
fingerprint = F1
ticket_id = T1

TransactWriteItems
    |
    +-> Put operation O1
    |      condition:
    |      attribute_not_exists(operation_id)
    |
    +-> Put ticket T1
           condition:
           attribute_not_exists(ticket_id)
```

The transaction provides an important property:

```text
both records commit
or
neither record commits
```

The operation condition also acts as the distributed concurrency boundary.

Only one concurrent request can successfully claim a previously unused logical `operation_id`.

## Matching Retry

If durable state already contains:

```text
operation_id = O1
fingerprint = F1
ticket_id = T1
```

and a retry arrives with:

```text
operation_id = O1
fingerprint = F1
```

Reliora returns:

```text
REPLAYED
ticket_id = T1
```

No second ticket write is attempted.

## Conflicting Retry

If the same logical operation identity is reused with changed business data:

```text
stored:
O1 + F1

incoming:
O1 + F2
```

and:

```text
F1 != F2
```

Reliora returns:

```text
CONFLICT
```

and does not create another ticket.

This prevents one idempotency identity from silently being reused for a different business action.

## Concurrent Race Reconciliation

Two workers can both initially observe that an operation does not exist.

For example:

```text
worker A             worker B
   |                    |
read O1 absent       read O1 absent
   |                    |
transaction A        transaction B
   |                    |
succeeds             condition loses
   |                    |
O1 + T1 committed    reread O1
```

The losing worker does not blindly retry the side effect.

It reconciles against authoritative DynamoDB state.

If the stored fingerprint matches:

```text
REPLAYED
```

If it differs:

```text
CONFLICT
```

If authoritative state still cannot be established:

```text
UNCONFIRMED
```

## Why Strongly Consistent Reads Are Used

The reconciliation read uses:

```text
ConsistentRead=True
```

At the idempotency boundary, immediate knowledge of current authoritative operation state is more important than optimizing a small read-cost difference.

This is a targeted consistency decision rather than a claim that every DynamoDB read in Reliora requires strong consistency.

## Ambiguous Outcomes Stay Explicit

A cancelled or ambiguous transaction does not automatically mean:

```text
ticket creation failed
```

and it does not mean:

```text
ticket creation succeeded
```

Reliora rereads authoritative operation state.

If no trustworthy result can be reconstructed, the execution result is:

```text
UNCONFIRMED
```

This preserves the existing rule:

> Do not guess when backend truth is unavailable.

## Deterministic AWS Adapter Tests

The AWS adapter is tested without requiring live AWS.

A fake DynamoDB client records and controls:

```text
GetItem
TransactWriteItems
AWS transaction errors
authoritative read responses
```

The clock and ticket ID generator are also injectable.

Seven tests currently verify:

```text
new operation creates both records atomically
matching retry replays without another transaction
conflicting retry is rejected without a write
matching concurrency race reconciles to replay
conflicting concurrency race reconciles to conflict
ambiguous cancelled transaction returns UNCONFIRMED
incomplete durable state returns UNCONFIRMED
```

The AWS-adapter test result was:

```text
7 passed
```

## Behavioural Regression Evidence

Before introducing the cloud execution abstraction, the core reliability contract produced:

```text
23 passed
```

After introducing `TicketExecutionPort`, the same contract suite again produced:

```text
23 passed
```

After adding the DynamoDB adapter and its seven tests, the complete Reliora suite produced:

```text
171 passed
```

using:

```powershell
uv run python -m pytest
```

This supports the claim that the distributed execution implementation was introduced without breaking behaviours represented by the existing test suite.

## What This Evidence Does Not Yet Prove

The current evidence does not yet prove:

```text
live DynamoDB behaviour
real Lambda concurrency
IAM correctness
AgentCore Gateway integration
network failure behaviour in AWS
production-scale concurrency
```

Those require live cloud verification.

The implementation should therefore be described as:

```text
locally tested distributed-state design
```

not:

```text
production-proven exactly-once system
```

## Updated Engineering Lesson

The most important lesson from this implementation is:

> Idempotency is not provided merely by having an idempotency key. The storage and execution boundary must enforce the logical operation identity safely under retries, concurrency, partial failure, and ambiguous outcomes.

The local implementation established the semantics.

The DynamoDB implementation introduced the distributed mechanics required to preserve those semantics across independent workers.

## Updated Interview Explanation

> I first implemented Reliora's ticket idempotency semantics locally so I could define and test what new, matching, conflicting, and unconfirmed operations meant. When moving to AWS, I did not simply replace the in-memory dictionaries with DynamoDB calls because the local check-create-save sequence was not atomic under concurrency. I introduced an execution port and a DynamoDB-backed implementation that uses conditional transactional writes to atomically claim the logical operation and create the ticket record. If another invocation loses the race, it performs a strongly consistent reconciliation read and returns replay, conflict, or unconfirmed based on authoritative state. I tested seven AWS-specific reliability cases locally and then ran the full 171-test regression suite. Live AWS integration is still a separate evidence gate, so I do not treat the local tests as proof of production concurrency behaviour.
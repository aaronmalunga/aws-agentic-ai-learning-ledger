# Reliora Runtime Safety Foundation

## Context

This learning note captures the reusable engineering lessons from building the local runtime safety foundation for Reliora before moving into AWS infrastructure.

Reliora is a production-oriented AI customer-support reliability and control platform.

The local runtime phase ended at Reliora commit:

`cc02e58 — feat: enforce authorization before ticket execution`

At that checkpoint:

- the Reliora working tree was clean;
- the runtime foundation had been merged into `main`;
- cloud work moved to `feat/aws-infrastructure`;
- the full local regression suite passed with **164 tests**.

This note is not intended to duplicate the Reliora implementation repository.

Its purpose is to preserve the underlying engineering concepts so they can be reused in future agentic AI systems and explained in interviews.

---

# 1. Separate Request, Session, and Operation Identity

A production agent often needs more than one identifier.

Reliora distinguishes:

| Identifier | Represents | Example lifecycle |
|---|---|---|
| `request_id` | One individual request or turn | Changes on every retry/request |
| `session_id` | One broader customer conversation | Shared across related turns |
| `operation_id` | One logical side effect | Stable across retries |

The key lesson is:

> A retry is often a new request but not a new business operation.

For example:

    Request 1
      request_id = R1
      session_id = S1
      operation_id = O1

    Retry
      request_id = R2
      session_id = S1
      operation_id = O1

The retry has a new request identity but refers to the same logical side effect.

This distinction is essential when preventing duplicate writes.

---

# 2. LLM Output Is Not Authorization

An LLM may:

- classify a request;
- extract structured information;
- recommend an action;
- construct proposed tool arguments.

That does not mean the LLM should be allowed to authorize a persistent side effect.

Reliora separates:

    model proposal
        ↓
    deterministic validation
        ↓
    authorization decision
        ↓
    side-effect execution

The reusable principle is:

> Probabilistic reasoning may propose; deterministic software authorizes.

Examples of deterministic controls include:

- schema validation;
- required-field validation;
- risk policy;
- idempotency;
- approved-tool checks;
- backend-result validation.

This reduces the possibility that prompt injection, hallucination, or malformed model output directly triggers a persistent action.

---

# 3. Authorization Must Happen Before Idempotency Admission

The order of controls matters.

Consider an incomplete logical operation:

    operation_id = O1
    description = present
    reproduction steps = missing
    environment = missing

If O1 were registered in the idempotency system before validation, the following could happen:

    incomplete O1
        ↓
    idempotency record created
        ↓
    customer supplies missing fields
        ↓
    O1 now contains a different payload
        ↓
    false idempotency conflict

Reliora therefore uses:

    operation
        ↓
    authorization preconditions
        ↓
    DENIED
        ↓
    stop

Only eligible operations reach:

    idempotency admission

This preserves the meaning of the operation identity.

The broader lesson is:

> Validation and authorization order can materially change reliability behaviour.

---

# 4. Idempotency Is More Than Deduplication

Reliora models three basic admission outcomes.

## New operation

    operation_id = O1
    payload = P1
        ↓
    no previous O1
        ↓
    NEW_OPERATION

This operation may proceed toward execution.

## Matching retry

    operation_id = O1
    payload = P1
        ↓
    O1 already exists with P1
        ↓
    MATCHING_OPERATION

This is the safe-retry path.

## Conflict

    operation_id = O1
    payload = P2
        ↓
    O1 already exists with P1
        ↓
    CONFLICT

The system must not silently reinterpret the same logical operation as a different action.

The reusable lesson is:

> An idempotency key identifies a logical operation, not simply a request.

---

# 5. Canonical Payload Fingerprints

Reliora compares logical operation payloads using a deterministic fingerprint.

Conceptually:

    normalized payload
        ↓
    canonical serialization
        ↓
    SHA-256
        ↓
    deterministic fingerprint

The fingerprint intentionally excludes request identity because a retry may legitimately have a different `request_id`.

The operation identifier is the lookup key.

The business payload determines whether the retry still represents the same logical operation.

Important:

> A SHA-256 fingerprint is a comparison mechanism. It is not encryption or anonymization.

---

# 6. Idempotency Admission and Result Storage Are Different Responsibilities

A useful architectural separation emerged during implementation.

The idempotency registry answers:

> Have I seen this operation before, and does the payload match?

The result store answers:

> Has this operation already received an authoritative confirmed result?

These responsibilities should not be collapsed.

Conceptually:

    Idempotency Registry
        ↓
    operation identity + payload relationship

    Ticket Result Store
        ↓
    authoritative backend outcome

This separation makes a future persistent implementation easier to reason about.

For example, an in-memory implementation can later be replaced by DynamoDB without changing the domain meaning.

---

# 7. Matching a Retry Is Not Enough

An early implementation could recognize:

`MATCHING_OPERATION`

but that alone did not satisfy complete retry safety.

The system also needed to retrieve the previous authoritative result.

The desired behaviour became:

    first request
        ↓
    provider executes
        ↓
    ticket created
        ↓
    authoritative confirmation stored

    matching retry
        ↓
    stored confirmation retrieved
        ↓
    original result replayed
        ↓
    provider NOT called again

This is stronger than simply detecting duplicates.

---

# 8. Backend Truth Overrides Model Claims

A model saying:

> "Your ticket has been created."

does not prove a ticket exists.

Reliora introduced a typed authoritative result:

`TicketCreationConfirmation`

The confirmation includes information such as:

- logical `operation_id`;
- backend-generated `ticket_id`;
- provider identity;
- timezone-aware confirmation timestamp.

The principle is:

> Customer-visible success must depend on authoritative backend state.

The application must not fabricate:

- ticket identifiers;
- execution success;
- provider confirmation.

---

# 9. Ambiguous Side-Effect Outcomes Must Remain Ambiguous

One of the most important reliability cases is:

    operation registered
        ↓
    provider call begins
        ↓
    response is lost or outcome is uncertain
        ↓
    retry arrives

The system may know the operation exists but have no authoritative confirmation.

Reliora represents this as:

`UNCONFIRMED`

It does not automatically call the provider again.

Why?

Because the previous attempt may have:

1. failed before creating the ticket; or
2. created the ticket but lost the response.

Blindly retrying could create a duplicate.

The safe rule is:

> If the side-effect outcome cannot be established, do not guess.

Production systems may resolve this using:

- backend lookup;
- idempotent backend APIs;
- durable operation records;
- reconciliation workflows;
- human escalation.

---

# 10. Provider Contracts Decouple Business Logic From Infrastructure

Reliora defines a ticket-provider contract instead of placing AWS logic directly inside business orchestration.

Conceptually:

    TicketExecutionCoordinator
            ↓
       TicketProvider
        /         \
       /           \
    Fake        AWS Provider
    tests       implementation

The coordinator cares about the contract.

It does not need to know whether the implementation eventually uses:

- Lambda;
- DynamoDB;
- an external ticket system;
- another backend.

The reusable lesson is:

> Depend on stable domain contracts at infrastructure boundaries.

This improves:

- testability;
- portability;
- maintainability;
- cloud migration;
- failure simulation.

---

# 11. Safe Ticket Execution State Model

The execution coordinator introduced explicit outcomes.

| Status | Meaning |
|---|---|
| `CREATED` | This request executed the provider and received authoritative confirmation |
| `REPLAYED` | An existing authoritative result was returned without another provider call |
| `CONFLICT` | The same operation identity was reused with materially different payload data |
| `UNCONFIRMED` | The operation is known but authoritative success cannot be established |

This is intentionally more precise than a generic boolean such as:

`success = true`

Explicit state models improve:

- observability;
- debugging;
- failure handling;
- testing;
- auditability.

---

# 12. Exactly-Once Behaviour Is an End-to-End Property

A particularly important test demonstrated:

    first logical operation
        ↓
    CREATED
        ↓
    provider call count = 1

    matching retry
        ↓
    REPLAYED
        ↓
    provider call count = 1

The provider was not invoked twice.

This illustrates that exactly-once-like behaviour cannot be obtained merely by naming an idempotency key.

It requires cooperation between:

- operation identity;
- deterministic payload comparison;
- persistent operation state;
- result persistence;
- backend semantics;
- application control flow.

In distributed systems, exact guarantees depend on the complete architecture.

---

# 13. Deterministic Gates Protect Different Failure Classes

The runtime work repeatedly demonstrated that different quality tools detect different problems.

## pytest

Protects expected runtime behaviour.

Examples:

- retry replay;
- conflict handling;
- authorization;
- immutable domain contracts;
- missing-field collection.

## mypy

Protects static type relationships.

It found issues such as ambiguous inferred dictionary value types during test construction.

## Ruff

Protects formatting and code-quality conventions.

It caught:

- import ordering;
- datetime practices;
- formatting issues.

## Git staged checks

`git diff --cached --check`

detected problems such as trailing whitespace that unit tests and type checking did not detect.

The broader lesson is:

> Passing one engineering gate does not imply another class of correctness.

---

# 14. Negative Tests Matter

Several important behaviours were verified by proving that the system refuses unsafe behaviour.

Examples:

- incomplete bug reports cannot execute;
- arbitrary `authorized = true` input is rejected;
- conflicting operation payloads are blocked;
- an existing ticket result cannot silently be replaced;
- a provider confirmation for the wrong operation is rejected;
- an unconfirmed operation does not become guessed success;
- unexpected model fields are rejected.

Production reliability is often demonstrated more convincingly by what a system refuses to do than only by its happy path.

---

# 15. Local Reference Implementations Before Cloud Complexity

Reliora intentionally implemented several controls locally first:

- idempotency registry;
- authoritative result store;
- ticket-provider protocol;
- ticket execution coordinator;
- application authorization boundary.

The local implementations are not claimed to be distributed production persistence.

For example:

`InMemoryIdempotencyRegistry`

cannot provide:

- persistence across process restarts;
- cross-instance coordination;
- distributed atomicity.

Its purpose is to establish and test the domain semantics.

AWS persistence can then implement those same semantics using durable infrastructure.

The lesson is:

> Prove the invariant before introducing infrastructure complexity.

---

# 16. Evidence Discipline

A system component existing in source code is not sufficient evidence that a complete requirement has been satisfied.

Reliora distinguishes between states such as:

- `NOT STARTED`;
- `PARTIAL`;
- `IMPLEMENTED`;
- `VERIFIED`.

It also distinguishes evidence labels such as:

- `TARGET`;
- `OBSERVED`;
- `MEASURED`;
- `EXPERIMENTAL`;
- `SYNTHETIC ASSUMPTION`.

For example:

recognizing a matching retry was only partial implementation of idempotent retry behaviour.

The requirement became materially stronger only after the previous authoritative result could be replayed without re-executing the provider.

---

# 17. Git as Engineering Provenance

The runtime foundation was built as explicit capability slices.

Representative sequence:

| Capability | Reliora commit |
|---|---|
| Runtime domain contracts | `6338cbc` |
| Bug collection state | `700b4db` |
| Logical operation identity | `912b932` |
| Authorization preconditions | `556b73c` |
| Idempotency admission | `4b89a89` |
| Ticket confirmation contract | `efbf257` |
| Ticket result storage | `ac54473` |
| Safe execution and replay | `d972211` |
| Authorization-to-execution integration | `cc02e58` |

This history communicates how the architecture evolved rather than presenting one unexplained final code dump.

---

# 18. Local Runtime Completion Checkpoint

Before the AWS infrastructure phase:

- Reliora `main` was fast-forwarded to `cc02e58`;
- the working tree was clean;
- the full regression suite reported **164 passed**;
- AWS development moved to `feat/aws-infrastructure`.

The architecture transition is therefore:

    LOCAL DOMAIN AND SAFETY SEMANTICS
                ↓
         proven with tests
                ↓
             AWS
                ↓
    durable persistence + managed execution

This gives the cloud implementation a defined behavioural contract to preserve.

---

# 19. Interview Explanation

A concise explanation:

> I separated request identity, session identity, and logical operation identity because retries should not automatically represent new business actions. The LLM can propose ticket creation, but deterministic validation authorizes it before idempotency or execution. Once an eligible operation is registered, a matching retry replays the authoritative backend result rather than calling the provider again. A changed payload becomes an idempotency conflict, while an operation with no confirmed backend result remains unconfirmed instead of being guessed successful. I implemented these semantics locally first behind provider contracts, then used the cloud layer to make the persistence and execution durable.

A shorter version:

> Reliora treats LLM output as a proposal, not authorization. Side effects use stable operation IDs, deterministic idempotency controls, and backend-confirmed results so retries do not blindly create duplicate tickets.

---

# 20. Reusable Architecture Pattern

The runtime safety pattern can be generalized beyond support tickets.

    untrusted request
          ↓
    typed domain input
          ↓
    model recommendation
          ↓
    deterministic authorization
          ↓
    logical operation identity
          ↓
    idempotency admission
          ↓
    provider execution
          ↓
    authoritative confirmation
          ↓
    durable result
          ↓
    replay on safe retry

Potential use cases include:

- refunds;
- order cancellation;
- account changes;
- cloud provisioning;
- workflow approvals;
- database mutations;
- notifications;
- financial actions.

The risk controls required should increase with the risk of the side effect.

---

## Key Takeaway

The central lesson from the Reliora runtime phase is:

> Reliable agentic systems require deterministic control around probabilistic intelligence.

The LLM can reason and propose.

The application must still control:

- validation;
- authorization;
- operation identity;
- idempotency;
- side-effect execution;
- authoritative confirmation;
- failure handling;
- evidence.
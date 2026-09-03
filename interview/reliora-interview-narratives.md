# Reliora Interview Narrative Register

## Purpose

This document preserves the engineering narratives developed while implementing Reliora.

The purpose is not to manufacture polished interview answers after the project is finished. The purpose is to record the actual decisions, trade-offs, failures, evidence, and reasoning while they occur so that future interview explanations remain grounded in the implemented system.

Reliora interview narratives should distinguish clearly between:

- implemented and evidence-backed behaviour;
- partially implemented architectural decisions;
- planned work that has not yet been proven;
- measured results and unverified expectations.

No narrative should claim production scale, commercial adoption, SLA performance, cost savings, compliance, reliability, or business impact that has not been demonstrated.

---

# Status Model

| Status | Meaning |
|---|---|
| `EVIDENCE-BACKED` | The implementation or decision has direct code, test, benchmark, Git, AWS, or other project evidence. |
| `PARTIAL` | The architectural reasoning is established, but implementation or live evidence is incomplete. |
| `PLANNED` | The narrative represents intended work and must not yet be presented as completed experience. |

---

# Narrative Register

| ID | Narrative | Status | Core Interview Story |
|---|---|---|---|
| `INT-001` | From coursework to product engineering | `EVIDENCE-BACKED` | Reliora was deliberately reframed from a Udacity chatbot assignment into a production-oriented AI customer-support reliability and control reference implementation. |
| `INT-002` | AWS is the implementation platform, not the product | `EVIDENCE-BACKED` | Reliora's identity comes from the customer-support reliability problem and its control architecture rather than from merely combining AWS services. |
| `INT-003` | Build less, prove more | `EVIDENCE-BACKED` | Scope was intentionally constrained so that a narrow support workflow could be proved end to end rather than adding technologies without requirements. |
| `INT-004` | Requirements before technology | `EVIDENCE-BACKED` | New infrastructure or AI capabilities enter the architecture only when justified by requirements, evidence, security, reliability, operational constraints, or measurable business needs. |
| `INT-005` | The LLM may propose; deterministic software authorizes | `EVIDENCE-BACKED` | Probabilistic model output is treated as a proposal. Deterministic application logic decides whether a side effect may occur. |
| `INT-006` | Authorization before side effects | `EVIDENCE-BACKED` | Bug-report completeness and authorization are evaluated before idempotency registration or ticket creation so rejected requests cannot mutate backend state. |
| `INT-007` | Request, session, and operation identities solve different problems | `EVIDENCE-BACKED` | `request_id`, `session_id`, and `operation_id` have deliberately different lifecycles. The logical operation identity controls side-effect idempotency. |
| `INT-008` | Matching retry versus conflicting retry | `EVIDENCE-BACKED` | Same operation plus the same normalized payload is a replay; the same operation with changed payload is a conflict; distinct operation IDs remain independent. |
| `INT-009` | Backend truth overrides model claims | `EVIDENCE-BACKED` | Reliora does not treat an LLM statement as proof that a ticket was created. Customer-visible success requires authoritative backend confirmation. |
| `INT-010` | Ambiguous outcomes remain explicit | `EVIDENCE-BACKED` | When authoritative result state cannot be established, Reliora returns `UNCONFIRMED` rather than fabricating either success or failure. |
| `INT-011` | Separate idempotency admission from authoritative result state | `EVIDENCE-BACKED` | Determining whether a logical operation is new, matching, or conflicting is distinct from retaining the authoritative business result associated with that operation. |
| `INT-012` | Provider boundaries preserve backend flexibility | `EVIDENCE-BACKED` | Ticket creation sits behind a provider contract so application logic is not directly coupled to a particular ticket backend. |
| `INT-013` | Evidence-driven routing selection | `EVIDENCE-BACKED` | Routing alternatives were benchmarked instead of selecting an LLM or classifier because it appeared more sophisticated. |
| `INT-014` | Frozen data before model selection | `EVIDENCE-BACKED` | Routing benchmark data and splits were frozen before final evaluation so model selection could be separated from held-out evidence. |
| `INT-015` | Held-out means held out | `EVIDENCE-BACKED` | Once the final held-out routing test was executed, its result became evidence rather than another tuning signal. Future improvements require new independent test data. |
| `INT-016` | A disappointing result can be valuable engineering evidence | `EVIDENCE-BACKED` | The selected router performed strongly on validation but dropped materially on the held-out test. The generalization gap was preserved rather than hidden through post-test tuning. |
| `INT-017` | Deterministic evaluation and LLM-as-judge answer different questions | `EVIDENCE-BACKED` | Behavioural invariants and regression tests protect deterministic requirements, while LLM-based evaluation can assess qualities that deterministic assertions cannot fully capture. |
| `INT-018` | RAG is not an automatic architectural upgrade | `EVIDENCE-BACKED` | For a small, stable, approved support knowledge source, a versioned source can be more proportional than introducing mandatory retrieval infrastructure. |
| `INT-019` | Human handoff is an engineered outcome | `PARTIAL` | Unsupported or higher-risk cases should transition through structured human handoff rather than relying on generic fallback language. |
| `INT-020` | Observability means reconstructability | `PARTIAL` | Useful observability should allow an operator to reconstruct one support operation across identity, authorization, execution, backend outcome, timing, and failure state. |
| `INT-021` | Cost claims require measurements | `PARTIAL` | Reliora establishes cost guardrails first and defers application-level cost and efficiency claims until token usage, invocations, latency, and unit economics are measured. |
| `INT-022` | Terraform is about reproducibility, not résumé decoration | `EVIDENCE-BACKED` | Terraform was selected to make AWS infrastructure reviewable, repeatable, auditable, and destroyable rather than merely to demonstrate Terraform syntax knowledge. |
| `INT-023` | Local correctness is not distributed correctness | `EVIDENCE-BACKED` | The local `check -> execute -> save` implementation established correct semantics but cannot be assumed atomic across processes, crashes, or concurrent Lambda invocations. |
| `INT-024` | Local reference semantics to distributed AWS correctness | `EVIDENCE-BACKED` | Moving to AWS was deliberately not a mechanical replacement of dictionaries with DynamoDB. The persistence design was changed to preserve the same behaviour under distributed failure and concurrency. |
| `INT-025` | Dependency inversion before cloud integration | `EVIDENCE-BACKED` | `TicketExecutionPort` allows the application layer to depend on execution behaviour while local and AWS implementations provide different mechanics behind the same contract. |
| `INT-026` | Behaviour-preserving architectural refactoring | `EVIDENCE-BACKED` | A 23-test reliability baseline was captured before introducing the execution port, and the same 23 tests passed after the refactor. |
| `INT-027` | Tests as executable architecture | `EVIDENCE-BACKED` | Tests explicitly encode authorization, replay, conflict, authoritative confirmation, fingerprinting, and ambiguous-state semantics that infrastructure implementations must preserve. |
| `INT-028` | Explicit cloud dependency management | `EVIDENCE-BACKED` | Boto3 was verified as absent, introduced explicitly through `uv`, locked reproducibly, and followed by regression testing rather than relying on an implicit machine dependency. |
| `INT-029` | Terraform versus Boto3 | `EVIDENCE-BACKED` | Terraform defines and provisions infrastructure; Boto3 is the runtime Python SDK through which application code interacts with AWS APIs. |
| `INT-030` | Boto3 versus Botocore | `EVIDENCE-BACKED` | Boto3 provides the developer-facing AWS Python SDK while Botocore provides much of the lower-level service metadata, signing, endpoint, retry, request, and response machinery. |
| `INT-031` | Distributed idempotency with DynamoDB | `EVIDENCE-BACKED` | The AWS adapter uses conditional transactional writes so operation-control state and ticket state are created atomically and only one concurrent invocation can claim a logical operation. |
| `INT-032` | Concurrency races require reconciliation | `EVIDENCE-BACKED` | Two workers can initially observe that an operation is absent. The DynamoDB transaction determines the winner, and the losing worker reconciles against authoritative state to return replay or conflict. |
| `INT-033` | Strong consistency where correctness requires it | `EVIDENCE-BACKED` | Idempotency reconciliation uses strongly consistent reads because immediate correctness of logical-operation state is more important at that boundary than minimizing a small read-cost difference. |
| `INT-034` | Design cloud adapters for deterministic testing | `EVIDENCE-BACKED` | Ticket ID generation and the clock are injectable so AWS adapter behaviour can be tested deterministically without random IDs or wall-clock-dependent assertions. |
| `INT-035` | Fake cloud clients before live-cloud validation | `EVIDENCE-BACKED` | Seven deterministic DynamoDB adapter tests verify atomic transaction construction, replay, conflict, race reconciliation, and ambiguous-state handling without live AWS dependencies. |
| `INT-036` | Full regression gate after infrastructure changes | `EVIDENCE-BACKED` | After introducing the DynamoDB execution adapter, the complete Reliora suite passed `171/171` tests. |
| `INT-037` | Distinguish collection failure from test failure | `EVIDENCE-BACKED` | A full pytest command initially failed during module collection. The failure was classified as an import-path/tooling problem rather than incorrectly treating it as an application regression. |
| `INT-038` | Troubleshoot before modifying working application code | `EVIDENCE-BACKED` | Direct Python import proved the repository-local `experiments` namespace was valid, and `uv run python -m pytest` then executed all 171 tests successfully. |
| `INT-039` | Transport identity is not business-operation identity | `PARTIAL` | AWS or AgentCore request identifiers should not automatically become Reliora's idempotency key because a transport attempt and a logical business operation have different lifecycles. |
| `INT-040` | Production-oriented, not production-proven | `EVIDENCE-BACKED` | Reliora is described as a production-oriented reference implementation and limits claims to evidence actually produced in the implemented environment. |
| `INT-041` | Failure boundaries are architecture | `PARTIAL` | Validation, authorization, Gateway, Lambda, DynamoDB, ticket-provider, and response-path failures require explicit behaviour rather than a generic AI failure state. |
| `INT-042` | CI/CD should enforce evidence, not only deployment | `PLANNED` | The intended release path will use deterministic regression and evaluation gates before cloud deployment rather than treating successful deployment as sufficient release evidence. |
| `INT-043` | Portability comes from boundaries, not fake cloud neutrality | `EVIDENCE-BACKED` | Reliora is intentionally AWS-first, while application contracts remain outside AWS adapters so future provider or infrastructure substitutions occur through defined boundaries rather than rewrites. |
| `INT-044` | Security is operational architecture | `PARTIAL` | Temporary AWS authentication, MFA, absence of long-lived access keys, least-privilege runtime roles, and future GitHub OIDC form one security architecture rather than isolated configuration tasks. |
| `INT-045` | FinOps begins before deployment | `EVIDENCE-BACKED` | AWS budget guardrails were established before meaningful cloud deployment, while performance and unit-cost claims remain deferred until measured. |

---

# Current High-Value Evidence

## Routing Evaluation

The routing work provides evidence for:

- `INT-013`
- `INT-014`
- `INT-015`
- `INT-016`
- `INT-017`

The frozen routing benchmark and held-out evaluation demonstrate evidence-driven model selection and disciplined separation between model development and final evaluation.

## Runtime Safety Foundation

The local runtime implementation and associated tests provide evidence for:

- `INT-005`
- `INT-006`
- `INT-007`
- `INT-008`
- `INT-009`
- `INT-010`
- `INT-011`
- `INT-012`
- `INT-023`
- `INT-027`

The local implementation intentionally acts as a reference semantics layer before distributed AWS infrastructure is introduced.

## AWS Durable Execution Foundation

Reliora commit:

`24d544a — feat: establish durable AWS ticket execution foundation`

provides evidence for:

- `INT-024`
- `INT-025`
- `INT-026`
- `INT-028`
- `INT-029`
- `INT-031`
- `INT-032`
- `INT-033`
- `INT-034`
- `INT-035`
- `INT-036`

Key evidence includes:

- explicit `TicketExecutionPort`;
- local coordinator retained as a reference implementation;
- DynamoDB-backed durable execution adapter;
- atomic `TransactWriteItems` design;
- conditional operation and ticket creation;
- strongly consistent reconciliation reads;
- deterministic fake DynamoDB testing;
- seven AWS-adapter reliability tests;
- complete `171/171` Reliora regression suite.

## Troubleshooting Evidence

The pytest collection incident provides evidence for:

- `INT-037`
- `INT-038`

Observed sequence:

1. `uv run pytest -q` failed during collection with `ModuleNotFoundError: No module named 'experiments'`.
2. Direct Python import of `experiments` succeeded as a namespace package.
3. No unnecessary package-structure changes were introduced.
4. `uv run python -m pytest -q` successfully executed all 171 tests.

The engineering lesson is to classify the failure boundary correctly before modifying application code.

---

# Narratives Awaiting Evidence

The following should not yet be presented as completed implementation stories.

## AgentCore and Gateway

Future evidence should support:

- stable logical `operation_id` propagation;
- AgentCore to Gateway to Lambda invocation;
- deterministic authorization before ticket mutation;
- successful authoritative DynamoDB confirmation.

Related narratives:

- `INT-039`
- `INT-041`

## Live AWS Idempotency

Required evidence:

- first operation creates exactly one ticket;
- matching retry returns the same ticket;
- no duplicate ticket is created;
- conflicting retry is blocked;
- ambiguous dependency behaviour does not fabricate success.

This will strengthen:

- `INT-031`
- `INT-032`
- `INT-041`

## Observability

Required evidence:

- correlated request, session, and operation identity;
- Lambda execution evidence;
- DynamoDB outcome;
- authorization decision;
- latency and failure reconstruction.

This will move `INT-020` toward `EVIDENCE-BACKED`.

## CI/CD and Deployment Security

Required evidence:

- GitHub Actions workflow;
- AWS OIDC authentication;
- least-privilege deployment role;
- regression/evaluation gate;
- unsafe change blocked before deployment.

This will support:

- `INT-042`
- `INT-044`

## Cost and Performance

Required evidence:

- measured invocation counts;
- model token usage where applicable;
- p50/p95 latency;
- estimated or measured AWS cost;
- cost per successful operation;
- clear measurement assumptions.

This will move `INT-021` from `PARTIAL` toward `EVIDENCE-BACKED`.

---

# Interview Answer Development Protocol

Not every register entry needs a memorized answer.

The strongest narratives should eventually receive three forms:

### Short answer

Approximately 20–30 seconds.

Purpose:

- recruiter screen;
- quick technical clarification;
- behavioural interview follow-up.

### Engineering answer

Approximately 60–120 seconds.

Structure:

1. problem;
2. constraint;
3. decision;
4. implementation;
5. evidence;
6. trade-off.

### System-design answer

Approximately 2–5 minutes.

Structure:

1. context;
2. requirement;
3. failure model;
4. alternatives considered;
5. selected architecture;
6. implementation boundary;
7. verification;
8. limitations;
9. scaling or future evolution.

---

# Priority Narratives for Deep Interview Preparation

The first narratives that should eventually receive full interview answers are:

1. `INT-005` — The LLM may propose; deterministic software authorizes.
2. `INT-007` — Request, session, and operation identity.
3. `INT-008` — Matching retry versus conflicting retry.
4. `INT-009` — Backend truth overrides model claims.
5. `INT-015` — Held-out means held out.
6. `INT-016` — Preserving a disappointing generalization result.
7. `INT-023` — Local correctness is not distributed correctness.
8. `INT-024` — Local semantics to distributed AWS correctness.
9. `INT-025` — Dependency inversion before cloud integration.
10. `INT-026` — Behaviour-preserving refactoring.
11. `INT-031` — Distributed idempotency with DynamoDB.
12. `INT-032` — Concurrency-race reconciliation.
13. `INT-036` — Full regression gate.
14. `INT-037` — Collection failure versus test failure.
15. `INT-039` — Transport identity versus business-operation identity.

---

# Update Rule

When a new interview-worthy event occurs:

1. assign the next `INT-xxx` identifier;
2. record the engineering problem rather than only the technology used;
3. identify whether the narrative is `EVIDENCE-BACKED`, `PARTIAL`, or `PLANNED`;
4. link the narrative to code, tests, ADRs, Git commits, cloud evidence, or measurements where available;
5. never upgrade a narrative to `EVIDENCE-BACKED` solely because the architecture is intended or documented;
6. preserve failures and trade-offs rather than rewriting the project history into a perfect implementation story.

The register should evolve with the project.
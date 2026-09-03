# Portfolio Product Engineering Standard

**Document type:** Cross-project engineering standard
**Applies to:** Reliora and all subsequent portfolio products
**Status:** Governing baseline
**Version:** 1.1
**Established:** 2026-09-03

---

# 1. Purpose

This document defines the common engineering standard inherited by every product in the portfolio.

Each product must still have its own:

- product name and identity;
- business problem;
- target users and buyers;
- North Star;
- architecture;
- requirements;
- product-specific risks;
- success metrics;
- product constitution.

This standard exists to ensure the products share a recognizable level of engineering maturity without forcing them into the same architecture.

The goal is that the portfolio reads like a small professional product studio rather than a collection of unrelated course assignments.

---

# 2. Governing Principle

> **The product chooses the technology. The technology does not define the product.**

AWS, Docker, Terraform, GitHub Actions, model providers, vector stores, frameworks, and observability tools are implementation choices.

They must earn their place by solving a documented product, reliability, operational, security, cost, or delivery problem.

## 2.1 Context Isolation and Source Precedence

This document defines **shared engineering quality standards only**.

It must never be used by itself to infer a product-specific:

- capability;
- AWS service;
- model;
- agent topology;
- RAG requirement;
- memory requirement;
- browser or code-execution requirement;
- workflow;
- business rule;
- product scope;
- acceptance criterion;
- architecture decision.

For any individual portfolio product, use this precedence order:

1. official project requirements and rubric for that product;
2. that product's constitution;
3. that product's North Star;
4. that product's requirements, ADRs, policies, architecture, evaluation contracts, and preserved evidence;
5. this Portfolio Product Engineering Standard;
6. general engineering knowledge and external documentation.

The product-specific source always wins when it is more specific.

The governing rule is:

> **This standard defines how well every product should be engineered. It does not define what every product must contain.**

### Cross-project contamination is prohibited

A capability used by one product must not be copied into another merely because:

- it is available in AWS;
- it is popular in industry;
- it appears elsewhere in the portfolio;
- it is attractive on a rÃ©sumÃ©;
- another product's rubric requires it.

Examples:

- Reliora's deterministic authorization and idempotent ticket semantics do not automatically become requirements for every other product.
- A later project's RAG architecture does not make RAG mandatory for Reliora.
- A multi-agent project does not make multi-agent orchestration a shared portfolio requirement.
- Browser, persistent memory, Code Interpreter, Knowledge Bases, vector stores, or distributed tracing are product-specific unless that project's own requirements justify them.
- A model required by one project's rubric must not be assumed to be the preferred model for another project.

Cross-project information may be used for:

- comparison;
- lessons learned;
- reusable engineering patterns;
- cost or operational trade-off analysis;
- future architecture options.

When used this way, it must remain clearly labelled as comparative or reusable context rather than product-specific fact.

---

# 3. Product Standard

Every portfolio project must be treated as a commercially credible product reference implementation.

A product should be understandable in terms of:

1. the business problem;
2. the user;
3. the buyer;
4. the workflow;
5. the measurable value;
6. the architecture;
7. the failure model;
8. the operating model;
9. the deployment model;
10. the evidence supporting its claims.

The portfolio should not present projects primarily as:

- "I used service X";
- "I completed a course project";
- "Here is my notebook";
- "Here is my GitHub repository."

The portfolio should present them as:

> **Products designed to solve real business problems, implemented with production-oriented engineering practices, and supported by evidence.**

---

# 4. Production-Oriented, Not Fictionally Production-Proven

The preferred wording across the portfolio is:

> **Production-oriented reference implementation.**

A project may demonstrate:

- reproducible deployment;
- automated testing;
- typed contracts;
- failure handling;
- observability;
- security controls;
- cost measurement;
- rollback;
- evaluation;
- realistic product UX.

That does not automatically prove:

- millions of production users;
- enterprise SLA performance;
- compliance certification;
- penetration-test completion;
- multi-region resilience;
- commercial customer adoption.

Claims must never exceed evidence.

---

# 5. Shared Engineering Baseline

Unless a product-specific constitution justifies otherwise, the portfolio baseline is:

## Backend

- Python 3.12 where appropriate;
- `uv` for dependency/environment management;
- Pydantic for typed contracts where Python boundaries require runtime validation;
- pytest for automated tests;
- mypy for static typing;
- Ruff for linting and formatting.

## Infrastructure

- Terraform for persistent cloud infrastructure;
- reproducible deploy/verify/destroy workflows;
- explicit environment configuration;
- no undocumented click-only production architecture.

## Containers

- Docker where it improves reproducibility, local development, packaging, or deployment;
- Docker Compose where multiple local services benefit from one-command startup;
- no container orchestration platform added merely for rÃ©sumÃ© keywords.

## CI/CD

- GitHub Actions;
- automated linting;
- automated typing;
- automated tests;
- evaluation/regression gates where appropriate;
- Terraform formatting/validation;
- security checks;
- short-lived cloud authentication such as OIDC where supported.

## Security

Preferred baseline:

- least-privilege IAM;
- no secrets committed to Git;
- secret scanning such as Gitleaks;
- IaC scanning such as Checkov;
- input/schema validation;
- explicit authorization boundaries for side effects;
- prompt-injection/adversarial tests where LLMs are involved;
- controlled logging and redaction.

## Observability

- structured logging;
- correlation identifiers;
- meaningful metrics;
- tracing where distributed workflows justify it;
- cloud-native observability first unless another tool solves a documented need.

## Frontend

For products requiring a product UI, preferred baseline:

- Next.js;
- React;
- TypeScript;
- a maintainable modern styling/design-system approach such as Tailwind CSS;
- responsive design;
- loading/error/empty states;
- accessibility-conscious interactions.

## End-to-end testing

Preferred baseline when a user-facing workflow exists:

- Playwright.

These tools are defaults, not mandatory decorations.

---

# 6. Tool Admission Rule

Before adding a technology, answer:

1. What requirement does it satisfy?
2. What problem does it solve?
3. What risk does it reduce?
4. What measurable limitation justifies it?
5. What operational burden does it add?
6. Is there a simpler existing option?
7. Does it materially improve the product?
8. Would a real engineering team plausibly adopt it here?

If the answers are weak, the tool remains out of scope.

---

# 7. AWS-First, Not AWS-Defined

The current portfolio is intentionally AWS-heavy because of the certification and training roadmap.

That is an advantage.

It should demonstrate deep AWS competence without making the products dependent on AWS branding for their identity.

Each product should be explainable in two layers:

## Product layer

What business capability is being sold or delivered?

## Infrastructure layer

How is that capability currently implemented on AWS?

Future versions may selectively replace implementation components with:

- Azure;
- Google Cloud;
- other model providers;
- external SaaS platforms.

Multi-cloud should be demonstrated through meaningful substitutions, not speculative wrappers around every cloud call.

---

# 8. Product-Specific Architecture Wins Over Uniformity

The three products should not be forced into identical technology stacks.

Shared engineering practice is desirable.

Shared architecture is not.

Examples:

- one product may justify a small embedded knowledge source;
- another may require RAG;
- another may require multiple agents;
- one may need browser automation;
- another may specifically exclude it.

The standard governs **quality**, not architecture sameness.

---

# 9. Interface and Adapter Principle

Business logic should be isolated from vendor-specific implementations at boundaries where replacement is plausible and useful.

Good adapter candidates include:

- model providers;
- ticketing systems;
- knowledge providers;
- telemetry backends;
- payment/order systems;
- external APIs;
- vector stores where substitution is realistic.

Do not abstract stable internals merely to claim portability.

---

# 10. Product Experience Standard

Every product that needs a UI should look like something a real company could plausibly pilot.

The UI must make the business problem visible.

It should not resemble:

- a raw notebook;
- an unstyled class demo;
- a cloud-console walkthrough;
- an empty generic chatbot page.

A professional UI should communicate:

- brand identity;
- product purpose;
- realistic workflows;
- meaningful states;
- errors and recovery;
- business context;
- operational credibility.

The frontend should consume product interfaces rather than contain cloud-specific business logic.

---

# 11. Local, Cloud, and Demo Modes

Products should be designed so the portfolio remains usable even when paid cloud infrastructure is offline.

Where appropriate, support:

## Local development mode

For:

- development;
- tests;
- UI work;
- zero-cloud-cost demonstrations.

## Cloud integration mode

For:

- real infrastructure validation;
- cloud evidence;
- performance/cost measurement;
- submission requirements.

## Ephemeral portfolio mode

Preferred pattern:

```text
deploy
â†’ verify
â†’ demonstrate
â†’ capture evidence
â†’ destroy
```

A permanent hosted demo is optional and requires explicit cost and abuse-control justification.

---

# 12. Testing Standard

Testing should match system risk.

The baseline layers are:

## Unit tests

Verify deterministic domain rules and contracts.

## Integration/service tests

Verify interactions between meaningful components.

## Cloud integration tests

Verify real provider and infrastructure boundaries.

## End-to-end tests

Verify realistic product journeys.

## Failure tests

Verify what happens when things break.

Important failures should be tested deliberately, not discovered accidentally during demos.

---

# 13. AI Evaluation Standard

For AI systems, separate:

## Deterministic requirements

Examples:

- schema adherence;
- authorization;
- state transitions;
- forbidden actions;
- duplicate prevention;
- leakage;
- tool arguments;
- routing invariants.

These should use deterministic evaluators whenever practical.

## Generative quality

Examples:

- helpfulness;
- clarity;
- semantic completeness;
- tone;
- response quality.

These may use LLM-based evaluation.

A strong LLM-judge score must never override a failed deterministic safety invariant.

---

# 14. Data and Benchmark Discipline

Datasets used for evaluation or model selection should be:

- versioned;
- purpose-labelled;
- traceable;
- separated into appropriate train/validation/test roles;
- protected from post-test tuning;
- clearly marked when synthetic.

Held-out evidence should remain held out.

If a test set has influenced design decisions, a future evaluation should use new independent data.

---

# 15. Reliability Standard

Every product should identify its critical reliability risks.

Depending on the product, controls may include:

- retries;
- idempotency;
- optimistic locking;
- bounded execution;
- timeouts;
- circuit breakers;
- reconciliation;
- graceful degradation;
- human handoff;
- rollback;
- dead-letter processing.

Do not add all reliability patterns everywhere.

Use the patterns required by the product's real failure modes.

---

# 16. Security Standard

Every product should include a realistic baseline security model.

At minimum:

- least privilege;
- secrets outside source control;
- trusted/untrusted input boundaries;
- schema validation;
- authorization where actions change state;
- prompt-injection testing for agentic systems;
- dependency/IaC scanning;
- log minimisation;
- environment separation.

Security claims must remain proportional to implemented controls.

---

# 17. Privacy and Governance Standard

Use synthetic data by default unless real data is necessary and appropriately governed.

Document:

- what data is collected;
- why it is required;
- what is retained;
- what is logged;
- what is redacted;
- what is synthetic;
- what would require customer-specific governance in a real deployment.

Do not imply regulatory compliance without evidence.

---

# 18. Observability Standard

A technical operator should be able to reconstruct important workflows.

Useful observability may include:

- request/session/workflow identifiers;
- route/agent decisions;
- model version;
- prompt version;
- tool calls;
- backend outcomes;
- latency;
- retry/conflict information;
- failure categories;
- deployment version.

Do not log hidden chain-of-thought merely to claim observability.

---

# 19. Performance Standard

Do not set arbitrary production-scale targets and then claim they were achieved.

Preferred process:

1. establish baseline;
2. measure;
3. identify bottleneck;
4. define justified target;
5. optimize;
6. remeasure.

Potential metrics:

- p50 latency;
- p95 latency;
- throughput;
- concurrency;
- errors;
- model latency;
- retrieval latency;
- tool latency;
- cost under load.

Scale claims require scale evidence.

---

# 20. FinOps Standard

Cost is an architecture dimension.

Each product should understand:

- major cost drivers;
- idle cost;
- request-level cost where practical;
- model token cost;
- retrieval cost;
- tool/runtime cost;
- evaluation cost;
- teardown procedure.

Cloud experiments should have:

- budget alerts;
- explicit cost ceiling;
- short resource lifetime;
- teardown verification.

Cost optimization should not reduce reliability or security without an explicit trade-off.

---

# 21. CI/CD Standard

Target pipeline:

```text
commit
    â†“
format/lint
    â†“
type check
    â†“
tests
    â†“
deterministic evaluation
    â†“
security checks
    â†“
IaC validation
    â†“
plan
    â†“
approved deployment
    â†“
verification
```

Projects should progressively approach this model.

Not every product must implement every stage on day one.

---

# 22. Infrastructure-as-Code Standard

Persistent infrastructure should be reviewable and reproducible.

Terraform should support:

- environment parameters;
- naming conventions;
- tags;
- outputs;
- dependency relationships;
- teardown;
- validation.

Manual console work may be used for:

- learning;
- debugging;
- temporary exploration;
- unsupported setup steps.

It should not become undocumented production state.

---

# 23. Docker Standard

Docker should solve a real delivery problem.

Good reasons:

- consistent local runtime;
- dependency isolation;
- onboarding;
- reproducible demos;
- deployment packaging;
- CI parity.

Bad reason:

> "Docker is popular, so every repository needs a Dockerfile."

Prefer the smallest container architecture that solves the product need.

---

# 24. Documentation Standard

Each product should eventually contain:

## Product

- product overview;
- problem;
- target users;
- business value;
- supported workflows;
- roadmap.

## Architecture

- logical architecture;
- deployment architecture;
- data flow;
- trust/control boundaries;
- ADRs.

## Engineering

- local setup;
- commands;
- tests;
- dependency management;
- Docker;
- CI/CD.

## Operations

- deploy;
- verify;
- observe;
- troubleshoot;
- destroy;
- rollback.

## Governance

- security assumptions;
- privacy assumptions;
- evaluation;
- evidence classification.

## Portfolio

- case study;
- demo;
- architecture summary;
- measurable outcomes;
- known limitations.

---

# 25. Evidence Standard

Every major claim should point to evidence.

Examples:

| Claim | Suitable evidence |
|---|---|
| Retry-safe | deterministic test + real integration evidence |
| Observable | trace/log/request reconstruction |
| Cost-aware | measured workload + pricing/cost calculation |
| Scalable | measured load test within stated envelope |
| Secure | implemented controls + tests/scans |
| Grounded | evaluation + source provenance |
| Deployable | IaC deployment + verification + teardown |
| Reliable | failure tests + observed outcomes |

The repository should distinguish:

- target;
- observed;
- measured;
- experimental;
- synthetic assumption.

---

# 26. Git and Engineering Provenance

Git history should communicate architecture evolution.

Prefer coherent capability commits such as:

- establish domain contracts;
- implement idempotent execution;
- add cloud persistence;
- add tracing;
- add evaluation gate.

Avoid:

- giant unexplained final commits;
- hundreds of meaningless micro-commits;
- mixing unrelated architecture changes.

Branches should describe the engineering phase or capability.

---

# 27. Learning Ledger Standard

Important reusable engineering lessons should be extracted into the Learning Ledger.

The product repository records:

- what Reliora/P2/P3 did;
- implementation evidence;
- product-specific rationale.

The Learning Ledger records:

- reusable concepts;
- command lessons;
- architecture patterns;
- troubleshooting;
- interview explanations.

Avoid duplicating entire product documentation into the Ledger.

---

# 28. Command and Verification Discipline

For meaningful engineering actions, understand:

- what the command does;
- why it is being run now;
- important arguments/flags;
- expected output;
- meaningful risk or side effect.

After important changes, verify:

- correct repository;
- correct branch;
- intended files;
- tests;
- lint/type gates;
- Git state.

High-risk cloud/destructive commands require additional care.

---

# 29. Product Acceptance Standard

A product should have canonical scenarios that prove its core value.

For each scenario define:

- starting state;
- user action;
- expected system behaviour;
- expected business result;
- failure variant;
- observable evidence;
- acceptance criteria.

These scenarios should later drive:

- integration tests;
- UI tests;
- demos;
- portfolio videos.

---

# 30. Product Constitution Requirement

Each major portfolio product must have its own constitution.

The constitution should define:

- product identity;
- target buyer/user;
- business problem;
- North Star;
- non-goals;
- product-specific architecture;
- product-specific risks;
- success metrics;
- acceptance scenarios;
- roadmap;
- production-grade definition.

The cross-project standard does not replace product constitutions.

A product constitution is authoritative for that product's capabilities, exclusions, architecture, scope, acceptance criteria, and product-specific engineering decisions. This standard may supplement those decisions with shared quality expectations, but it may not override or broaden them by implication.

---

# 31. Portfolio Differentiation Rule

Products must solve distinct problems and demonstrate distinct architectural strengths.

They should not become three versions of the same chatbot.

The current intended differentiation is:

## Reliora

Primary story:

**AI reliability, deterministic control, safe side effects, evaluation, and operational trust.**

## Product 2

Primary story:

**Enterprise capability integration across tools, APIs, knowledge, memory, computation, and browser-like capabilities with governance.**

## Product 3

Primary story:

**Multi-agent orchestration, concurrent/shared state, parallel RAG, distributed tracing, guardrails, and workflow reliability.**

The shared engineering standard makes them feel related.

Their architectures make them worth comparing.

---

# 32. Commercial Adaptation Standard

Each product should answer:

> What would have to change for a company to adopt this?

Ideal answer:

```text
configure
â†’ connect adapters
â†’ deploy
â†’ verify
â†’ pilot
```

not:

```text
rewrite entire project
```

Each product should identify its real adaptation boundaries.

---

# 33. Product Demo Standard

A strong demo should show:

1. the business problem;
2. the user experience;
3. the important system behaviour;
4. the engineering control;
5. evidence;
6. the outcome.

Do not make AWS Console the main product demo.

AWS Console belongs in technical evidence.

The customer-facing product should look and behave like a product.

---

# 34. Recording Standard

Record high-value engineering evidence when the system demonstrates something difficult to reconstruct later.

Examples:

- cloud deployment;
- real side-effect execution;
- retry/idempotency behaviour;
- distributed traces;
- guardrail blocking;
- multi-agent orchestration;
- performance/cost measurement;
- failure recovery.

Keep raw masters.

Edit later into:

- recruiter clip;
- technical demo;
- deeper architecture walkthrough.

Do not allow video editing to delay cloud-critical engineering work.

---

# 35. Time-Critical Cloud Strategy

When cloud credits or service access have a deadline:

Prioritize:

1. cloud resources;
2. integration;
3. real measurements;
4. evidence;
5. teardown.

Postpone:

- frontend polish;
- diagrams;
- README polishing;
- video editing;
- portfolio-site layout;
- narration.

These can be completed after cloud teardown if the architecture is properly decoupled.

---

# 36. Product ROI Standard

Additional engineering work should be prioritized by portfolio and product value.

High ROI work typically includes:

- meaningful reliability control;
- production-like failure handling;
- observability;
- IaC;
- CI/CD;
- measurable evaluation;
- cost evidence;
- realistic product UX;
- strong architecture trade-off.

Lower ROI work often includes:

- adding another framework for no requirement;
- duplicate dashboards;
- unnecessary microservices;
- speculative multi-cloud abstraction;
- excessive visual polish before the system works.

---

# 37. Definition of Done â€” Cross-Project

A product is portfolio-ready when:

## Business

- real problem is clear;
- buyer/user is clear;
- value proposition is clear.

## Product

- coherent product identity;
- realistic workflow;
- professional user experience where required.

## Engineering

- typed or otherwise explicit contracts;
- meaningful tests;
- failure handling;
- reproducible environments;
- IaC;
- CI/CD baseline;
- security baseline.

## AI

- model choice justified;
- quality evaluated;
- hard invariants separated from subjective evaluation;
- grounding/retrieval governed where applicable.

## Operations

- observable;
- diagnosable;
- cost-aware;
- teardown/redeployment documented.

## Evidence

- important claims supported;
- synthetic claims clearly identified;
- limitations disclosed.

## Portfolio

- concise case study;
- architecture diagram;
- demo;
- measurable evidence;
- interview-ready trade-offs.

---

# 38. Change Control

Update this standard when the common engineering philosophy changes across products.

Do not update it merely because one product makes a unique architecture choice.

Product-specific decisions belong in that product's constitution or ADRs.

Never change this shared standard merely to encode a capability that belongs to only one product.

---

# 39. Final Standard

> **Every portfolio project should look like a product a company could plausibly pilot, inspect like software an engineering team could plausibly maintain, and explain like a system an architect could defend.**

> **Use mainstream tools where they solve real problems. Use AWS deeply without allowing AWS service names to become the product. Build professional interfaces, reproducible infrastructure, measurable controls, and evidence-backed claims.**

> **The portfolio should demonstrate product judgment, architecture judgment, engineering discipline, and AI/ML judgmentâ€”not simply tool familiarity.**

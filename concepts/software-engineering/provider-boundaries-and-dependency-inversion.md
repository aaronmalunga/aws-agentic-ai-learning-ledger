# Provider Boundaries and Dependency Inversion

## Why This Lesson Exists

Reliora will eventually interact with several external capabilities.

Examples include:

```text
LLM/model routing
knowledge retrieval
ticket creation
human handoff
telemetry
AWS services
```

A simple implementation could directly call AWS SDKs or model APIs from everywhere in the application.

For example:

```python
def handle_customer_message(message):
    # call Bedrock
    # inspect response
    # query knowledge
    # invoke Lambda
    # write telemetry
    # construct customer response
```

This may work in a small prototype.

However, it creates strong coupling between:

```text
business logic
```

and:

```text
specific infrastructure implementations
```

Reliora instead moves toward explicit provider boundaries such as:

```text
Router
KnowledgeProvider
TicketProvider
HandoffProvider
TelemetryProvider
```

The application depends on what these capabilities **do**, while the infrastructure layer determines **how** they are implemented.

This is closely related to the software-engineering principle called:

```text
dependency inversion
```

---

# 1. The Core Problem: Tight Coupling

Suppose application logic contains code like:

```python
bedrock_client.invoke_model(...)
```

throughout multiple modules.

Now the application implicitly depends directly on:

```text
Amazon Bedrock API details
model request format
AWS authentication
response structure
provider errors
```

Changing the model implementation may require changing business logic in many places.

---

# 2. Another Tight-Coupling Example

Suppose ticket creation directly contains:

```python
lambda_client.invoke(...)
```

inside the conversation workflow.

Now the workflow understands:

```text
Lambda
AWS SDK
function names
AWS error types
serialization format
```

even though its actual business concern is simply:

> Create one validated bug ticket.

That mixes responsibilities.

---

# 3. What Is Coupling?

Coupling describes how strongly one part of software depends on another.

High coupling means:

```text
change component A
→ many changes required in component B
```

Lower coupling means components communicate through stable boundaries.

---

# 4. Tight Coupling Is Not Always Bad

Some dependencies are natural.

For example:

```text
BugWorkflow
```

must understand:

```text
bug-report state
```

because that is its responsibility.

The problem is unnecessary coupling.

For example:

```text
BugWorkflow
```

should not need to know:

```text
how boto3 signs an AWS request
```

if that detail can be isolated.

---

# 5. The Provider Boundary

A provider boundary defines a capability.

For example:

```text
TicketProvider
```

can mean:

> A component capable of creating and retrieving support-ticket operations through a defined contract.

The application does not need to know whether the implementation uses:

```text
Lambda
AgentCore Gateway
direct HTTP
local fake backend
```

as long as the contract is satisfied.

---

# 6. Mental Model

Instead of:

```text
Application
    ↓
AWS-specific implementation everywhere
```

use:

```text
Application
    ↓
Provider interface
    ↓
Infrastructure implementation
```

For example:

```text
BugWorkflow
     ↓
TicketProvider
     ↓
AWS TicketProvider implementation
     ↓
AgentCore Gateway / Lambda / DynamoDB
```

---

# 7. What Is Dependency Inversion?

A traditional dependency might look like:

```text
High-level business logic
        ↓
Low-level AWS implementation
```

The high-level code directly depends on infrastructure details.

Dependency inversion introduces an abstraction:

```text
High-level business logic
        ↓
Abstract capability
        ↑
Low-level implementation
```

Both sides depend on a contract.

---

# 8. High-Level vs Low-Level

### High-level policy

Examples:

```text
Do we have enough information to create a ticket?

Should this request be handed off?

Which workflow should handle the request?
```

These represent business/application behaviour.

### Low-level implementation

Examples:

```text
boto3 invocation
Bedrock request payload
Lambda ARN
DynamoDB conditional write
CloudWatch API call
```

These implement infrastructure details.

---

# 9. Why the High-Level Logic Should Dominate

Reliora's important rules include:

```text
do not create tickets prematurely
do not duplicate side effects
do not fabricate ticket IDs
do not answer unsupported policy questions
handoff when appropriate
```

These requirements should survive even if the AWS implementation changes.

Therefore they should not be buried inside vendor-specific code.

---

# 10. Reliora's Conceptual Provider Boundaries

The current architecture identifies provider responsibilities such as:

```text
Router
KnowledgeProvider
TicketProvider
HandoffProvider
TelemetryProvider
```

These describe capabilities rather than technologies.

---

# 11. `Router`

The `Router` answers a question such as:

> What intent category best represents this request?

Potential output:

```text
BUG
PLATFORM
OTHER
```

The application should not necessarily care whether routing is implemented using:

```text
deterministic rules
TF-IDF + Logistic Regression
Linear SVM
Bedrock LLM
hybrid routing
```

---

# 12. Router Contract

Conceptually:

```python
class Router:
    def route(self, request: RoutingRequest) -> RoutingDecision:
        ...
```

The exact implementation may evolve.

The stable idea is:

```text
normalized request
        ↓
Router
        ↓
RoutingDecision
```

---

# 13. Why This Helps the Routing Experiment

Reliora plans to compare routing approaches.

Without an abstraction:

```text
rules
classical ML
LLM
```

might each require different application code.

With a stable Router boundary:

```text
RuleRouter ──────┐
                 │
LogisticRouter ──┼→ RoutingDecision
                 │
LLMRouter ───────┘
```

the rest of the system can remain unchanged.

---

# 14. This Produces a Fairer Experiment

If all routers produce the same:

```text
RoutingDecision
```

then Reliora can compare:

```text
macro F1
per-class F1
latency
cost
failure behaviour
```

without changing downstream workflow logic.

That makes the experiment easier to interpret.

---

# 15. `KnowledgeProvider`

The `KnowledgeProvider` answers:

> What trusted knowledge supports this request?

It may eventually return:

```text
retrieved evidence
source identifiers
coverage state
metadata
```

depending on the design.

---

# 16. Knowledge Provider Implementations

One implementation could use:

```text
version-controlled FAQ
```

Another could later use:

```text
Amazon Bedrock Knowledge Base
```

Another could use:

```text
structured policy database
```

The application should depend primarily on the knowledge contract.

---

# 17. Why This Matters

Reliora currently does not need to introduce RAG simply for architectural prestige.

If:

```text
StaticFaqKnowledgeProvider
```

satisfies the requirement, it can be used.

Later:

```text
BedrockKnowledgeProvider
```

can replace it if the knowledge requirements justify that change.

---

# 18. This Is Technology Substitution

Conceptually:

```text
Application
     ↓
KnowledgeProvider
     ↓
┌─────────────────────────┐
│ Static FAQ              │
│ Bedrock Knowledge Base  │
│ Structured database     │
└─────────────────────────┘
```

The application workflow remains stable.

---

# 19. `TicketProvider`

The `TicketProvider` represents the side-effect boundary for ticket operations.

Conceptually:

```text
validated CreateBugReportRequest
        ↓
TicketProvider
        ↓
CreateBugReportResult
```

The application should not have to know every AWS implementation detail.

---

# 20. Ticket Provider Responsibilities

Depending on the final design, this boundary may be responsible for operations such as:

```text
create ticket
retrieve operation status
return authoritative ticket result
```

It should not decide unrelated application policy such as:

```text
whether enough fields exist
```

unless that validation is intentionally duplicated for defence in depth.

---

# 21. Application vs Provider Responsibility

Application:

```text
Is CREATE_TICKET allowed now?
```

TicketProvider:

```text
Perform the approved ticket operation safely.
```

This separation keeps workflow policy above infrastructure execution.

---

# 22. Idempotency Can Cross the Boundary

The application may provide:

```text
operation_id
```

to `TicketProvider`.

The provider/backend implementation can enforce:

```text
conditional uniqueness
result replay
```

close to the side-effect boundary.

Therefore idempotency is coordinated across layers.

---

# 23. Example Conceptual Contract

```python
class TicketProvider:
    def create_ticket(
        self,
        request: CreateBugReportRequest,
    ) -> CreateBugReportResult:
        ...
```

The application sends a validated request.

The provider returns a validated result.

---

# 24. Potential AWS Implementation

Later, an AWS-backed implementation might conceptually be:

```text
AwsTicketProvider
        ↓
AgentCore Gateway / MCP
        ↓
Lambda
        ↓
DynamoDB
```

The high-level application should not need this entire infrastructure chain embedded into its workflow logic.

---

# 25. `HandoffProvider`

The `HandoffProvider` represents the mechanism used to transfer or register escalation.

Its responsibility might eventually include:

```text
create handoff request
preserve selected context
record handoff reason
return handoff status
```

---

# 26. Why Handoff Needs a Provider Boundary

Today, handoff may simply be represented locally.

Later, it could connect to:

```text
support queue
ticketing system
human operator console
CRM
```

The workflow should not be rewritten every time the handoff backend changes.

---

# 27. `TelemetryProvider`

The `TelemetryProvider` represents operational telemetry.

Potential responsibilities include:

```text
record structured event
record metric
create trace annotation
record evaluation finding
```

---

# 28. Why Telemetry Should Not Dominate Business Logic

Application code should be able to say conceptually:

```text
record route decision
```

rather than repeatedly implementing:

```text
CloudWatch client creation
namespace configuration
metric formatting
trace API details
```

throughout the codebase.

---

# 29. Future Telemetry Implementations

Development:

```text
ConsoleTelemetryProvider
```

or:

```text
InMemoryTelemetryProvider
```

Cloud environment:

```text
CloudWatch / AgentCore observability implementation
```

Testing:

```text
FakeTelemetryProvider
```

The application contract can remain stable.

---

# 30. Why "Provider" Does Not Mean Cloud Provider

In this architecture:

```text
Provider
```

does not mean:

```text
AWS vs Azure vs GCP
```

It means:

> A component that provides a capability behind a defined interface.

For example:

```text
TicketProvider
```

provides ticket functionality.

---

# 31. Interface vs Implementation

These words are important.

### Interface / contract

Defines:

```text
what operations exist
which inputs are accepted
which outputs are returned
what errors can occur
```

### Implementation

Defines:

```text
how those operations actually happen
```

---

# 32. Example

Contract:

```text
Create one ticket and return its result.
```

Possible implementation:

```text
AgentCore Gateway
→ Lambda
→ DynamoDB
```

The second is one way of satisfying the first.

---

# 33. Protocols and Abstract Base Classes

Python offers several ways to represent interfaces.

Examples include:

```text
Protocol
Abstract Base Class
ordinary class convention
callable interfaces
```

Reliora should use whichever provides sufficient clarity without unnecessary complexity.

---

# 34. Python `Protocol`

Conceptually:

```python
from typing import Protocol


class TicketProvider(Protocol):
    def create_ticket(
        self,
        request: CreateBugReportRequest,
    ) -> CreateBugReportResult:
        ...
```

Any implementation with the required compatible method can satisfy the protocol.

---

# 35. Why Protocols Can Be Useful

They can provide:

```text
static typing
low runtime coupling
easy fakes
clear contracts
```

They work well with tools such as:

```text
mypy
```

---

# 36. Abstract Base Class

Another option is:

```python
from abc import ABC, abstractmethod


class TicketProvider(ABC):
    @abstractmethod
    def create_ticket(...):
        ...
```

Implementations explicitly inherit from the base class.

---

# 37. Protocol vs ABC

Neither is universally superior.

### Protocol

Useful when:

```text
structural typing is desirable
looser inheritance coupling is useful
```

### ABC

Useful when:

```text
explicit inheritance
shared implementation
runtime abstract-method enforcement
```

is desirable.

The choice should follow the project requirement.

---

# 38. Do Not Add Abstractions Without Purpose

Dependency inversion can also be overused.

For example, wrapping:

```python
len(value)
```

behind:

```text
LengthProvider
```

would add unnecessary complexity.

A provider boundary is justified when it isolates a meaningful capability or volatile dependency.

---

# 39. Good Candidates for Boundaries

External or likely-to-change capabilities are common candidates.

Examples:

```text
models
retrieval systems
databases
external APIs
telemetry backends
identity systems
ticket systems
```

These have real substitution or testing value.

---

# 40. Boundary Around Business Logic?

Not everything should become a provider.

For example:

```text
evaluate whether exactly one missing bug field is requested
```

is deterministic application logic.

It can simply be a function/module.

There may be no benefit in hiding it behind:

```text
BugInvariantProvider
```

---

# 41. Stable Core, Replaceable Edges

A useful architecture goal is:

```text
Stable core
→ business rules
→ workflow
→ contracts
→ validation

Replaceable edges
→ model
→ AWS services
→ retrieval implementation
→ telemetry implementation
```

This is sometimes described as keeping infrastructure at the edges.

---

# 42. Hexagonal Architecture Connection

This idea resembles concepts from:

```text
Hexagonal Architecture
Ports and Adapters
Clean Architecture
```

The exact architectural label is less important than the principle.

---

# 43. Port

A:

```text
port
```

represents what the application needs.

For example:

```text
TicketProvider
```

can be thought of as an outbound port.

---

# 44. Adapter

An:

```text
adapter
```

implements that port using a specific technology.

For example:

```text
AgentCoreGatewayTicketAdapter
```

could implement:

```text
TicketProvider
```

using AWS infrastructure.

---

# 45. Mental Model

```text
Application
     ↓
Port
TicketProvider
     ↓
Adapter
AWS implementation
     ↓
External system
```

This limits the amount of infrastructure knowledge inside the application core.

---

# 46. Why This Helps Testing

Suppose the real provider calls AWS.

Unit tests should not necessarily make live AWS calls.

Instead, a fake implementation can return controlled results.

---

# 47. Example Fake

Conceptually:

```python
class FakeTicketProvider:
    def create_ticket(self, request):
        return CreateBugReportResult(
            success=True,
            ticket_id="TEST-123",
        )
```

The test can now focus on application behaviour.

---

# 48. Why Fake Providers Are Useful

They can simulate:

```text
success
timeout
invalid response
duplicate replay
authorization failure
backend failure
```

without requiring real cloud resources.

This supports Reliora's evaluation-first approach.

---

# 49. Deterministic Tests Stay Cheap

If every unit test required:

```text
Bedrock
Lambda
DynamoDB
```

then tests would become:

```text
slow
costly
network-dependent
credential-dependent
less reproducible
```

Provider boundaries allow most behavioural tests to stay local.

---

# 50. But Fakes Do Not Replace Integration Tests

A fake can prove:

```text
application reacts correctly to a simulated timeout
```

It cannot prove:

```text
the real AgentCore Gateway configuration behaves identically
```

Therefore we need multiple test layers.

---

# 51. Test Pyramid Concept

Conceptually:

```text
many fast unit tests
        ↓
fewer integration tests
        ↓
fewer cloud/end-to-end tests
```

Each layer provides different evidence.

---

# 52. Unit Tests

With fake providers, verify things such as:

```text
missing fields block ticket creation
unsupported knowledge triggers handoff
provider error is handled safely
```

These should be fast.

---

# 53. Integration Tests

With real or realistic backend components, verify:

```text
request schema
AgentCore Gateway integration
Lambda invocation
DynamoDB idempotency
result parsing
```

These test component boundaries.

---

# 54. End-to-End Tests

Eventually:

```text
customer request
      ↓
full application
      ↓
provider implementations
      ↓
AWS services
      ↓
customer result
```

This validates the assembled system.

---

# 55. Provider Boundaries Make Failure Injection Easier

Suppose we want to test:

```text
TicketProvider timeout
```

A fake can deliberately raise a timeout.

The application should then demonstrate the required recovery behaviour.

---

# 56. Example

```text
FakeTicketProvider:
create_ticket → timeout

Expected Reliora behaviour:
do not fabricate success
do not fabricate ticket ID
apply retry policy
handoff if recovery exhausted
```

This is easy to test locally.

---

# 57. Failure Simulation Without Provider Boundary

If Lambda calls are embedded directly inside workflow code, simulating a timeout may require:

```text
extensive monkeypatching
AWS mocking
implementation-specific test logic
```

That increases test fragility.

---

# 58. Provider Boundaries Improve Fault Injection

Later Reliora can intentionally substitute:

```text
SlowTicketProvider
FailingKnowledgeProvider
MalformedResultProvider
```

during tests.

This creates controlled failure experiments.

---

# 59. Provider Boundaries Improve Observability

Each provider can emit standardized telemetry.

For example:

```text
provider = TicketProvider
operation = create_ticket
result = success
latency_ms = ...
```

This produces consistent operational signals.

---

# 60. Provider Boundaries Improve Cost Attribution

If calls pass through defined providers, Reliora can measure:

```text
router calls
knowledge calls
ticket calls
model tokens
provider latency
```

This helps later FinOps analysis.

---

# 61. Provider Boundaries Improve Latency Attribution

Suppose end-to-end latency is:

```text
1.4 seconds
```

A trace might show:

```text
Router          100 ms
Knowledge       250 ms
LLM generation  900 ms
Other           150 ms
```

The architecture reveals where optimization matters.

---

# 62. Provider Boundaries Improve Error Classification

Instead of:

```text
support request failed
```

Reliora can identify:

```text
ROUTER_FAILURE
KNOWLEDGE_FAILURE
TICKET_PROVIDER_FAILURE
HANDOFF_PROVIDER_FAILURE
```

This improves diagnosis.

---

# 63. Normalized Errors

External APIs often return vendor-specific errors.

For example:

```text
botocore exception
HTTP status
provider-specific code
```

An adapter can translate these into application-level errors.

---

# 64. Example

Infrastructure error:

```text
EndpointConnectionError
```

Application-level meaning:

```text
TicketProviderUnavailable
```

The workflow can respond to the business meaning rather than vendor details.

---

# 65. Why Error Translation Matters

If business logic depends directly on every AWS exception class:

```text
AWS details leak into application core
```

This increases coupling.

A normalized error vocabulary provides a cleaner boundary.

---

# 66. Do Not Hide Useful Failure Information

Normalization should not destroy diagnostics.

An application error can preserve:

```text
high-level category
cause
correlation ID
safe metadata
```

while avoiding unnecessary infrastructure coupling.

---

# 67. Dependency Injection

Once application code depends on provider interfaces, the actual implementation must be supplied somehow.

This is called:

```text
dependency injection
```

---

# 68. Without Dependency Injection

Weak pattern:

```python
class BugWorkflow:
    def __init__(self):
        self.ticket_provider = AwsTicketProvider()
```

The workflow chooses its own infrastructure.

This makes substitution harder.

---

# 69. With Dependency Injection

Conceptually:

```python
class BugWorkflow:
    def __init__(self, ticket_provider: TicketProvider):
        self.ticket_provider = ticket_provider
```

Now whoever constructs the application supplies the implementation.

---

# 70. Production Construction

```text
BugWorkflow
    ↓
AwsTicketProvider
```

---

# 71. Test Construction

```text
BugWorkflow
    ↓
FakeTicketProvider
```

Same workflow.

Different dependency.

---

# 72. Why This Matters

Application logic no longer decides:

```text
which infrastructure exists
```

It declares:

```text
which capability it requires
```

The composition layer supplies the implementation.

---

# 73. Composition Root

A composition root is the place where an application's major dependencies are assembled.

Conceptually:

```python
router = BedrockRouter(...)
knowledge = StaticFaqProvider(...)
tickets = AwsTicketProvider(...)
telemetry = CloudWatchTelemetry(...)

app = SupportApplication(
    router=router,
    knowledge_provider=knowledge,
    ticket_provider=tickets,
    telemetry=telemetry,
)
```

The exact design can remain simple.

---

# 74. Why Centralized Construction Helps

It creates one place to see:

```text
which implementations are active
```

instead of discovering them throughout the codebase.

This helps with:

```text
configuration
testing
deployment
debugging
```

---

# 75. Configuration Is Not Business Logic

Values such as:

```text
AWS region
model ID
Lambda function name
knowledge source location
timeouts
```

belong primarily to configuration/infrastructure concerns.

They should not be scattered through workflow rules.

---

# 76. Provider Interfaces Support Configuration Changes

For example:

```text
model = Nova Pro
```

can later become:

```text
model = another Bedrock model
```

through the router implementation/configuration.

The business workflow still consumes:

```text
RoutingDecision
```

---

# 77. Provider Boundaries and Model Portability

This is particularly important for AI systems because models evolve quickly.

A model adapter can isolate:

```text
request syntax
response syntax
model identifier
provider metadata
```

from application behaviour.

---

# 78. Switching Models Is Still Not Free

An abstraction does not guarantee:

```text
all models behave identically
```

A model change still requires:

```text
evaluation
latency measurement
cost measurement
safety regression testing
```

The abstraction makes the implementation change easier.

It does not eliminate validation.

---

# 79. Abstraction Does Not Remove Semantics

Suppose both models implement:

```text
Router
```

but Model B performs badly.

Technically satisfying the interface does not mean satisfying the business requirement.

Contracts define structure.

Evaluations establish quality.

---

# 80. This Mirrors Structured Outputs

A structured:

```text
RoutingDecision
```

may be schema-valid.

But:

```text
intent
```

can still be wrong.

Likewise, a provider can satisfy the programming interface while providing poor results.

Multiple quality layers remain necessary.

---

# 81. Provider Boundaries and AWS

Reliora can remain AWS-native while still using abstraction boundaries.

This is not an attempt to hide AWS or make every component cloud-agnostic.

The purpose is:

```text
separation of responsibilities
testability
replaceability where useful
```

---

# 82. Avoid Fake Cloud Agnosticism

A portfolio project should not claim:

```text
fully cloud agnostic
```

merely because an interface exists.

If the infrastructure is built deeply around:

```text
AgentCore
Bedrock
Lambda
DynamoDB
CloudWatch
```

then AWS remains an intentional architectural choice.

The boundary simply keeps AWS-specific implementation out of unrelated business code.

---

# 83. Good AWS-Specific Architecture

A strong design can say:

> The system is intentionally implemented on AWS, while application-level provider contracts isolate model, retrieval, side-effect, and telemetry implementations to improve testing and reduce infrastructure coupling.

That is more accurate than pretending the platform dependency does not exist.

---

# 84. Provider Boundaries and Terraform

Terraform manages infrastructure such as:

```text
Lambda
IAM
DynamoDB
observability
```

Application provider interfaces consume the resulting resources.

These are different architectural layers.

---

# 85. Terraform Should Not Define Business Behaviour

Terraform can declare:

```text
DynamoDB table
```

It should not define:

```text
ask exactly one missing field at a time
```

Infrastructure-as-code and application policy solve different problems.

---

# 86. Provider Boundaries Help Local Development

Before AWS resources exist, local providers can support development.

For example:

```text
StaticFaqKnowledgeProvider
InMemoryTicketProvider
ConsoleTelemetryProvider
```

This allows core behaviour to be tested without cloud cost.

---

# 87. This Supports Reliora's Current Phase

Reliora began with:

```text
evaluation-first local engineering
```

before deploying AWS resources.

Provider boundaries make this development order easier.

Application logic can mature before infrastructure is required.

---

# 88. Local Provider Does Not Prove Cloud Integration

Passing local tests supports:

```text
application behaviour
```

not:

```text
AWS integration success
```

Once again, evidence scope matters.

Cloud integration receives its own tests later.

---

# 89. Provider Contracts Should Be Small

A weak interface might become:

```text
EverythingProvider
```

with dozens of unrelated methods.

That simply recreates coupling behind one large class.

---

# 90. Interface Segregation

A useful related principle is:

> Components should depend only on the capabilities they actually need.

For example:

```text
BugWorkflow
```

may need:

```text
TicketProvider
```

but not:

```text
KnowledgeProvider
```

unless its requirements actually involve knowledge retrieval.

---

# 91. Smaller Interfaces Improve Tests

A fake:

```text
TicketProvider
```

is easier to implement than a fake provider containing:

```text
routing
knowledge
tickets
telemetry
identity
handoff
```

Small interfaces reduce testing burden.

---

# 92. Providers Should Have Cohesive Responsibilities

`TicketProvider` should focus on ticket capability.

`TelemetryProvider` should focus on telemetry.

If one provider owns:

```text
tickets
model routing
authentication
metrics
```

it lacks cohesion.

---

# 93. What Is Cohesion?

Cohesion describes how closely related the responsibilities inside a component are.

High cohesion means:

```text
component has a focused purpose
```

This generally makes code easier to:

```text
understand
test
change
```

---

# 94. Low Coupling + High Cohesion

A common software-design goal is:

```text
low unnecessary coupling
+
high cohesion
```

Provider boundaries help support this.

---

# 95. Boundary Placement Requires Judgment

Too few boundaries:

```text
everything becomes tangled
```

Too many:

```text
architecture becomes abstract and difficult to navigate
```

The right design isolates meaningful responsibilities.

---

# 96. Architecture Should Remain Understandable

For Reliora, a new engineer should be able to understand:

```text
Router
→ chooses intent

KnowledgeProvider
→ provides trusted knowledge

TicketProvider
→ performs ticket operations

HandoffProvider
→ performs escalation

TelemetryProvider
→ records operational signals
```

If explaining the abstraction becomes harder than explaining the implementation, it may be over-engineered.

---

# 97. Provider Boundaries and Repository Structure

As Reliora grows, provider contracts may live near:

```text
application/domain interfaces
```

while implementations may live under:

```text
infrastructure/adapters
```

The exact directory structure should follow the actual code that emerges.

Do not create elaborate empty directories merely to imitate enterprise repositories.

---

# 98. Folder Structure Should Follow Responsibilities

Conceptually:

```text
src/reliora/
├── application/
├── evaluation/
├── providers/
└── infrastructure/
```

could eventually make sense.

But the repository should evolve from real needs.

The existing repository map should be updated only when actual files justify the structure.

---

# 99. Avoid Architecture Theatre

Creating folders named:

```text
domain
adapters
ports
services
interfaces
gateways
repositories
```

without meaningful code inside them does not make a project sophisticated.

Architecture quality comes from:

```text
clear responsibilities
testability
evidence
appropriate trade-offs
```

not folder count.

---

# 100. Provider Boundary as an Architectural Decision

A boundary should answer:

```text
What volatile/external capability are we isolating?

Which part of the application depends on it?

What contract is stable?

What failures cross the boundary?

How will it be tested?
```

If these questions have no good answer, the abstraction may be unnecessary.

---

# 101. Example: Router Decision

Why abstract?

```text
multiple implementations will be benchmarked
model providers can change
offline fakes are useful
```

Strong justification.

---

# 102. Example: Ticket Provider Decision

Why abstract?

```text
external side effect
cloud dependency
failure injection required
idempotency tests required
```

Strong justification.

---

# 103. Example: Telemetry Provider Decision

Why abstract?

```text
local and AWS telemetry differ
business logic should not embed CloudWatch calls
testing should not emit real telemetry
```

Reasonable justification.

---

# 104. Provider Interfaces Need Failure Contracts

A provider contract should describe not only successful output.

It should also define expected failure categories.

For example:

```text
TicketProvider
→ success
→ timeout
→ unavailable
→ rejected request
→ duplicate replay
```

The application needs predictable failure semantics.

---

# 105. Why Failure Contracts Matter

If every implementation throws unrelated exceptions:

```text
workflow logic becomes provider-specific again
```

The abstraction only works if important failures are normalized appropriately.

---

# 106. Timeouts Belong in the Contract

External operations need defined timeout behaviour.

For example:

```text
TicketProvider.create_ticket
```

should not hang indefinitely.

Timeout policies support:

```text
latency control
retry safety
handoff
cost control
```

---

# 107. Cancellation Matters Too

In long workflows, the application may need to stop an operation because:

```text
user disconnected
deadline exceeded
workflow cancelled
```

Provider design should eventually consider cancellation where the underlying technology supports it and the requirement warrants it.

---

# 108. Provider Boundaries and Async Code

Some external providers may eventually use:

```python
async def
```

because they perform network I/O.

Whether Reliora should be synchronous or asynchronous should be driven by:

```text
concurrency requirements
framework
latency
complexity
```

not by fashion.

---

# 109. Do Not Add Async Prematurely

For a small support workflow, synchronous code may initially be simpler.

If real concurrency/latency requirements justify asynchronous execution later, provider boundaries can ease the transition.

---

# 110. Provider Boundaries and Caching

Suppose knowledge retrieval later benefits from caching.

The application should not need caching code everywhere.

The `KnowledgeProvider` implementation can potentially handle:

```text
cache lookup
retrieval
cache population
```

if that design is justified.

---

# 111. But Hidden Behaviour Needs Observability

If a provider performs caching, telemetry should make outcomes such as:

```text
cache hit
cache miss
```

observable when operationally relevant.

Abstraction should not mean invisibility.

---

# 112. Provider Boundaries and Security

Each provider creates a security boundary.

For example:

```text
TicketProvider
```

may require write permissions.

```text
KnowledgeProvider
```

may require read permissions.

These can use different IAM privileges.

---

# 113. This Supports Least Privilege

Instead of one component requiring every permission:

```text
Router:
Bedrock invoke only

Ticket implementation:
Gateway/Lambda permissions

Telemetry implementation:
observability permissions
```

Exact IAM architecture will depend on deployment boundaries.

But separation helps reason about least privilege.

---

# 114. Provider Boundary Does Not Automatically Create IAM Isolation

If everything runs in one process under one broad IAM role, Python interfaces alone do not enforce cloud privilege separation.

This is another important evidence/architecture distinction.

---

# 115. Logical Boundary vs Security Boundary

```text
Python provider interface
→ logical software boundary
```

A stronger security boundary may require:

```text
separate IAM role
separate service
policy enforcement
separate runtime
```

Do not claim security isolation merely because classes are separated.

---

# 116. Boundary Types Matter

A system can have:

```text
code boundary
process boundary
network boundary
identity boundary
security boundary
```

These are not equivalent.

Reliora should describe exactly which kind exists.

---

# 117. Microservices Are Not Automatically Required

Separating:

```text
Router
TicketProvider
KnowledgeProvider
```

in code does not mean each must become its own microservice.

Logical separation can exist inside one deployable application.

---

# 118. Why Not Automatically Use Microservices?

Microservices introduce:

```text
network calls
deployment complexity
distributed tracing
service discovery
failure modes
cost
```

They should solve a real scaling/team/security requirement.

---

# 119. Modular Monolith Can Be Appropriate

A well-structured single application with provider boundaries can offer:

```text
clear modules
testability
simple deployment
lower operational overhead
```

while requirements remain modest.

This is often a strong starting point.

---

# 120. Extract Services Only When Justified

A provider might later become a separate service because of:

```text
independent scaling
different security boundary
different deployment lifecycle
shared use across applications
team ownership
```

The boundary makes that extraction easier.

But it should not be assumed from day one.

---

# 121. This Is Why "Boring Technology" Matters

A clean Python module and interface can sometimes solve the problem better than:

```text
five microservices
Kafka
Kubernetes
service mesh
```

The simplest architecture that satisfies the requirement is often easier to operate and defend.

---

# 122. Provider Boundaries Support Future Refactoring

If requirements grow, existing logical boundaries create natural seams.

For example:

```text
InMemoryTicketProvider
        ↓
AwsTicketProvider
        ↓
later separate ticket service
```

The contract can remain relatively stable through those transitions.

---

# 123. Provider Boundaries and ADRs

Important implementation choices can be captured as architecture decisions.

For example:

```text
Why use a provider boundary?

Why use AgentCore Gateway?

Why retain static FAQ instead of mandatory RAG?

Why use managed observability?
```

The provider contract and the infrastructure decision are separate ADR concerns.

---

# 124. Example Architecture Reasoning

```text
Requirement:
benchmark multiple routing approaches

Architectural response:
stable Router contract

Experiment:
RuleRouter vs LogisticRouter vs LLMRouter

Decision:
select candidate based on quality/latency/cost evidence
```

This connects requirement, architecture, experiment, and evidence.

---

# 125. Another Example

```text
Requirement:
duplicate retries must not create multiple tickets

Architectural response:
TicketProvider receives stable operation ID

Backend:
conditional idempotency enforcement

Evaluation:
duplicate and timeout-after-commit tests
```

Again, each layer has a clear responsibility.

---

# 126. Application Code Should Read Like the Business Workflow

A strong high-level workflow should conceptually resemble:

```text
route request
check knowledge/workflow state
validate action
authorize action
execute provider
respond
```

rather than:

```text
construct boto3 client
serialize AWS payload
inspect AWS error
construct another AWS client
```

Infrastructure details belong lower in the dependency graph.

---

# 127. Infrastructure Code Should Read Like Infrastructure

Conversely, an AWS provider can focus on:

```text
API calls
AWS authentication
service configuration
timeouts
response normalization
```

without deciding broad customer-support policy.

This separation improves cohesion.

---

# 128. Testing the Contract

Provider implementations should have tests proving that they satisfy expected contracts.

For example:

```text
TicketProvider success
→ returns valid CreateBugReportResult

duplicate replay
→ returns original result

backend malformed result
→ normalized failure
```

---

# 129. Contract Tests

When multiple implementations exist, the same suite can sometimes be run against all of them.

For example:

```text
InMemoryTicketProvider
AwsTicketProvider
```

could share behavioural contract tests where appropriate.

This reduces implementation drift.

---

# 130. But Cloud Tests May Have Extra Requirements

An AWS provider additionally needs tests for:

```text
IAM
networking
service configuration
AWS-specific failure modes
```

Shared contract tests do not replace implementation-specific integration tests.

---

# 131. Dependency Inversion and Interview Questions

Interviewers may ask:

```text
How would you make the system testable?

How do you avoid vendor logic everywhere?

How would you compare multiple LLM providers?

How would you test tool failures without AWS?

How would you migrate from static FAQ to RAG?
```

Provider boundaries provide a coherent answer.

---

# 132. Weak Interview Answer

> I would mock boto3 everywhere.

This may work technically, but it leaves business code coupled directly to boto3.

---

# 133. Stronger Interview Answer

> I isolate external capabilities behind narrow typed interfaces. Application workflows depend on a `TicketProvider` or `KnowledgeProvider`, while AWS-specific adapters implement those contracts. Unit tests inject deterministic fakes, and integration tests verify the real adapter. This lets me test workflow logic without cloud dependencies while still validating AWS integration separately.

This communicates architecture rather than just mocking technique.

---

# 134. Important Lessons

1. Application business logic should not unnecessarily depend directly on infrastructure details.
2. A provider boundary represents a capability rather than a specific technology.
3. Dependency inversion makes high-level workflow logic depend on stable abstractions instead of volatile implementations.
4. Reliora's conceptual boundaries include `Router`, `KnowledgeProvider`, `TicketProvider`, `HandoffProvider`, and `TelemetryProvider`.
5. The Router can support rules, classical ML, LLM, or hybrid implementations behind one decision contract.
6. Knowledge-provider separation allows static FAQ and future RAG approaches to be substituted when requirements justify it.
7. Ticket-provider separation isolates privileged side effects and makes idempotency and failure testing easier.
8. Telemetry-provider separation prevents CloudWatch-specific details from spreading throughout business logic.
9. Interfaces define what a capability does; adapters define how it is implemented.
10. Python `Protocol` or abstract base classes are possible interface mechanisms, but abstraction should remain proportional to need.
11. Provider boundaries make deterministic fake implementations possible for fast local tests.
12. Fake providers do not replace integration or end-to-end testing.
13. Failure injection becomes easier when provider outcomes can be controlled deliberately.
14. External provider errors should be normalized into meaningful application failure categories where appropriate.
15. Dependency injection allows production and test implementations to use the same application workflow.
16. A composition root centralizes which concrete provider implementations are active.
17. Structured provider inputs and outputs strengthen type checking, validation, testing, and observability.
18. Provider boundaries can improve latency, error, and cost attribution.
19. Provider boundaries can help reason about least privilege, but Python interfaces alone do not create IAM or security isolation.
20. Logical code boundaries, process boundaries, network boundaries, and security boundaries are different things.
21. Modular separation does not require microservices.
22. Microservices should be introduced only when independent scaling, security, ownership, or lifecycle requirements justify them.
23. A modular monolith can be a strong production-oriented architecture when complexity is still limited.
24. Stable boundaries create seams that make future infrastructure refactoring easier.
25. Architecture should isolate meaningful volatile dependencies without creating abstraction for its own sake.
26. Folder count and design-pattern vocabulary are not evidence of architectural quality.
27. Provider contracts should remain cohesive and small enough to understand.
28. Application code should express business workflow; infrastructure code should express infrastructure behaviour.
29. Contracts establish structure, while evaluation establishes whether an implementation is actually good enough.
30. The goal is a **stable, testable application core surrounded by replaceable infrastructure adapters where replacement or isolation provides real value**.

---

## Interview Explanation

> I use narrow provider boundaries to keep Reliora's application logic independent from unnecessary AWS and model-specific implementation details. For example, the workflow depends on a `Router`, `KnowledgeProvider`, and `TicketProvider` contract rather than embedding Bedrock or boto3 calls throughout the business logic. That lets me benchmark multiple routing implementations behind the same interface, use deterministic fake providers for unit and failure-injection tests, and then validate the real AgentCore, Lambda, DynamoDB, or observability adapters separately through integration tests. I am not trying to make an AWS-native project artificially cloud-agnostic; the purpose is to keep the business rules stable, the infrastructure edges replaceable where useful, and the failure boundaries easier to test and observe.
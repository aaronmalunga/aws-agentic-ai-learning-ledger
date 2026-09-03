# Terraform, Boto3, and Botocore

## Why This Lesson Exists

During the Reliora AWS implementation, Terraform and Boto3 appeared in the project at roughly the same time.

They both interact with AWS, but they solve fundamentally different problems.

The simplest mental model is:

```text
Terraform
-> defines and provisions AWS infrastructure

Boto3
-> lets running Python code call AWS services

Botocore
-> provides much of the lower-level AWS machinery used by Boto3
```

Understanding this distinction prevents infrastructure provisioning logic from being mixed with application runtime behaviour.

---

# 1. Terraform

Terraform is infrastructure as code.

Reliora currently uses Terraform to describe resources such as:

```text
DynamoDB tables
AWS provider configuration
environment configuration
future Lambda resources
future IAM resources
future observability resources
```

For example, Terraform can declare:

```hcl
resource "aws_dynamodb_table" "operations" {
  name         = "reliora-${var.environment}-operations"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "operation_id"
}
```

Terraform answers a question such as:

> What AWS infrastructure should exist?

It does not normally execute Reliora's customer-support business workflow.

---

# 2. Terraform Lifecycle

A simplified Terraform lifecycle is:

```text
write infrastructure configuration
-> terraform init
-> terraform validate
-> terraform plan
-> review proposed changes
-> terraform apply
-> AWS resources exist
```

Later:

```text
terraform destroy
```

can remove managed development infrastructure when it is no longer required.

This supports:

```text
reproducibility
reviewability
auditability
repeatable deployment
controlled teardown
```

---

# 3. Boto3

Boto3 is the official AWS SDK for Python.

It allows running Python code to make AWS API calls.

For example:

```python
import boto3

dynamodb = boto3.client("dynamodb")
```

Application code can then perform operations such as:

```python
dynamodb.get_item(...)
```

or:

```python
dynamodb.transact_write_items(...)
```

Boto3 answers a different question:

> How does this running Python workload interact with an AWS service?

---

# 4. Terraform and Boto3 in Reliora

The two technologies occupy different layers.

Conceptually:

```text
Terraform
        |
        v
creates DynamoDB tables
        |
        v
AWS infrastructure exists
        |
        v
Reliora Python runtime
        |
        v
Boto3
        |
        v
DynamoDB API
```

Terraform creates the infrastructure.

Boto3 uses the infrastructure at runtime.

---

# 5. What Terraform Should Not Do

Terraform should not define business rules such as:

```text
same operation + same payload
-> REPLAYED

same operation + different payload
-> CONFLICT
```

Those are application and reliability semantics.

Terraform can create the DynamoDB tables that support those behaviours, but the behaviour itself belongs in application/runtime code.

---

# 6. What Boto3 Should Not Do

Boto3 should not become the architecture of the entire application.

A weak design could spread code such as:

```python
boto3.client(...)
```

through:

```text
domain models
authorization logic
business workflows
evaluation logic
routing logic
```

That would couple unrelated application code directly to AWS APIs.

Reliora instead isolates AWS-specific runtime behaviour under:

```text
src/reliora/adapters/aws/
```

This allows higher-level application logic to depend on contracts such as:

```text
TicketExecutionPort
TicketProvider
```

rather than directly depending on DynamoDB API calls.

---

# 7. Boto3 Client API

Boto3 supports a low-level client interface.

For example:

```python
dynamodb = boto3.client("dynamodb")
```

This maps closely to AWS service APIs.

Reliora's durable execution implementation needs operations such as:

```text
GetItem
TransactWriteItems
ConditionExpression
ConsistentRead
```

A client-style interface is useful when the implementation needs explicit control over these DynamoDB semantics.

The choice should be driven by the required API behaviour rather than by a rule that one Boto3 interface is always better.

---

# 8. Boto3 Resource API

Some AWS services also expose higher-level Boto3 resource interfaces.

For DynamoDB, an example is:

```python
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("example-table")
```

Resource interfaces can be convenient for common operations.

However, reliability-sensitive code may benefit from using APIs that expose the exact transactional and conditional behaviour required by the design.

---

# 9. Botocore

Installing Boto3 also installs Botocore.

During Reliora development, `uv add boto3` resolved:

```text
boto3==1.43.87
botocore==1.43.87
```

Botocore provides much of the lower-level machinery behind Boto3.

Conceptually:

```text
Reliora Python code
        |
        v
Boto3
developer-facing SDK
        |
        v
Botocore
lower-level AWS machinery
        |
        v
AWS HTTPS APIs
```

---

# 10. What Botocore Handles

Botocore is involved in capabilities such as:

```text
AWS service models
request construction
authentication and request signing
endpoint resolution
retry behaviour
response parsing
AWS error representation
```

Most Reliora application code does not need to work with Botocore directly.

One important exception is AWS error handling.

For example:

```python
from botocore.exceptions import ClientError
```

The DynamoDB execution adapter uses `ClientError` to inspect AWS transaction failures.

---

# 11. Supporting Dependencies

Adding Boto3 also installed supporting packages including:

```text
jmespath
python-dateutil
s3transfer
six
urllib3
```

These are transitive dependencies.

Reliora did not independently decide that its architecture required each package.

The package manager resolved them because Boto3 depends on them.

This is one reason the lockfile matters.

---

# 12. Explicit Dependency Management

Before adding Boto3, the project was checked with:

```powershell
git grep -n "boto3"
```

and:

```powershell
Get-Content "pyproject.toml"
```

Boto3 was not already declared or used.

It was then added with:

```powershell
uv add boto3
```

This updated:

```text
pyproject.toml
uv.lock
local project environment
```

The resolved version was verified with:

```powershell
uv run python -c "import boto3; print(boto3.__version__)"
```

Observed result:

```text
1.43.87
```

---

# 13. Why Declare Boto3 Explicitly?

AWS Lambda environments may provide AWS SDK libraries.

However, Reliora also needs predictable behaviour for:

```text
local development
unit tests
CI
dependency resolution
packaging
reproducible builds
```

Depending silently on whatever happens to exist on one machine would weaken reproducibility.

Therefore Boto3 became an explicit project dependency.

---

# 14. Terraform Provider vs Boto3 SDK

These concepts can sound similar but should not be confused.

```text
Terraform AWS provider
-> Terraform plugin that allows Terraform to manage AWS resources

Boto3
-> Python SDK used by running Python applications
```

For example:

```text
Terraform AWS provider
-> creates reliora-dev-operations table

Boto3 DynamoDB client
-> reads and writes items in reliora-dev-operations
```

---

# 15. Authentication

Terraform and Boto3 ultimately need AWS credentials when making real AWS API calls.

They can use AWS credential-provider mechanisms rather than hard-coded secrets.

For local Reliora development, temporary AWS authentication can be supplied through the configured AWS profile.

The infrastructure configuration itself should not contain:

```text
access keys
secret keys
session tokens
```

Later CI/CD can use short-lived AWS identity through GitHub OIDC rather than long-lived credentials.

---

# 16. Testing Boundary

The DynamoDB adapter accepts a minimal DynamoDB client contract.

Unit tests therefore inject a deterministic fake client rather than making real AWS calls.

Conceptually:

```text
production
-> real Boto3 DynamoDB client

unit tests
-> FakeDynamoDBClient
```

This means application reliability tests do not require:

```text
AWS credentials
network connectivity
live DynamoDB tables
AWS cost
```

Live AWS integration remains a separate evidence gate.

---

# 17. Infrastructure Tests vs Runtime Tests

Terraform validation can prove things such as:

```text
configuration parses
provider schema is compatible
required variables exist
```

It does not prove:

```text
Reliora replay behaviour is correct
```

Likewise, Boto3 adapter unit tests can prove application decisions represented by those tests.

They do not prove:

```text
Terraform can deploy the infrastructure successfully
IAM permissions are correct
real AWS concurrency behaves as expected
```

Different layers require different evidence.

---

# 18. Current Reliora Evidence

As of 2026-09-03:

```text
Terraform initialization
-> complete

Terraform validation
-> successful

AWS provider
-> configured

DynamoDB table definitions
-> created in Terraform

Boto3
-> explicit runtime dependency

DynamoDB durable execution adapter
-> implemented

AWS adapter unit tests
-> 7 passed

complete Reliora regression suite
-> 171 passed
```

Live AWS deployment and integration remain separate future evidence.

---

# 19. Engineering Mental Model

A useful summary is:

```text
Terraform
-> infrastructure lifecycle

Boto3
-> runtime AWS interaction

Botocore
-> lower-level SDK machinery

Reliora application contracts
-> business and reliability semantics
```

These layers cooperate but should not be collapsed into one concern.

---

# 20. Important Lessons

1. Terraform and Boto3 solve different problems.
2. Terraform describes infrastructure state.
3. Boto3 allows Python workloads to call AWS services at runtime.
4. Botocore supplies much of the lower-level machinery underneath Boto3.
5. Application business rules should not be encoded in Terraform.
6. AWS SDK calls should not leak unnecessarily into application and domain logic.
7. Infrastructure adapters provide a useful boundary for Boto3-specific code.
8. Explicit dependencies improve local, CI, and packaging reproducibility.
9. A Terraform validation result does not prove runtime application behaviour.
10. A mocked or fake Boto3 test does not prove live AWS integration.
11. Infrastructure, runtime, and business semantics need separate verification.
12. AWS-native architecture can still use clean software boundaries without pretending to be cloud-agnostic.

---

## Interview Explanation

> I separate AWS provisioning from runtime service interaction. Terraform defines what infrastructure should exist, such as Reliora's DynamoDB tables and later Lambda and IAM resources. Boto3 is used by the running Python application to call AWS APIs such as DynamoDB `GetItem` and `TransactWriteItems`, while Botocore provides much of the lower-level request, signing, service-model, and error machinery underneath Boto3. I keep those SDK calls inside AWS adapters so application-level reliability rules remain independent of the API mechanics. I also test the adapter with deterministic fake clients locally, then treat Terraform deployment, IAM, and live AWS behaviour as separate integration evidence gates.
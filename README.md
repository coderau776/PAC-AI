# PAC-AI

## PR Policy Guard

PR Policy Guard is an AI-assisted Policy-as-Code platform designed to help developers comply with firm policies without requiring them to manually read and interpret large policy documents for every pull request.

The system combines deterministic code analysis, LLM-based semantic understanding, hybrid policy discovery using structured search + RAG/vector search, an MCP-based policy platform, and a deterministic Rule Engine / OPA enforcement layer.

The core design principle is:

> **AI understands → RAG discovers → Policy Registry provides the source of truth → Rule Engine / OPA decides → AI explains and helps remediate**

---

## 1. Problem

Enterprise firms often have hundreds or thousands of policies covering areas such as:

- Security
- Data privacy and PII
- Dependency and license management
- Logging
- API design
- Authentication and authorization
- Container and infrastructure configuration
- Architecture standards
- Secrets management
- Compliance and regulatory requirements

Developers should not have to manually search through all of these documents every time they create or modify an application.

PR Policy Guard moves policy discovery and enforcement into the pull-request workflow.

Instead of asking a developer to determine:

> "Which firm policies apply to my change?"

PR Policy Guard determines the relevant policies, evaluates the code against them, and explains the required changes directly in the PR.

---

# 2. High-Level Architecture

```text
Developer
    |
    | Creates / updates PR
    v
+----------------------+
| Git Provider         |
| GitHub / GitLab /    |
| Bitbucket            |
+----------+-----------+
           |
           | PR webhook
           v
+----------------------+
| PR Collector         |
| - Validates event    |
| - Gets PR metadata   |
| - Gets changed files |
+----------+-----------+
           |
           v
+-------------------------------+
| PR Analysis                   |
|                               |
|  +-------------------------+  |
|  | Deterministic Analysis  |  |
|  | - git diff              |  |
|  | - AST                    |  |
|  | - dependencies           |  |
|  | - configs / IaC          |  |
|  | - static rules           |  |
|  +-------------------------+  |
|                               |
|  +-------------------------+  |
|  | LLM Semantic Analysis   |  |
|  | - purpose of change     |  |
|  | - data classification   |  |
|  | - PII / secret signals  |  |
|  | - logging / API intent  |  |
|  +-------------------------+  |
+---------------+---------------+
                |
                | Code facts + signals
                v
+-----------------------------------------------+
| Hybrid Policy Discovery                       |
|                                               |
|  Structured filtering      Semantic RAG       |
|  - domain                 - embeddings        |
|  - language               - vector search     |
|  - artifact type          - top-K documents   |
|  - environment                                |
+-------------------+---------------------------+
                    |
                    | Candidate policy/doc IDs
                    v
+-----------------------------------------------+
| Policy Registry                               |
|                                               |
| Authoritative policy metadata + versions +    |
| applicability + severity + exceptions + Rego  |
+-------------------+---------------------------+
                    |
                    v
+-----------------------------------------------+
| Applicability Determination + Input Builder   |
|                                               |
| Builds canonical PRPolicyInput JSON            |
+-------------------+---------------------------+
                    |
                    v
+-----------------------------------------------+
| Rule Engine / OPA                             |
|                                               |
| Deterministic policy evaluation               |
| - evaluates rules                             |
| - checks exceptions                            |
| - produces violations / warnings / decisions |
+-------------------+---------------------------+
                    |
                    | Evaluation result
                    v
+-----------------------------------------------+
| AI Explanation & Remediation Agent            |
| LLM + RAG                                     |
| - explains violation                          |
| - retrieves relevant policy guidance           |
| - suggests remediation                         |
| - generates code examples                      |
+-------------------+---------------------------+
                    |
                    v
+-----------------------------------------------+
| PR Feedback                                   |
| - PR comment                                  |
| - status / check                              |
| - violations and fixes                        |
| - policy references                           |
+-----------------------------------------------+
```

---

# 3. Architecture Components

## 3.1 Developer

The developer works normally by creating or updating a pull request.

No special policy workflow is required from the developer.

```text
Developer -> Pull Request -> Git Provider
```

---

## 3.2 Git Provider

The Git provider can be GitHub, GitLab, Bitbucket, or another supported source-control platform.

It provides:

- Pull-request metadata
- Repository information
- Base and head commit information
- Changed files
- Diff
- Author information
- Branch information
- Labels and PR metadata

A webhook/event triggers the PR Policy Guard pipeline.

---

## 3.3 PR Collector

The PR Collector is the entry point for the PR analysis pipeline.

Responsibilities:

- Validate the webhook
- Extract PR metadata
- Identify repository, branch and commits
- Retrieve the changed files/diff
- Create an analysis job
- Trigger downstream processing

The collector should remain lightweight. Heavy analysis should be performed asynchronously.

---

# 4. PR Analysis Layer

The analysis layer is deliberately split into **deterministic analysis** and **LLM semantic analysis**.

This avoids using an LLM for information that can be extracted reliably and cheaply using normal tooling.

## 4.1 Deterministic Analysis

The deterministic analyzers extract facts from the repository.

Examples:

- Changed files
- File types
- Programming languages
- AST structure
- Imports
- Function/method calls
- API endpoints
- Dependency changes
- Dependency versions
- Dependency licenses
- Dockerfile changes
- Base image changes
- Infrastructure-as-Code changes
- Configuration changes
- Static analysis results
- Secret/configuration patterns

Example output:

```json
{
  "repository": "customer-service",
  "language": "java",
  "changed_files": [
    "src/main/java/com/example/UserService.java",
    "pom.xml"
  ],
  "dependency_added": true,
  "logging_changed": true,
  "api_changed": false
}
```

---

## 4.2 LLM Semantic Analysis

Some policy-relevant information cannot be reliably determined using simple static analysis.

The LLM is used for semantic understanding such as:

- Understanding the purpose of the change
- Identifying the type of data being processed
- Detecting semantic PII usage
- Understanding whether a new API handles sensitive information
- Identifying intent behind logging or external service calls
- Producing high-level semantic signals

Example:

```json
{
  "signals": [
    {
      "type": "PII_LOGGING",
      "data_type": "CUSTOMER_EMAIL",
      "file": "UserService.java",
      "line": 142,
      "confidence": 0.97
    }
  ]
}
```

The LLM is **not** the final policy decision-maker.

---

# 5. Hybrid Policy Discovery

This is one of the most important parts of the architecture.

A large firm may have hundreds or thousands of policies. Sending every policy to the LLM or Rule Engine for every PR is inefficient.

The system therefore performs **policy discovery before enforcement**.

The discovery layer combines:

1. Structured filtering
2. Semantic RAG/vector search

---

## 5.1 Structured Filtering

Structured signals can narrow the policy search space.

Examples:

```text
Domain       = Data Privacy
Language     = Java
Artifact     = Application Code
Environment  = Production
Signal       = PII_LOGGING
```

This can eliminate policies that clearly cannot apply.

---

## 5.2 RAG / Vector Store

Policy documents are chunked and embedded into a vector store.

Each chunk contains metadata such as:

```json
{
  "document_id": "POL-DATA-003",
  "section": "3.1",
  "domain": "data-privacy",
  "policy_version": "2.1"
}
```

The PR analyzer produces a semantic search query such as:

```text
"Java service logging customer email / customer PII"
```

The vector store returns the most relevant policy document/chunk IDs.

Example:

```text
POL-DATA-003
POL-LOG-002
POL-DATA-007
```

### Important distinction

The vector store answers:

> **"Which policy documents appear relevant?"**

It is not the authoritative source for enforcement.

The authoritative policy information is retrieved from the **Policy Registry** using the returned document/policy IDs.

---

# 6. Policy Registry

The Policy Registry is the authoritative system of record for policies.

It contains information such as:

```text
Policy ID
Name
Description
Domain / category
Applicability signals
Supported languages
Supported artifact types
Severity
Owner
Version
Lifecycle status
Exceptions
Rego reference
Human-readable policy document reference
```

A policy may look conceptually like:

```json
{
  "policy_id": "DATA-003",
  "version": "2.1",
  "name": "No Customer PII in Application Logs",
  "domain": "data-privacy",
  "severity": "HIGH",
  "signals": [
    "PII_LOGGING"
  ],
  "status": "ACTIVE",
  "rego_policy": "data.pac.data_003"
}
```

The registry allows the platform to separate:

- Policy discovery
- Policy metadata
- Policy lifecycle
- Human-readable documentation
- Executable policy
- Exceptions
- Ownership
- Versioning

---

# 7. MCP Policy Server

The Policy Registry is exposed to AI agents through an MCP server.

MCP provides a standardized interface for AI systems to interact with policy capabilities.

Possible MCP tools include:

```text
search_policies()
get_policy()
get_policy_version()
get_policy_metadata()
get_policy_guidance()
get_policy_exceptions()
find_applicable_policies()
```

The important separation is:

```text
Policy Registry = source of truth
MCP             = AI-facing interface to policy capabilities
Vector DB       = semantic retrieval mechanism
```

The PR evaluation pipeline may use the registry directly through a service API where deterministic behavior is preferred, while AI agents can use MCP when they need to discover or retrieve policy information.

---

# 8. Applicability Determination

RAG retrieval is probabilistic, so a high vector similarity score should not automatically mean that a policy definitely applies.

The architecture therefore distinguishes between:

```text
Candidate policies
        ↓
Relevant policies
        ↓
Applicable policies
```

For example:

```text
Vector Search
     ↓
Top 20 candidate policy IDs
     ↓
Policy Registry metadata filtering
     ↓
Applicability conditions
     ↓
5 applicable policies
```

Applicability can consider:

- Code signals
- Programming language
- Artifact type
- Environment
- Repository metadata
- Changed files
- Policy domain
- Policy status
- Exceptions

This keeps the Rule Engine focused on enforcing policies that are actually relevant to the PR.

---

# 9. Policy Input Builder

The Input Builder converts all analysis results into a canonical input contract for the Rule Engine / OPA.

The input should be stable even if the internal analyzers change.

Example:

```json
{
  "context": {
    "repository": "customer-service",
    "branch": "feature/add-customer-profile",
    "language": "java",
    "environment": "production"
  },
  "changes": {
    "files": [
      "src/main/java/com/example/UserService.java"
    ]
  },
  "signals": [
    {
      "type": "PII_LOGGING",
      "data_type": "CUSTOMER_EMAIL",
      "file": "UserService.java",
      "line": 142,
      "confidence": 0.97
    }
  ],
  "applicable_policies": [
    {
      "policy_id": "DATA-003",
      "version": "2.1"
    }
  ]
}
```

The canonical input schema gives the Rule Engine a stable contract.

---

# 10. Rule Engine / OPA

The Rule Engine / OPA is the deterministic enforcement layer.

Its job is to answer:

> **Given these facts and applicable policies, is the PR compliant?**

The Rule Engine should not be responsible for discovering policies through semantic search.

It evaluates structured input against executable policy rules.

Responsibilities include:

- Loading policy rules
- Evaluating policy conditions
- Checking exceptions
- Producing allow / warn / deny / review decisions
- Producing structured violations
- Returning evidence for each decision

Example result:

```json
{
  "decision": "DENY",
  "violations": [
    {
      "policy_id": "DATA-003",
      "version": "2.1",
      "severity": "HIGH",
      "rule": "PII must not be written to application logs",
      "file": "UserService.java",
      "line": 142,
      "evidence": {
        "data_type": "CUSTOMER_EMAIL",
        "operation": "LOG"
      }
    }
  ],
  "warnings": [],
  "passed_policies": [],
  "policy_version": "2026.08"
}
```

### Why use a Rule Engine / OPA?

The final compliance decision should be deterministic and auditable.

The LLM can interpret code and policy language, but it should not be trusted to make the authoritative compliance decision.

This gives us:

```text
LLM       -> semantic understanding
RAG       -> policy discovery / retrieval
Rule Engine / OPA -> deterministic enforcement
LLM       -> explanation and remediation
```

---

# 11. Policy Repository and OPA Bundles

Executable policies should be versioned independently from the PR request.

A policy repository can contain:

```text
policies/
├── data/
│   ├── data_003.rego
│   └── data_007.rego
├── dependencies/
│   ├── dep_001.rego
│   └── dep_004.rego
├── security/
│   ├── sec_001.rego
│   └── sec_002.rego
└── logging/
    └── log_002.rego
```

A policy CI/CD pipeline validates and packages policies into an OPA bundle.

Conceptually:

```text
Policy Repository
       ↓
Policy Tests / Validation
       ↓
Versioned OPA Bundle
       ↓
Rule Engine / OPA
```

The PR request therefore does not need to send the entire Rego source for every evaluation.

---

# 12. AI Explanation & Remediation Agent

Once the Rule Engine / OPA produces the authoritative evaluation result, the AI Agent turns the structured result into developer-friendly feedback.

The agent can use:

- Evaluation result
- Violations and evidence
- Relevant policy sections
- RAG retrieval of policy guidance
- Relevant code context
- Repository conventions

It can generate:

- Human-readable explanations
- Why the policy was violated
- Exact remediation steps
- Code examples
- Links to the relevant policy
- A concise PR summary

The AI Agent **does not override the Rule Engine decision**.

---

# 13. PR Feedback

The final result is returned to the developer through the Git provider.

Possible outputs:

- PR comment
- Status check
- Inline review comments
- Violation summary
- Severity
- Suggested fix
- Policy reference
- Documentation link

Example:

```text
❌ DATA-003 — Customer PII must not be logged

UserService.java:142 logs the customer's email address.

Why this is a violation:
Customer email is classified as PII and DATA-003 prohibits
customer PII from being written to application logs.

Suggested fix:
Replace the email with a non-sensitive identifier such as customerId.

Example:
logger.info("Processing customer {}", customerId);

Policy: DATA-003 v2.1
```

---

# 14. Infrastructure

The platform can run on Kubernetes and use the following infrastructure components.

| Component | Purpose |
|---|---|
| Kubernetes | Container orchestration |
| PostgreSQL | Policy registry and application metadata |
| Vector DB / pgvector | Policy document embeddings and semantic retrieval |
| Object Storage | Policy documents, analysis artifacts and large files |
| Redis | Caching frequently accessed policy/data information |
| RabbitMQ / Kafka / SQS | Asynchronous analysis jobs and events |
| Secrets Manager | API keys, database credentials and service secrets |
| Prometheus / Grafana | Metrics and dashboards |
| ELK / centralized logging | Logs and troubleshooting |
| OpenTelemetry | Distributed tracing and telemetry |
| Git | Policy source, Rego and policy versioning |

A PostgreSQL + pgvector implementation can be a practical starting point because it can store both policy metadata and embeddings without introducing another database immediately.

---

# 15. Data Stores

## Policy Registry Database

Stores authoritative structured policy metadata:

```text
policy_id
version
name
domain
severity
applicability
status
owner
exceptions
rego_reference
```

## Vector Store

Stores embeddings of policy documents and guidance:

```text
document_id
chunk_id
embedding
section
policy_id
version
domain
```

The vector store is optimized for semantic retrieval, not authoritative policy lifecycle management.

## Policy Repository

Stores version-controlled executable policies and tests.

## Object Storage

Useful for large policy documents, reports, analysis artifacts and generated evidence.

---

# 16. End-to-End Example

Consider a developer working on a Java customer-service application.

The developer adds the following code:

```java
public void processCustomer(Customer customer) {
    logger.info("Processing customer email: {}", customer.getEmail());
    // ...
}
```

The firm's policy says:

> Customer PII must not be written to application logs.

Assume this policy is registered as `DATA-003`.

---

## Step 1 — Developer raises a PR

The developer opens:

```text
PR #142
feature/customer-processing
        ↓
main
```

The Git provider sends a PR webhook.

```text
Git Provider
    ↓
PR Collector
```

---

## Step 2 — PR Collector retrieves the PR

The collector retrieves:

```json
{
  "repository": "customer-service",
  "author": "developer1",
  "base": "main",
  "head": "feature/customer-processing",
  "changed_files": [
    "UserService.java"
  ]
}
```

It starts an asynchronous analysis job.

---

## Step 3 — Deterministic analysis runs

The static analyzer detects:

```text
Language: Java
File: UserService.java
Logging call added: yes
Method: processCustomer()
Method call: customer.getEmail()
```

It creates preliminary facts:

```json
{
  "language": "java",
  "logging_changed": true,
  "customer_email_reference": true
}
```

---

## Step 4 — LLM semantic analysis runs

The LLM receives the relevant code context and determines that:

```text
customer.getEmail()
```

represents customer PII being passed to a logging operation.

It produces:

```json
{
  "signals": [
    {
      "type": "PII_LOGGING",
      "data_type": "CUSTOMER_EMAIL",
      "file": "UserService.java",
      "line": 142,
      "confidence": 0.97
    }
  ]
}
```

The LLM has identified a **signal**, not made the compliance decision.

---

## Step 5 — Build policy discovery query

The Query Builder converts the analyzer output into search context:

```text
Java application
customer data
customer email
logging
PII
```

It can also use structured filters:

```text
Domain = Data Privacy / Security
Language = Java
Signal = PII_LOGGING
Artifact = Application Code
```

---

## Step 6 — Hybrid policy search

The system performs both structured filtering and semantic vector search.

The vector store searches embedded policy documents and returns candidate document IDs:

```text
POL-DATA-003
POL-LOG-002
POL-DATA-007
```

These are **candidate policies**, not yet the final enforcement set.

---

## Step 7 — Policy Registry lookup

The candidate IDs are used to retrieve authoritative policy metadata from the Policy Registry.

For `POL-DATA-003`:

```json
{
  "policy_id": "DATA-003",
  "version": "2.1",
  "name": "No Customer PII in Application Logs",
  "domain": "data-privacy",
  "severity": "HIGH",
  "signals": [
    "PII_LOGGING"
  ],
  "status": "ACTIVE"
}
```

The registry is the source of truth for the policy definition and lifecycle.

---

## Step 8 — Applicability is determined

The system checks the policy metadata and applicability requirements.

The PR contains:

```text
Signal = PII_LOGGING
Language = Java
Artifact = Application Code
```

`DATA-003` is applicable.

The result becomes:

```text
Applicable policies:

DATA-003 v2.1
LOG-002 v1.4
```

---

## Step 9 — Input Builder creates canonical PRPolicyInput

The Input Builder combines:

```text
PR metadata
+ changed files
+ deterministic analysis
+ LLM signals
+ applicable policies
+ relevant context
```

and produces:

```json
{
  "context": {
    "repository": "customer-service",
    "language": "java",
    "environment": "production"
  },
  "changes": {
    "files": [
      "UserService.java"
    ]
  },
  "signals": [
    {
      "type": "PII_LOGGING",
      "data_type": "CUSTOMER_EMAIL",
      "file": "UserService.java",
      "line": 142,
      "confidence": 0.97
    }
  ],
  "applicable_policies": [
    {
      "policy_id": "DATA-003",
      "version": "2.1"
    }
  ]
}
```

---

## Step 10 — Rule Engine / OPA evaluates the PR

The Rule Engine / OPA already has the versioned executable policies loaded through its policy bundle.

It evaluates the canonical input against `DATA-003`.

Conceptually:

```text
PII_LOGGING
      +
CUSTOMER_EMAIL
      +
Application Logging
      +
DATA-003
      ↓
DENY
```

The engine returns structured output:

```json
{
  "decision": "DENY",
  "violations": [
    {
      "policy_id": "DATA-003",
      "severity": "HIGH",
      "file": "UserService.java",
      "line": 142,
      "reason": "Customer email is written to application logs"
    }
  ]
}
```

This is the authoritative compliance decision.

---

## Step 11 — AI Agent retrieves policy guidance

The AI Agent receives the evaluation result and relevant code context.

It can use RAG again, this time for **explanation/remediation**, to retrieve the relevant human-readable policy sections and implementation guidance.

For example:

```text
DATA-003 §3.1
DATA-003 implementation guidance
Approved logging examples
```

This is a different use of RAG from enforcement.

Earlier:

```text
RAG -> discover candidate policies
```

Now:

```text
RAG -> retrieve relevant policy guidance for explanation
```

---

## Step 12 — Final LLM explanation

The AI Agent generates developer-friendly feedback:

```text
❌ DATA-003 — Customer PII must not be logged

UserService.java:142 logs customer.getEmail().

The email address is classified as customer PII and DATA-003
prohibits customer PII from being written to application logs.

Suggested fix:
Log the customer identifier instead of the email address.

Before:
logger.info("Processing customer email: {}", customer.getEmail());

After:
logger.info("Processing customer {}", customer.getId());

Policy: DATA-003 v2.1
```

The LLM has made the result understandable and actionable, but it has **not changed the Rule Engine / OPA decision**.

---

## Step 13 — PR feedback

The PR Policy Guard posts the result back to the PR:

```text
PR #142

❌ Policy check failed

High severity: 1
Warnings: 0
Passed: 0

DATA-003
Customer PII must not be logged

UserService.java:142

Suggested remediation available below.
```

The developer can now fix the code without manually searching through the firm's policy library.

---

# 17. Complete Request Flow

The complete flow can be summarized as:

```text
1. Developer raises PR
        ↓
2. Git Provider sends webhook
        ↓
3. PR Collector gets metadata + diff
        ↓
4. Deterministic analyzers extract code facts
        ↓
5. LLM performs semantic code analysis
        ↓
6. Query Builder creates policy search context
        ↓
7. Structured filters + RAG/vector search find candidate policies
        ↓
8. Candidate document IDs are returned
        ↓
9. Policy Registry provides authoritative policy metadata
        ↓
10. Applicability determination selects applicable policies
        ↓
11. Input Builder creates canonical PRPolicyInput JSON
        ↓
12. Rule Engine / OPA evaluates the input against executable policies
        ↓
13. Rule Engine / OPA returns decision + violations + evidence
        ↓
14. AI Agent retrieves relevant policy guidance using RAG
        ↓
15. LLM explains violations and suggests remediation
        ↓
16. PR comment + status check are posted
```

---

# 18. Why the Architecture Uses Multiple AI / Policy Technologies

Each technology has a specific responsibility.

| Technology | Responsibility |
|---|---|
| LLM | Understand code and policy context semantically |
| Static analysis | Extract deterministic code facts |
| RAG | Find semantically relevant policy documents/sections |
| Vector DB | Store policy embeddings for semantic retrieval |
| Policy Registry | Authoritative policy metadata, versions, lifecycle and references |
| MCP | Standardized AI-facing interface to policy capabilities |
| Rule Engine / OPA | Deterministic policy evaluation and enforcement |
| AI Agent | Explain decisions and suggest remediation |
| Git Provider | PR workflow and developer feedback |

This separation prevents any single component from doing a job it is not designed for.

---

# 19. Key Design Principles

### 1. AI should not make the authoritative compliance decision

LLMs can be probabilistic. The final enforcement decision should come from deterministic policy rules.

### 2. RAG should discover, not enforce

Vector similarity is useful for finding relevant policy information, but policy applicability and enforcement should use structured metadata and deterministic rules.

### 3. The Policy Registry is the source of truth

The vector database should not become the authoritative policy database.

### 4. Policy and application code should be decoupled

Changing a policy should not require changing the PR Guard application itself.

### 5. Use deterministic analysis wherever possible

Do not spend LLM tokens on information that can be extracted reliably using ASTs, dependency scanners, Git diffs and static analysis.

### 6. AI should improve the developer experience

The final output should not simply say `DENY`. It should tell the developer:

- What happened
- Which policy was violated
- Why it was violated
- Where it happened
- How to fix it
- What code to write instead

### 7. Every decision should be explainable and auditable

The system should be able to trace:

```text
PR
 → analyzer finding
 → policy candidate
 → policy version
 → applicability decision
 → Rule Engine decision
 → violation evidence
 → AI explanation
```

This traceability is critical for enterprise governance.

---

# 20. Future Extensions

Potential future capabilities include:

- AI-assisted policy authoring
- Natural-language policy → executable Rego generation
- Automatic policy test generation
- Policy conflict detection
- Policy impact analysis
- Policy exception workflows
- Organization/team-specific policies
- Policy version comparison
- Developer-specific remediation suggestions
- Automatic fix PRs
- Policy effectiveness metrics
- False-positive feedback loops
- Human approval workflows for high-risk policies
- Multi-repository and monorepo support

The long-term vision is to evolve PR Policy Guard from a PR checker into an **AI-assisted enterprise Policy-as-Code platform** covering the full lifecycle:

```text
Policy Creation
      ↓
Policy Registration
      ↓
Policy Discovery
      ↓
Code Understanding
      ↓
Policy Applicability
      ↓
Deterministic Enforcement
      ↓
AI Explanation
      ↓
Developer Remediation
      ↓
Continuous Policy Improvement
```

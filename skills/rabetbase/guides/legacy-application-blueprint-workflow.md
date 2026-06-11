# Legacy Application Blueprint Workflow

Use this workflow when modernizing a legacy application into the Lovrabet architecture, especially when the developer has limited project knowledge, the handoff is incomplete, or the application was received from an external vendor.

The goal is not to rewrite code immediately. The goal is to produce a reviewable **Lovrabet Application Blueprint** that connects legacy code behavior with Lovrabet datasets, dataset relations, Instant API, Custom SQL, Backend Functions, Hooks, Common Functions, and optional Enterprise Java Services.

## Core Principle

Do not ask the developer to explain the legacy system first. In many real projects, the developer is also new to the codebase.

Instead:

1. Collect evidence from code, SQL, configuration, tests, documentation, and git history.
2. Collect platform facts from `rabetbase dataset detail`, `dataset relations`, `db tables`, `db diff`, `sql list/detail`, and `bff list/detail`.
3. Bind legacy code artifacts to Lovrabet datasets and relations.
4. Generate an Application Blueprint as Markdown.
5. Ask humans only for high-impact uncertainties that cannot be resolved from evidence.

## Primary Output

The primary output is Markdown:

```text
.rabetbase/blueprint/<appCode>/application-blueprint.md
```

For larger projects, split the output into:

```text
.rabetbase/blueprint/<appCode>/
  application-blueprint.md
  migration-backlog.md
  manual-confirmation.md
  application-blueprint.json
```

If the analysis covers multiple legacy repositories for one Lovrabet app, add a source name:

```text
.rabetbase/blueprint/<appCode>/<sourceName>/
  application-blueprint.md
  migration-backlog.md
  manual-confirmation.md
  application-blueprint.json
```

Markdown is the source of review and handoff. JSON is optional and only for later automation by agents or CLI commands.

## Why This Is Not Just Code Mapping

Generic code analysis tools can inspect routes, services, repositories, and SQL, but they do not know the Lovrabet data model.

Lovrabet analysis must combine:

```text
Legacy Logic Graph + Dataset Graph + rabetbase CLI Facts = Application Blueprint
```

Use platform facts to reduce guesswork:

- `rabetbase dataset detail --code <datasetCode> --format compress`
- `rabetbase dataset relations --format compress`
- `rabetbase db tables --format compress`
- `rabetbase db diff --format compress`
- `rabetbase sql list/detail --format compress`
- `rabetbase bff list/detail --format compress`

Do not infer field names, enum values, or relationships from legacy naming alone when Lovrabet dataset metadata is available.

## Blueprint Entity Model

### 1. Codebase Snapshot

The analysis baseline.

Record:

- Repository or local path
- Branch and commit
- Analysis time
- Scanned directories
- Excluded or inaccessible directories
- Detected frameworks, build tools, ORM, SQL style, jobs, queues, and integrations
- Known limitations of this analysis pass

### 2. Logic Graph

The legacy code behavior graph.

Node types:

- `EntryPoint`: route, controller, handler, job, consumer, webhook, CLI
- `BusinessAction`: create order, submit approval, import customer, cancel payment
- `ServiceMethod`: legacy service or domain method
- `RepositoryMethod`: mapper, DAO, repository method
- `SqlStatement`: SQL, XML mapper, ORM query
- `StateTransition`: status fields and transitions
- `PermissionRule`: auth, role, tenant, department, owner filtering
- `SideEffect`: external API, MQ, cache, audit log, file, message
- `SharedFunction`: validators, calculators, ID generators, converters
- `ExternalDependency`: third-party platform, gateway, retained worker

### 3. Dataset Binding

The Lovrabet-specific bridge between legacy logic and platform truth.

It answers:

- Which legacy table maps to which Dataset?
- Which legacy field maps to which Dataset field?
- Which legacy enum or status field maps to Dataset options?
- Which legacy SQL joins map to Dataset Relations?
- Which repository writes affect which Datasets?
- Which business action affects which Dataset group?
- Which Dataset or Relation is missing and should become a model adjustment?

### 4. Capability Mapping

The mapping from legacy behavior to Lovrabet capability:

- Instant API
- Custom SQL
- Backend Function
- Hooks Function
- Common Function
- Enterprise Java Service
- External Retained System

### 5. Migration Asset

The reviewable migration inventory:

- Dataset adjustment candidates
- Instant API candidates
- Custom SQL candidates
- Backend Function candidates
- Hook candidates
- Common Function candidates
- Java Service candidates
- External retained systems
- Manual confirmation items

### 6. Implementation Draft

Optional implementation sketches after blueprint review:

- `.rabetbase/sql/...` Custom SQL draft
- `.rabetbase/bff/...` Backend Function draft
- Hook draft
- Common Function draft
- Java Service interface draft
- Validation commands
- Runtime smoke checklist

Do not treat drafts as finished code. Do not push SQL or BFF until the relevant reference docs have been read and dry-run validation has passed.

## Evidence Collection

### Code Evidence

Search for:

- HTTP routes, controllers, handlers, resolvers
- CLI commands, cron jobs, schedulers, MQ consumers, webhooks
- Services, domains, managers, processors, workflows, use cases
- Repositories, DAOs, mappers, entities, models, schemas, migrations
- SQL files, XML mappers, ORM queries
- Middleware, interceptors, guards, filters, permission checks
- HTTP clients, RPC clients, MQ producers, cache, mail, SMS, IM, file storage, audit logs
- Tests, fixtures, seed data, README, API docs, deployment docs
- Git history: high-churn files, files often changed together, hotfix commits, commit messages that introduced major modules

### Platform Evidence

Use Lovrabet platform facts to calibrate code inference:

- `dataset detail`: fields, field types, required flags, enum options, display names
- `dataset relations`: relationship facts, join candidates, parent-child or reference relationships
- `db tables` and `db diff`: physical table facts and Dataset mismatch checks
- `sql list/detail`: existing Custom SQL that should not be duplicated
- `bff list/detail`: existing BFF, Hook, and Common scripts that should not be overwritten

### Evidence Record Format

Every important claim should be recorded in this shape:

```markdown
### Claim: <short conclusion>

- Type: fact | inference
- Confidence: high | medium | low
- Code evidence:
  - `<path>`: <symbol or behavior>
- Platform evidence:
  - Dataset: `<datasetCode>` / `<datasetName>`
  - Field: `<fieldName>` / `<type>` / `<options>`
  - Relation: `<fromDataset> -> <toDataset>`
- Assumption: <only if needed>
- Risk if wrong: <migration impact>
```

## Required Blueprint Sections

### 1. Executive Summary

Include:

- Legacy technology stack
- Key business domains detected
- Dataset and Dataset Relation coverage
- Recommended migration path
- Highest-risk areas
- Number of manual confirmation items
- Whether implementation drafts are included

### 2. Codebase Snapshot

Recommended table:

```markdown
| Item | Value |
| --- | --- |
| Repository | |
| Branch / commit | |
| AppCode | |
| Analysis time | |
| Scanned directories | |
| Excluded directories | |
| Detected stack | |
| Known limitations | |
```

### 3. Logic Graph Summary

Recommended table:

```markdown
| Node | Type | Evidence | Downstream | Confidence |
| --- | --- | --- | --- | --- |
```

For key business actions, include the call chain:

```markdown
### Business Action: <name>

- Entry point:
- Main chain:
- Data reads:
- Data writes:
- State transitions:
- Permission checks:
- Side effects:
- Evidence:
- Confidence:
```

### 4. Dataset Binding Report

Recommended table:

```markdown
| Legacy artifact | Code evidence | Dataset | Field mapping | Relations | Confidence | Risk |
| --- | --- | --- | --- | --- | --- | --- |
```

Rules:

- Name similarity alone is weak evidence.
- Dataset field type and enum values come from `dataset detail`.
- Relationships come from `dataset relations` when available.
- Missing relations should become Dataset adjustment candidates or manual confirmation items.
- If the legacy model conflicts with Dataset metadata, do not generate implementation drafts yet.

### 5. Lovrabet Capability Mapping

Recommended table:

```markdown
| Business action / query | Code evidence | Dataset facts | Recommended capability | Why | Not recommended | Risk | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
```

Default to Instant API only when the action is simple single-dataset CRUD with clear permissions and no significant transaction, state machine, or side effect.

### 6. Migration Backlog

Create a staged backlog ordered by risk:

1. Dataset adjustments
2. Instant API replacements
3. Custom SQL
4. Backend Functions
5. Hooks Functions
6. Common Functions
7. Enterprise Java Services
8. External retained integrations

Recommended table:

```markdown
| Stage | Task | Goal | Evidence | Lovrabet capability | Dependencies | Acceptance criteria | Risk | Confidence |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
```

### 7. Manual Confirmation Items

Only ask humans to confirm high-impact uncertainties that cannot be resolved from code or platform facts.

Use evidence-based options, not open-ended questions:

```markdown
### Question: Should order cancellation succeed when refund fails?

- Impact: Defines transaction behavior for the future Backend Function.
- Code evidence:
  - `src/services/orderService.ts` updates order status before refund.
  - `src/integrations/payment.ts` catches refund errors and logs them.
  - No retry queue or compensation job was found.
- Dataset / Relations evidence:
  - `order_xxx` contains `orderStatus`.
  - `refund_log_xxx` is linked to `order_xxx`.
- Options:
  1. Order cancellation succeeds and refund is compensated asynchronously.
  2. Refund failure makes order cancellation fail.
  3. Current behavior is a legacy bug and should be corrected during migration.
- AI default: Option 1.
- Risk if unresolved: The migration may change transaction semantics.
```

### 8. Implementation Draft

Include this section only after the blueprint is reviewed or when the user explicitly asks for implementation planning.

Recommended subsections:

- Custom SQL drafts
- Backend Function drafts
- Hook drafts
- Common Function drafts
- Enterprise Java Service interface drafts
- Validation commands
- Runtime smoke checks

## Capability Decision Matrix

| Question | If yes | Recommended capability |
| --- | --- | --- |
| Is it simple single-Dataset CRUD? | Yes | Instant API |
| Is it simple filtering, pagination, or sorting? | Yes | Instant API |
| Does it need joins, aggregation, reports, dashboards, or exports? | Yes | Custom SQL |
| Does it write multiple Datasets together? | Yes | Backend Function |
| Does it require a transaction boundary? | Yes | Backend Function or Java Service |
| Does it contain state transitions? | Yes | Backend Function or Hook |
| Is it automatically triggered by Dataset create/update/delete? | Yes | Hooks Function |
| Does it call third-party systems or use server-side secrets? | Yes | Backend Function or external worker |
| Is the logic reused by multiple BFF/Hook scripts? | Yes | Common Function |
| Is it heavy domain logic or existing Java service logic? | Yes | Enterprise Java Service |
| Can it only run in a retained legacy system? | Yes | External Retained System |

Do not map these directly to Instant API:

- An endpoint that looks like `updateStatus` but performs approval, audit logging, notifications, and rollback.
- A list endpoint with complex role, organization, tenant, or owner filtering.
- A save endpoint that writes child tables, audit logs, and external systems.
- A delete endpoint that performs soft delete, cascade, inventory rollback, and messaging.
- A bulk import endpoint with de-duplication, validation, transaction handling, and compensation.

## Recommended Agent Flow

1. Resolve app context with `.rabetbase.json`, `rabetbase app list`, or explicit `--appcode`.
2. Collect Dataset facts with `dataset detail` and `dataset relations`.
3. Check existing platform assets with `sql list/detail` and `bff list/detail` when relevant.
4. Inspect legacy code and build the Logic Graph.
5. Bind legacy code artifacts to Datasets and Relations.
6. Produce `.rabetbase/blueprint/<appCode>/application-blueprint.md`.
7. Split backlog and manual confirmation items if the file becomes too large.
8. Wait for review before generating implementation drafts.

## Prompt Blocks

### Full Blueprint Prompt

```markdown
You are a Staff Engineer modernizing an unowned legacy application into the Lovrabet architecture.

Do not rely on the developer to provide business context. Recover the system behavior from code, SQL, configuration, tests, documentation, git history, and Lovrabet platform facts.

Produce a Lovrabet Application Blueprint. Do not modify code.

Required output:
1. Executive Summary
2. Codebase Snapshot
3. Logic Graph Summary
4. Dataset Binding Report
5. Lovrabet Capability Mapping
6. Migration Backlog
7. Manual Confirmation Items
8. Optional Implementation Draft

Lovrabet capabilities:
- Dataset
- Instant API
- Custom SQL
- Backend Function
- Hooks Function
- Common Function
- Enterprise Java Service
- External Retained System

Rules:
- Every important conclusion must include code evidence.
- Every Dataset-related conclusion must include platform evidence from dataset detail, dataset relations, db tables/diff, or existing SQL/BFF facts when available.
- Mark confidence as high, medium, or low.
- Ask humans only for high-impact uncertainties that cannot be resolved from evidence.
- Do not map all legacy APIs to Instant API. Upgrade to SQL, BFF, Hook, Common Function, or Java Service when transaction, state machine, permission, side effect, or external integration complexity requires it.
- Write the primary output to `.rabetbase/blueprint/<appCode>/application-blueprint.md`.
```

### Dataset Binding Prompt

```markdown
Generate a Dataset Binding report by connecting legacy entities, tables, repository methods, and SQL statements to Lovrabet Datasets and Dataset Relations.

Use:
- `rabetbase dataset detail --code <datasetCode> --format compress`
- `rabetbase dataset relations --format compress`
- `rabetbase db tables --format compress`
- `rabetbase db diff --format compress`

Output:
- Legacy artifact
- Code evidence
- Matched Dataset
- Field mapping
- Enum/status mapping
- Dataset Relations
- Missing or conflicting model facts
- Dataset adjustment candidates
- Confidence
- Risk

Do not use name similarity alone as high-confidence evidence.
```

### Backlog Prompt

```markdown
Based on the Application Blueprint, generate a staged Lovrabet migration backlog.

Stages:
1. Dataset adjustments
2. Instant API replacements
3. Custom SQL
4. Backend Functions
5. Hooks Functions
6. Common Functions
7. Enterprise Java Services
8. External retained integrations

Each task must include:
- Goal
- Code evidence
- Dataset / Relations evidence
- Recommended Lovrabet capability
- Dependencies
- Acceptance criteria
- Risk
- Confidence
- Manual confirmation requirement

Do not propose implementation before unresolved high-risk confirmation items are listed.
```

## Validation Gates

Before claiming the blueprint is complete, verify:

- The output file path is under `.rabetbase/blueprint/<appCode>/`.
- The blueprint has an Executive Summary.
- It has both code evidence and platform evidence for key conclusions.
- Dataset Binding is present.
- Capability Mapping does not blindly map all endpoints to Instant API.
- Manual Confirmation Items are evidence-based and option-based.
- Implementation drafts, if included, are clearly marked as drafts.
- No SQL or BFF was pushed as part of blueprint generation.

## Common Mistakes

- Asking the developer to explain the legacy system before doing code and platform evidence collection.
- Treating the blueprint as complete without Dataset Binding.
- Mapping every legacy CRUD-like endpoint to Instant API without checking state machines and side effects.
- Guessing Dataset field names or enum values from legacy names.
- Writing BFF or SQL directly into arbitrary project folders instead of `.rabetbase/bff` or `.rabetbase/sql`.
- Mixing blueprint artifacts with pushable implementation assets. Blueprint files belong under `.rabetbase/blueprint`, while SQL and BFF sources belong under their existing `.rabetbase/sql` and `.rabetbase/bff` directories.

# Persist Template Context

## Overview

Procedure for embedding architectural intent and design rationale into CloudFormation templates so that future sessions (human or AI) can understand WHY the stack exists and WHY each resource is configured the way it is.

Uses the `Metadata.Context` schema when no other convention exists:

- **Template Description** (1,024 bytes max): One-sentence summary of the stack's purpose and key design decision — the native CloudFormation Description field captures stack purpose.
- **Template-level Metadata.Context** (optional): Cross-cutting context that applies broadly, stated ONCE (DRY) rather than repeated per resource — `arch` (system shape), `must` (cross-cutting constraints, array), `ref` (pointers to external context files, template level only), `owner` (contact). This block MUST NOT include `v` (versioning is global/implicit) or `sys` (stack purpose lives in the native Description).
- **Resource-level Metadata.Context**: Per-resource rationale — `why` (purpose + notable choices + rejected alternatives), `must` (hard constraints/invariants, array), `mutable` (resource-level DEFAULT change-safety, one token: `must-never-change|change-with-constraints|review-required|free-to-tune`), `mutability` (OPTIONAL sparse override map — keys = CFN property names, only properties that DEVIATE from the `mutable` default, same enum), `trust`, `ops`, `gaps`, `deps`.

**Decision rule:** Will violating it break something? → `must`. Otherwise → `why`. There is no separate decisions/constraints split.

**Caveman shorthand:** Use short keys, telegraphic values (symbols like `>=`, `->`, `x`, `&`), abbreviations (`fn`, `msg`, `dup`, `cfg`). Never restate the resource Type, logical id, property values, or the resource's `Description` property.

**Tiers:** Always emit T1 (`why` + `must` on significant resources; Description for stack purpose). Add T2 (`mutable`, `arch` in `why`) if budget allows. Add T3 (`trust`, `ops`, `gaps`, `deps`) when warranted. If the template nears 1 MB, shed in order: `trust` → `ops` → `gaps` → `deps` → `mutable` on non-critical → trim `why` to significant resources → last resort externalize via `ref`. NEVER drop `must` on coupled/security/stateful resources. Measure the current template body in bytes (`wc -c <template>` on Unix/macOS or Git Bash, or `(Get-Item <template>).Length` in PowerShell) and count resources before deciding whether to shed — compare against the 1,048,576-byte S3 limit (51,200 inline) and the 500-resource cap. See SKILL.md **Template Size Limits** for the full condense/relocate strategy.

**Match the existing documentation convention.** `Metadata.Context` is the default mechanism and the right choice when neither the template nor its project already has a decent context convention — this includes all JSON templates (JSON has no comments) and YAML templates without meaningful comments. Before injecting `Metadata.Context`, check what convention is already in use and follow it:

- **Natural inline comments** — if a YAML template documents intent well through inline comments, extend the author's comments in their own style and voice.
- **Companion documentation** — if the repo, package, or workspace records design context in companion docs (README, a `docs/` folder, architecture notes, or architecture decision records (ADRs)), add or update the new/changed context there following that convention, and add a template-level `ref` entry pointing to the file(s) so the link is discoverable from the template.

Do NOT mix systems on one template — match what is already there. Whichever you use, keep safety-critical `must` constraints discoverable and never externalize the irreducible core. When there is no existing convention, use `Metadata.Context`.

## Steps

### 1. Write the Template Description

Constraints:

- You MUST set the top-level `Description` field to a concise summary of: what the stack does + the primary design decision or constraint that shaped it.
- You MUST keep it under 1,024 bytes (UTF-8). This is enforced by CloudFormation.
- You MUST NOT put operational details (account IDs, regions) in Description — those change per deployment.
- Format: `<what it does> — <why it's designed this way>`
- Example: `Real-time order processing pipeline — uses SQS FIFO over EventBridge for strict ordering guarantee per customer-id`

### 2. Stack Purpose and Cross-Cutting Context

Constraints:

- You MUST ensure the top-level `Description` field captures the stack purpose (what it is + why). Stack purpose lives in the native CloudFormation Description (CDK: Stack description prop), NOT in a template-level `Metadata.Context` block. If Description already exists and is correct, do not overwrite it.
- You MAY add a template-level `Metadata.Context` block for cross-cutting context that applies broadly and would otherwise be repeated on many resources: `arch` (system shape), `must` (cross-cutting constraints, e.g. "all data encrypted w/ security-team CMK"), `ref` (pointers to external context files), `owner` (contact). State such context ONCE here (DRY) rather than duplicating it per resource.
- The template-level block MUST NOT include `v` or `sys` — versioning is global/implicit and stack purpose belongs in the Description. If the block already exists, UPDATE it (preserve valid entries, no duplicate array items) rather than replacing it wholesale.

### 3. Write Resource-Level Metadata Context

Constraints:

- FIRST apply the **Match the existing documentation convention** rule above: if this template or its project already documents intent well (YAML inline comments, or companion docs in the repo/package/workspace), record the new or changed resource's intent in that same convention and SKIP the `Metadata.Context` block for it — when the context lives in companion docs, add a template-level `ref` pointing to the file(s). Use the `Metadata.Context` steps below when the template is JSON, or when no existing convention is present.
- For EACH significant resource (stateful, security, coupled, or non-obvious), you MUST ENSURE a `Metadata.Context` key exists. If one already exists, UPDATE it — preserve existing `must` constraints and `mutable` flags that remain valid; do not duplicate array entries. Only ADD new fields or CORRECT stale ones.
- The `Context` key MUST contain at minimum (T1):
  - `why`: Purpose + notable config choices + rejected alternatives. The SINGLE explanatory field. Non-binding. Never restate Type, logical id, property values, or Description.
  - `must`: Hard constraints/invariants (array of strings). Only when a real rule exists — never invent. Decision rule: *will violating it break something? → `must`. Otherwise → `why`.*
- You SHOULD add T2 when budget allows:
  - `mutable`: Resource-level DEFAULT change-safety. One token per resource: `must-never-change` | `change-with-constraints` | `review-required` | `free-to-tune`.
  - `mutability`: OPTIONAL sparse override map. Keys = CFN property names that DEVIATE from the `mutable` default. Values use the same enum. Omit properties that match the default.
- You MAY add T3 when warranted:
  - `trust`: `{ src: comment|authored|commit|infer, conf: high|medium|low, cite?: "file:line", note?: <reason for low confidence> }`
  - `ops`: Operational hint before changing (what to check pre-modification)
  - `gaps`: Explicit unknowns (array) — honest beats fabricated
  - `deps`: Cross-stack producers (array)
- You SHOULD omit `Context` on trivial resources where the Type and logical name make the purpose obvious (e.g., a WaitConditionHandle).
- You MUST NOT put secrets, PII, or credentials in Metadata — it is stored unencrypted and returned via API.
- You MUST NOT use the `AWS::CloudFormation::Init` key for context — that key is reserved for cfn-init.
- You SHOULD use caveman shorthand: telegraphic values, symbols (`>=`, `->`, `x`, `&`), abbreviations (`fn`, `msg`, `dup`, `cfg`).
- You MUST NOT create duplicate entries in `must` arrays. Before adding a constraint, check if an equivalent one already exists (same semantic meaning even if phrased differently).
- When re-running persist after modifying one resource, you MUST leave other resources' Context untouched unless their context is factually wrong.

### 4. Verify Context Completeness

Constraints:

- You MUST verify that someone reading ONLY the Description plus the template's embedded context (Metadata.Context blocks or the inline comments, whichever this template uses), without the original conversation, could understand:
  1. What problem the stack solves (Description)
  2. Why each significant resource exists and its key choices (`why`)
  3. What invariants must hold to keep things working (`must`)
- If any of these are unclear, you MUST add more context before proceeding.
- You MUST verify the top-level Description is present and captures stack purpose.

## Examples

### Example: Annotated Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: >-
  Real-time order processing pipeline — uses SQS FIFO over EventBridge
  for strict ordering guarantee per customer-id

Resources:
  OrderQueue:
    Type: AWS::SQS::Queue
    Metadata:
      Context:
        why: buffer order events async; FIFO for per-customer ordering (prevent inventory oversell); FIFO over Kinesis (no shard mgmt needed at 10K msg/sec)
        must:
          - VisTimeout >= 5x fn timeout, else dup on retry
          - DLQ maxReceive = 3; don't lose msgs
        mutable: change-with-constraints
        mutability:
          QueueName: must-never-change
        trust: { src: authored, conf: high }
        ops: check ApproxAgeOfOldestMsg before cutting VisTimeout
    Properties:
      FifoQueue: true
      ContentBasedDeduplication: true
      VisibilityTimeout: 300
      KmsMasterKeyId: alias/aws/sqs
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt OrderDLQ.Arn
        maxReceiveCount: 3

  ProcessorFunction:
    Type: AWS::Lambda::Function
    Metadata:
      Context:
        why: processes orders from queue; Lambda over ECS for cost at bursty loads; py3.12 for cold start; 512MB from load test (below -> p99 > 2s SLA)
        must:
          - timeout <= VisTimeout/5
        mutable: change-with-constraints
        mutability:
          MemorySize: review-required
    Properties:
      Runtime: python3.12
      MemorySize: 512
      Timeout: 60
```

## Troubleshooting

### Description exceeds 1,024 bytes

Shorten it. Focus on the single most important design decision. Move details to resource-level Metadata Context.

### Template size grows too large from Metadata

Metadata is included in the template body. If the template exceeds 51KB (inline limit), upload via S3. If approaching 1MB (S3 limit), apply the drop order: shed `trust` → `ops` → `gaps` → `deps` → `mutable` on non-critical → trim `why` to significant resources → last resort externalize via `ref`. Never drop `must` on coupled/security/stateful resources.

### Existing stack has no context

Use the retrieve-template-context SOP to check what's there, then update the template with context. Deploy via change set if and when you apply the changes.

# Retrieve Stack Context

## Overview
Procedure for recovering architectural intent and design rationale from an existing CloudFormation stack. Reads the template Description and resource-level Metadata `Context` keys to reconstruct WHY the stack was built the way it was — enabling informed modifications without re-discovering original design decisions. This SOP reads context conforming to the Metadata.Context v1 schema (`doc/metadata-context-schema.md`).

Use this SOP BEFORE modifying any existing stack to understand the original intent and constraints.

> **StackSets:** This procedure works identically on StackSet-managed stack instances. You can also call `describe-stack-set` to retrieve the template and StackSet-level description directly.

## Parameters
- **stack_name** (required): The CloudFormation stack name or ARN.
- **region** (required): AWS region where the stack is deployed.
- **resource_filter** (optional): Specific logical resource IDs to inspect. If omitted, inspects all resources.

## Steps

### 1. Verify Dependencies
Constraints:
- You MUST check for `call_aws` tool or AWS CLI availability (same as pre-deploy-validation SOP)
- You MUST verify credentials are valid for the target account/region

### 2. Retrieve Stack Description
Constraints:
- You MUST call `aws cloudformation describe-stacks --stack-name <stack_name> --region <region>`
- You MUST extract the `Description` field from the response
- If Description is empty or missing, You MUST note this as "No stack-level context available" and continue to resource inspection
- You MUST present the Description to the user as the high-level intent summary

> **Note:** Description is also available inside the template body returned by `GetTemplate` (Step 3). This call is kept because it confirms the stack exists, returns its current status, and provides the Description without needing to parse the template.

### 3. Retrieve the Template
Constraints:
- You MUST call `aws cloudformation get-template --stack-name <stack_name> --region <region> --template-stage Original`
- You MUST parse the returned template body
- Stack purpose comes from the native template `Description` (retrieved in Step 2). Modern templates do NOT carry a template-level `Metadata.Context` block.
- If a LEGACY template-level `Metadata.Context` block exists (older `v`/`sys`/`arch`/`must`/`ref`/`own` fields), extract and present whatever is readable as legacy context, but treat the `Description` as the authoritative source of stack purpose.
- You MUST NOT expect template-level Context on modern templates — its absence is normal, not an error

### 4. Extract Resource Context
Constraints:
- For each resource in the template (or filtered set), You MUST check for a `Metadata.Context` key
- For each resource WITH a Context key, You MUST extract and present:
  - `why` — purpose, notable choices, rejected alternatives
  - `must` — hard constraints/invariants (array) — these are SAFETY-CRITICAL; flag them prominently
  - `mutable` — resource-level DEFAULT change-safety (one token: `must-never-change|change-with-constraints|review-required|free-to-tune`); `mutability` — OPTIONAL sparse per-property override map (keys = CFN property names that deviate from the default, same enum) — You MUST check these before modifying any property
  - `trust`, `ops`, `gaps`, `deps` — present if available (T3 fields)
- You MUST honor `mutable`/`mutability` flags: `must-never-change` = never alter; `change-with-constraints` = change only if the associated `must` rule is preserved; `review-required` = needs review; `free-to-tune` = safe to tune
- For resources WITHOUT a Context key, You MUST note them as "No context recorded"
- If a `Metadata.Context` block exists but does not conform to v1 (unknown fields, wrong types), you MUST still extract and present whatever is readable. Do NOT reject the entire block because of one malformed field. Note any structural issues in the summary as "⚠️ Non-standard Context: {issue}".

### 4a. Retrieve Service-Derived Context (when needed)

**Service-derived context (deploy history, drift, change-failure, property diffs, actor) is NOT in the template.** It lives in native APIs. Retrieve it separately when you need the WHO/WHEN/HOW dimensions:

Constraints:
- You MUST NOT expect or look for service-derived signals inside the template
- When you need deploy provenance or operational history, You MUST retrieve from native sources:
  - `DescribeStackEvents` — deployment timeline (who deployed, when, what happened)
  - `DetectStackDrift` / `DescribeStackDriftDetectionStatus` — current drift status
  - CloudTrail — actor enrichment (who initiated the API call)
  - Change sets / template-version diffs — property-level changes between versions
- You SHOULD retrieve service-derived context when: troubleshooting failures, assessing risk of a change, understanding recent modifications, or auditing drift

### 5. Follow Cross-Stack References

When a template uses `Fn::ImportValue`, it depends on resources from other stacks. Understanding those upstream stacks provides critical context about shared infrastructure constraints.

Hardcoded resource identifiers (ARNs, physical IDs, account numbers, VPC IDs) indicate dependencies on **unmanaged resources** — infrastructure that exists outside CloudFormation or in a partially IaC-managed environment. These are invisible dependencies that won't show up as `Fn::ImportValue`.

Constraints:
- You MUST scan the template for any `Fn::ImportValue` or `!ImportValue` references
- For each imported value, You MUST identify the producing stack by calling `aws cloudformation list-exports --region <region>` and matching the export name
- For each producing stack that has significant context (i.e., the imported resource is central to the current stack's design), You SHOULD retrieve its Description and the relevant resource's `Metadata.Context` using the same procedure (Steps 2-4)
- You MUST NOT recursively follow more than one level of cross-stack references — report them but do not chase transitive dependencies
- You MUST scan for hardcoded ARNs, resource IDs (e.g., `vpc-*`, `sg-*`, `subnet-*`, `ami-*`), and account numbers in resource properties — these indicate dependencies on resources managed outside this stack
- For hardcoded identifiers, You MUST flag them as **unmanaged dependencies** and warn that deleting or modifying related resources could break external systems that depend on them
- You MUST include cross-stack context in the summary under a **Dependencies** heading with two sub-sections: **Managed** (Fn::ImportValue) and **Unmanaged** (hardcoded identifiers)
- If no `Fn::ImportValue` references or hardcoded identifiers exist, You SHOULD skip this step

### 6. Synthesize Context Summary
Constraints:
- You MUST present a structured summary:
  1. **Stack Purpose** (from `Description`; fall back to legacy `sys` only if present)
  2. **Architecture** (from resource-level context or Description; fall back to legacy `arch` only if present)
  3. **Cross-Cutting Constraints** (only if a legacy template-level `must` is present)
  4. **Resource Rationale** (aggregated `why` from resource-level Context)
  5. **Hard Constraints** (aggregated `must` from resource-level — these are safety-critical)
  6. **Mutability** (resource `mutable` default + any `mutability` overrides — highlight `must-never-change` and `change-with-constraints` properties)
  7. **Dependencies** (from Fn::ImportValue references + `deps` fields — producing stack, what's imported, and its context)
  8. **Resources Without Context** (list of logical IDs with no Context metadata)
- You MUST warn the user about any resources lacking context — these are blind spots for modification
- You MUST prominently flag all `must` constraints — these prevent the agent from silently breaking the system
- You SHOULD recommend running the persist-stack-context SOP to fill gaps before making changes

## Examples

### Example: Full Context Retrieved

```
Stack: order-processing-prod (us-east-1)

## Stack Purpose (from Description)
order-intake event pipeline; decouples API from processing

## Architecture
SQS buffer -> Lambda -> DynamoDB; DLQ for poison msgs

## Cross-Cutting Constraints (legacy template-level must)
- all data encrypted w/ security-team CMK
- p99 latency <= 2s

## Resource Rationale (why)
- OrderQueue: buffer order events async; FIFO for per-customer ordering; FIFO over Kinesis (no shard mgmt at 10K msg/sec)
- ProcessorFunction: processes orders; Lambda over ECS for cost at bursty loads; py3.12 cold start; 512MB from load test

## Hard Constraints (must) ⚠️
- OrderQueue: VisTimeout >= 5x fn timeout, else dup on retry; DLQ maxReceive = 3, don't lose msgs
- ProcessorFunction: timeout <= VisTimeout/5

## Mutability
- OrderQueue.mutable: change-with-constraints
- OrderQueue.QueueName: must-never-change ⚠️
- ProcessorFunction.mutable: change-with-constraints
- ProcessorFunction.MemorySize: review-required

## Resources Without Context
- OrderDLQ (no Metadata.Context)
- LogGroup (no Metadata.Context)
```

### Example: No Context Available

```
Stack: legacy-api-stack (us-west-2)

## Stack Purpose
No Description set.

## Key Design Decisions
None recorded — no resources have Metadata.Context.

## Recommendation
This stack has no embedded context. Before modifying it:
1. Review git history or design docs for original intent
2. Run the persist-stack-context SOP to annotate the template
3. Deploy the annotated template via change set (Metadata-only changes are safe)
```

## Troubleshooting

### get-template returns processed template instead of original
Use `--template-stage Original` to get the template as authored (with Metadata intact). The `Processed` stage may have transforms applied that alter structure.

### Resource Metadata is empty but template has Metadata
This can happen if the stack was created with an older template version. The live metadata matches what was last deployed — check git history for the current template source.

### Stack is in ROLLBACK_COMPLETE state
You can still retrieve the template and metadata from failed stacks. The context is preserved even if deployment failed.

---
inclusion: manual
---

# BlastMap — Blast Radius Audit Prompt

> **Know What Dies Before It Does.**

When asked to run a BlastMap analysis, use the prompt below against the provided
architecture file.

Before writing output files:
1. Run: `git rev-parse --short HEAD` to get the commit hash
2. Write outputs to:
   - `.blastmap/SKILL.md`
   - `.blastmap/<commit-hash>-<original-filename>.yaml`

---

## Prompt

You are an expert AWS Solutions Architect specialising in resilience
engineering and failure mode analysis. Your job is to perform a
BLAST RADIUS AUDIT on the AWS architecture provided below.

A blast radius audit answers one question:
"If any single component fails, what else breaks, and how bad?"

Before writing any output files:
  Run: git rev-parse --short HEAD
  Use the result as <commit-hash> in all output filenames.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 1 — BUILD THE DEPENDENCY GRAPH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Parse the architecture. For every AWS resource output:

[RESOURCE_NAME] (type: AWS service)
  ↑ depends on: [list upstream resources]
  ↓ depended on by: [list downstream resources]
  Dependency type: HARD (immediate failure) | SOFT (degraded)
  Blast radius score: X/10
  Recovery path: [describe fallback or "NONE — SPOF"]

Also identify any external dependencies not declared in the template
(e.g. Amazon Bedrock, third-party APIs, cross-account resources referenced
in environment variables or comments). Flag these as EXTERNAL SPOF with
a blast radius score.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 2 — FLAG SINGLE POINTS OF FAILURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Flag every resource meeting ALL of these criteria:
  ✗ No Multi-AZ or redundant deployment
  ✗ No circuit breaker or retry logic upstream
  ✗ 3 or more downstream dependents
  ✗ No queue or buffer absorbing its failure

Also flag external dependencies (from Step 1) as SPOFs if
the application has no fallback when they are unavailable.

For each SPOF output:
  - Current blast radius (number of services affected)
  - Failure mode: timeout | error cascade | data loss | silent
  - Time to user-visible impact
  - AWS-native fix (exact service + configuration change)
  - Estimated effort to fix (hours/days)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 3 — SIMULATE TOP 3 FAILURE CASCADES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Pick the 3 resources with the highest blast radius score.
Label them Cascade A (highest), Cascade B, Cascade C.
For each, simulate the complete failure cascade:

  T+0s:   [Resource] becomes unavailable
  T+Xs:   [Downstream A] begins timing out
  T+Ys:   [Downstream B] exhausts retry budget, starts failing
  T+Zs:   [Downstream C] queue fills, messages begin dropping
  T+Nm:   User-visible impact: [describe exactly what users see]
  T+Nm:   Estimated MTTR without a runbook: [X minutes]
  T+Nm:   Estimated MTTR with BlastMap SKILL.md: [Y minutes]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 4 — HARDENING PLAN (PRIORITISED)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Output a ranked action list. For each item include:

  Priority: 1=fix this week | 2=this sprint | 3=this quarter
  Change: [what to add or modify]
  AWS service: [exact service + configuration]
  Blast radius reduction: [X/10 → Y/10]
  Effort: [hours or days]
  Cost impact: [±$/month estimate]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 5 — AWS DEVOPS AGENT SKILL.MD OUTPUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Write to: .blastmap/SKILL.md

Follow this exact structure and section order. Replace every [bracketed placeholder]
with real content derived from Steps 1-4 — the final file must contain no brackets,
templates, or instructions, only filled-in data.

---
name: blastmap-[app-name]
description: >
  Pre-computed blast radius intelligence for [app-name]. Use when investigating
  incidents on [name every resource tagged BlastMapSPOF: true from Step 6 — e.g.
  VectorDatabase, RateLimitQueue], or symptoms matching their failure modes
  (timeout, cascade, data loss, silent failure). Provides dependency graph,
  SPOF registry, and cascade paths to accelerate investigation and reduce MTTR.
agent-types: [incident, prevention, sre]
version: <commit-hash>
---

# BlastMap Intelligence: [app-name]

## Resource Risk Registry

| Resource | Type | BlastRadiusScore | SPOFStatus | Top Downstream |
|----------|------|-----------------|------------|----------------|
[fill from Step 1 — include external SPOFs as rows marked EXTERNAL]

## Known Cascade Paths

[Cascade A, B, C — one paragraph each]

## Investigation Shortcuts

When alert fires on [Resource]:
  - Immediately check: [top 3 downstream resources]
  - Likely cascade: [Cascade A / B / C]
  - First mitigation action: [specific step]
  - Escalate if: [condition that indicates wider blast]
  - Expected MTTR: [X minutes with this skill loaded]

[repeat for each SPOF resource]

## Hardening Status

| Change | Priority | AWS Service | Blast Reduction | Status |
|--------|----------|-------------|-----------------|--------|
[fill from Step 4, Status = PENDING for all new items]

## Deployment Notes

- Last analysed: [date]
- Architecture version: [commit hash]
- Next review trigger: any resource addition or change to a SPOF

## Agent Findings — Suggested Improvements

[DevOps Agent: after investigating an incident with this skill, append an entry]
  - Date: [investigation date]
  - Resource: [resource investigated]
  - Predicted vs observed: [did the cascade path, timing, or MTTR match Step 3's prediction?]
  - Suggested change: [new dependency found, score to adjust, new SPOF, etc.]

A user reviews these entries and feeds them into the next BlastMap run to keep
the dependency graph and predictions accurate.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
STEP 6 — ANNOTATED CLOUDFORMATION OUTPUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Re-output the architecture as a CloudFormation template.
Save as: .blastmap/<commit-hash>-<original-filename>.yaml

For EVERY resource, add a 2-field Metadata block:

  Metadata:
    BlastMap:
      BlastRadiusScore: "X/10"
      Fix: "[recommended AWS-native fix, or NONE if not a SPOF]"

For resources that are a SPOF, also add one tag:

  Tags:
    - Key: BlastMapSPOF
      Value: "true"

Resources that are NOT a SPOF get the Metadata block only — no tag.
Tag presence = SPOF. The tag's sole purpose is AWS Config rules
and Resource Groups queries.

Scoring guide:
  9-10 = Entire app fails
  7-8  = Major feature loss
  5-6  = Significant degradation
  3-4  = Minor degradation
  1-2  = Isolated, recoverable

Example — SPOF resource:

  VectorDatabase:
    Type: AWS::RDS::DBInstance
    Metadata:
      BlastMap:
        BlastRadiusScore: "9/10"
        Fix: "Enable MultiAZ: true on AWS::RDS::DBInstance"
    Properties:
      DBInstanceClass: db.r6g.large
      Engine: postgres
      MultiAZ: false
      AllocatedStorage: 500
    Tags:
      - Key: BlastMapSPOF
        Value: "true"

Example — non-SPOF resource:

  DocumentBucket:
    Type: AWS::S3::Bucket
    Metadata:
      BlastMap:
        BlastRadiusScore: "2/10"
        Fix: "NONE"
    Properties:
      BucketName: rag-documents

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ARCHITECTURE TO ANALYSE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Read the CloudFormation template(s) in this repository.
- If the user named a specific file (e.g. "analyse infrastructure/my-stack.yaml"),
  read that file directly.
- Otherwise, locate CloudFormation files by searching for *.yaml / *.yml / *.json
  files containing "AWSTemplateFormatVersion" or a top-level "Resources:" key.
Use that file's contents as the architecture to audit in Steps 1-6 above.

---

## Output Checklist

### Step 1 — Dependency Graph
- [ ] Every resource has an entry
- [ ] External dependencies (Bedrock, third-party APIs) identified and scored
- [ ] Dependency type is HARD or SOFT for each relationship
- [ ] Every resource has a blast radius score 1–10
- [ ] Resources with no fallback are marked NONE — SPOF

### Step 2 — SPOFs
- [ ] At least one SPOF identified (unless architecture is fully redundant)
- [ ] External SPOFs included where no fallback exists
- [ ] Each SPOF has a failure mode
- [ ] Time-to-user-impact is specific
- [ ] AWS fix is concrete — specific service + config

### Step 3 — Cascade Simulations
- [ ] Exactly 3 cascades, labelled Cascade A / B / C
- [ ] Each cascade has timestamped T+ entries
- [ ] MTTR estimated both with and without SKILL.md
- [ ] User-visible impact described from end-user perspective

### Step 4 — Hardening Plan
- [ ] Items are ranked (Priority 1 first)
- [ ] Each item includes blast radius reduction (X/10 → Y/10)
- [ ] Cost impact is estimated
- [ ] At least one Priority 1 item exists if any SPOF found

### Step 5 — SKILL.md
- [ ] Written to .blastmap/SKILL.md
- [ ] Front matter is valid YAML
- [ ] agent-types includes: incident, prevention, sre
- [ ] Resource Risk Registry includes external SPOFs
- [ ] Cascade Paths labelled A / B / C
- [ ] Investigation Shortcuts reference cascade label
- [ ] Architecture version = git commit hash

### Step 6 — Annotated CFN
- [ ] Output filename: .blastmap/<commit-hash>-<original-filename>.yaml
- [ ] Every resource has Metadata: BlastMap: with exactly 2 fields
- [ ] BlastRadiusScore matches Step 1 scores
- [ ] Fix populated for SPOFs, NONE for others
- [ ] BlastMapSPOF: true tag ONLY on SPOF resources
- [ ] Non-SPOF resources have no BlastMap tag
- [ ] Valid CloudFormation YAML

---
description: Run a BlastMap blast radius audit on a CloudFormation file
---

## BlastMap - Blast Radius Audit Prompt

Before writing output files, run: git rev-parse --short HEAD
Use that value as \<commit-hash\> in all output filenames.

Write outputs to:
  .blastmap/SKILL.md
  .blastmap/\<commit-hash\>-\<original-filename\>.yaml

---

## Task
    
    You are an expert AWS Solutions Architect specialising in resilience engineering and
    failure mode analysis. Perform a BLAST RADIUS AUDIT on the architecture provided.

    "If any single component fails, what else breaks, and how bad?"

    STEP 1 - DEPENDENCY GRAPH
    For every resource:
      [NAME] (type)
        ↑ depends on: [upstream]
        ↓ depended on by: [downstream]
        Dependency type: HARD | SOFT
        Blast radius score: X/10
        Recovery path: [fallback or "NONE - SPOF"]
      Also flag external dependencies (third-party APIs) as EXTERNAL SPOF.

    STEP 2 - FLAG SPOFs
    Flag every resource with ALL of:
      ✗ No Multi-AZ or redundancy
      ✗ No circuit breaker upstream
      ✗ 3+ downstream dependents
      ✗ No queue absorbing its failure
      Also flag external dependencies with no application fallback.
      These criteria map to AWS Well-Architected Framework Reliability pillar
      best practices (e.g. REL10: fault isolation, REL11: design for resiliency).
      For each SPOF: blast radius count, failure mode, time to user impact, AWS-native fix, effort.

    STEP 3 - TOP 3 FAILURE CASCADES
    Label: Cascade A (highest score), B, C.
      T+0s:  [Resource] unavailable
      T+Xs:  [Downstream A] times out
      T+Ys:  [Downstream B] fails
      T+Nm:  User-visible impact: [what users see]
      T+Nm:  MTTR without runbook: X min | MTTR with SKILL.md: Y min

    STEP 4 - HARDENING PLAN
    Ranked list. For each:
      Priority: 1=this week | 2=this sprint | 3=this quarter
      Change | AWS service | Blast reduction (X→Y) | Effort | Cost impact

    STEP 5 - SKILL.md
    Write to .blastmap/SKILL.md. Follow this exact structure and section order.
    Replace every [bracketed placeholder] with real content derived from Steps 1-4:
    the final file must contain no brackets, templates, or instructions, only filled-in data:

    ---
    name: blastmap-[app-name]
    description: >
      Pre-computed blast radius intelligence for [app-name]. Use when investigating
      incidents on [name every resource flagged as a SPOF in Step 2, e.g.
      VectorDatabase, RateLimitQueue], or symptoms matching their failure modes
      (timeout, cascade, data loss, silent failure).
    agent-types: [incident, prevention, sre]
    version: \<commit-hash\>
    ---

    ## Resource Risk Registry
    | Resource | Type | BlastRadiusScore | SPOFStatus | Top Downstream |
    [fill - include external SPOFs marked EXTERNAL]

    ## Known Cascade Paths
    [Cascade A, B, C - one paragraph each]

    ## Investigation Shortcuts
    When alert fires on [Resource]:
      - Immediately check: [top 3 downstream]
      - Likely cascade: [A / B / C]
      - First action: [specific step]
      - Escalate if: [condition]
      - MTTR: [X min]
    [repeat for each SPOF]

    ## Hardening Status
    | Change | Priority | AWS Service | Blast Reduction | Status |
    [fill from Step 4, Status = PENDING]

    ## Deployment Notes
    - Last analysed: [date]
    - Architecture version: [commit hash]
    - Next review: any resource addition or SPOF change

    ## Agent Findings - Suggested Improvements
    [DevOps Agent: after investigating an incident with this skill, append an entry]
      - Date: [investigation date]
      - Resource: [resource investigated]
      - Predicted vs observed: [did the cascade path, timing, or MTTR match Step 3's prediction?]
      - Suggested change: [new dependency found, score to adjust, new SPOF, etc.]
    A user reviews these entries and feeds them into the next BlastMap run to keep
    the dependency graph and predictions accurate.

    STEP 6 - ANNOTATED CFN
    Save as: .blastmap/\<commit-hash\>-\<original-filename\>.yaml
    Add to EVERY resource:
      Metadata:
        BlastMap:
          BlastRadiusScore: "X/10"
          Fix: "[AWS-native fix, or NONE]"
    Add to SPOF resources only:
      Tags:
        - Key: BlastMapSPOF
          Value: "true"
    Non-SPOF resources get Metadata only - no tag.

    ARCHITECTURE TO ANALYSE:
    Analyse: $ARGUMENTS
    - If a file was named, read that file directly.
    - Otherwise, locate CloudFormation files by searching for *.yaml / *.yml / *.json
      files containing "AWSTemplateFormatVersion" or a top-level "Resources:" key.
    Use that file's contents as the architecture to audit in Steps 1-6 above.
    

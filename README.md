# BlastMap
> **Know What Dies Before It Does.**

BlastMap analyzes your AWS architecture and produces a blast radius audit to identify dependencies, single points of failure, and potential failure cascades. It creates a DevOps Agent skill, and annotates CloudFormation templates with risk scores and proposed improvements.

---

## Prerequisites

- **A git repository.** BlastMap use git to version its outputs; initialise one with `git init` if needed.
- **An AI coding agent that supports commands/steering files** (e.g. Claude Code, Kiro). The setup prompt below creates `.claude/commands/blastmap.md` and/or `.kiro/steering/` files.
- **CloudFormation template(s) to analyse**: `*.yaml` / `*.yml` / `*.json` files containing `AWSTemplateFormatVersion` or a top-level `Resources:` key.

---

## Use Case

**What problem does this solve?**

Architecture reviews usually happen once during the design phase then go stale. But the real game beigns after deployment, when production issues occur and on-call engineers have only minutes to understand what's failing and how to respond. BlastMap hands it ready to go before that moment ever happens:
- an annotated CloudFormation template that carries blast-radius scores and proposed fixes
- (optional integration) a `SKILL.md` for AWS DevOps agent to skip straight past "what could even be wrong" and go right to the documented cascade path and first mitigation step

**Who is this for?**

**DevOps engineers**: the people who get paged when production breaks and need answers fast, not a cold investigation.

---

## Setup

Copy the prompt below and run it in any AI coding agent inside your repo. It creates everything BlastMap needs.

```
Set up BlastMap in this repository.

Create these directories:
  .kiro/steering/
  .kiro/hooks/
  .blastmap/

─────────────────────────────────────────
CREATE .kiro/steering/README.md
─────────────────────────────────────────
Write this content:

---
inclusion: manual
---

# BlastMap Steering Files

| File | Purpose |
|------|---------|
| `blastmap.md` | Full 6-step blast radius audit prompt |

Usage: "Using the BlastMap prompt, analyse \<your-cfn-file\>.yaml"
Outputs are written to `.blastmap/`.

─────────────────────────────────────────
CREATE .kiro/steering/blastmap.md
─────────────────────────────────────────
Write this content:

---
inclusion: manual
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
    Read the CloudFormation template(s) in this repository.
    - If the user named a specific file (e.g. "analyse infrastructure/my-stack.yaml"),
      read that file directly.
    - Otherwise, locate CloudFormation files by searching for *.yaml / *.yml / *.json
      files containing "AWSTemplateFormatVersion" or a top-level "Resources:" key.
    Use that file's contents as the architecture to audit in Steps 1-6 above.
    

─────────────────────────────────────────
CREATE .kiro/hooks/blastmap.kiro.hook
─────────────────────────────────────────
Write this content:

{
  "enabled": true,
  "name": "BlastMap Blast Radius Analysis",
  "description": "Runs blast radius audit on every CloudFormation change and writes SKILL.md + annotated CFN output",
  "version": "1",
  "when": {
    "type": "fileEdited",
    "patterns": [
      "**/*.yaml"
    ]
  },
  "then": {
    "type": "askAgent",
    "prompt": "First, check if the edited YAML file contains 'AWSTemplateFormatVersion' or 'Resources:' with AWS resource types (indicating it's a CloudFormation template). If it's NOT a CloudFormation file, respond with 'Not a CloudFormation template, skipping BlastMap analysis' and stop. If it IS a CloudFormation file, run the BlastMap analysis from .kiro/steering/blastmap.md against it."
  }
}

─────────────────────────────────────────
CREATE .claude/commands/blastmap.md
─────────────────────────────────────────
Write this content:

---
description: Run a BlastMap blast radius audit on a CloudFormation file
---

Use the same body content as `.kiro/steering/blastmap.md` above, with these
adjustments for the Claude Code command format:
  - Replace the `inclusion: manual` front matter with the `description:` front
    matter shown above (Claude Code commands don't use Kiro's `inclusion` field).
  - Replace the closing "ARCHITECTURE TO ANALYSE" section with:
      Analyse: $ARGUMENTS
      - If a file was named, read that file directly.
      - Otherwise, locate CloudFormation files by searching for *.yaml / *.yml / *.json
        files containing "AWSTemplateFormatVersion" or a top-level "Resources:" key.
      Use that file's contents as the architecture to audit in Steps 1-6 above.

─────────────────────────────────────────
CREATE .blastmap/.gitkeep
─────────────────────────────────────────
Create an empty file at .blastmap/.gitkeep

─────────────────────────────────────────
CONFIRM
─────────────────────────────────────────
  .kiro/steering/README.md          ✓
  .kiro/steering/blastmap.md ✓
  .kiro/hooks/blastmap.kiro.hook    ✓
  .claude/commands/blastmap.md      ✓
  .blastmap/.gitkeep                ✓
```

---

## Usage

```
/blastmap infrastructure/my-stack.yaml                              ← Claude Code
Using the BlastMap prompt, analyse infrastructure/my-stack.yaml     ← Kiro / other agents
```

**Output:**
```
.blastmap/
├── SKILL.md                        ← AWS DevOps Agent skill
└── a3f9c12-my-stack.yaml           ← annotated CFN
```

After setup, Kiro also runs BlastMap automatically on every `.yaml` save, no manual trigger needed.

---

## What the Output Is Used For

| Output | Used by | How |
|--------|---------|-----|
| `SKILL.md` | AWS DevOps Agent | Loaded into Agent Space; agent reads cascade paths on every incident to cut MTTR |
| `<hash>-<file>.yaml` | Your team / IaC review | Annotated CFN with blast radius scores and fix recommendations per resource |
| `BlastMapSPOF` tag | AWS Config + Resource Groups | Config rule blocks unmitigated SPOFs in prod; Resource Groups lists all live SPOFs |

---

## (Optional) Integrations

> ⚠️ The integrations below use AWS managed services and will incur additional cost.

### AWS DevOps Agent

Upload `SKILL.md` to your DevOps Agent Space. The agent loads it on every incident, reads the pre-computed cascade paths, and posts findings

```bash
cd .blastmap && zip blastmap.zip SKILL.md
```

1. `SKILL.md` must sit at the zip root and include valid frontmatter (`name` + `description`).
2. In the Operator Web App, open the **Skills** page → **Add skill** → **Upload skill**.
3. Drag and drop `blastmap.zip` (max 6 MB; `scripts/` directories are rejected).
4. Select which agent type(s) can use it: `All tasks` (default) makes it available to all.
5. Review the validation results, then click **Upload**.

### AWS X-Ray

X-Ray validates BlastMap's static dependency graph at runtime. Enable it on resources in the cascade paths, primarily Lambda and API Gateway:

| Resource | How to enable |
|----------|--------------|
| Lambda | `TracingConfig: Mode: Active` |
| API Gateway | `TracingEnabled: true` on Stage |
| ECS tasks | X-Ray daemon sidecar container |

Once traces are flowing, compare the X-Ray Service Map against BlastMap's Step 1 graph. Any gap is a hidden runtime dependency: update your template and re-run BlastMap. Replace estimated T+ cascade timings in `SKILL.md` with measured values from real traces.

### Kiro IDE

Already configured by the setup prompt. The hook in `.kiro/hooks/blastmap.kiro.hook` fires on every CloudFormation save and re-runs the full 6-step audit automatically.

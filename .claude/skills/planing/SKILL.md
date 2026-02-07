---
name: planning
description: Generate comprehensive plans for new features by exploring the codebase, synthesizing approaches, validating with spikes, and decomposing into beads. Use when asked to plan a feature, create a roadmap, or design an implementation approach.
---

# Feature Planning Pipeline

Generate quality plans through systematic discovery, synthesis, verification, and decomposition.

## Pipeline Overview

```
USER REQUEST → Discovery → Synthesis → Verification → Decomposition → Validation → Track Planning → Ready Plan
```

| Phase             | Tool                                     | Output                              |
| ----------------- | ---------------------------------------- | ----------------------------------- |
| 1. Discovery      | Parallel sub-agents (A, B, C, D), gkg, exa | Discovery Report                  |
| 2. Synthesis      | Oracle                                   | Approach + Risk Map                 |
| 3. Verification   | Spikes via MULTI_AGENT_WORKFLOW          | Validated Approach + Learnings      |
| 4. Decomposition  | file-beads skill                         | .beads/\*.md files                  |
| 5. Validation     | bv + Oracle                              | Validated dependency graph          |
| 6. Track Planning | bv --robot-plan                          | Execution plan with parallel tracks |

## Required Tools

Before running the pipeline, ensure all required MCP servers and CLI tools are installed:

### MCP Servers (Required)

| MCP Server | Tools Provided | Used By | Install |
| ---------- | -------------- | ------- | ------- |
| **gkg** | `repo_map`, `search_codebase_definitions`, `get_references` | Agent A, Agent B | Install gkg MCP server |
| **exa** | `web_search_exa`, `get_code_context_exa` | Agent D | `claude mcp add -e EXA_API_KEY=<key> exa -- npx -y exa-mcp-server` |

### CLI Tools (Required)

| Tool | Purpose | Used In |
| ---- | ------- | ------- |
| `bd` | Bead management (create, close, update, dep add/remove, show) | Phase 3, 4, 5 |
| `bv` | Bead validator (--robot-suggest, --robot-insights, --robot-priority, --robot-plan) | Phase 5, 6 |
| `jq` | Parse JSON output from bv commands | Phase 5, 6 |

### Skills (Required)

| Skill | Purpose | Used In |
| ----- | ------- | ------- |
| `file-beads` | Convert approach into beads (epics + issues) via `bd create` | Phase 4 |
| `orchestrator` | Spawn parallel worker agents to execute beads | After Phase 6 |

### Note on Librarian

AmpCode's Librarian is NOT available in Claude Code. Agent D (`discovery-external`) replaces this role by using Exa's `web_search_exa` to find design patterns and best practices from the web.

## Phase 1: Discovery (Parallel Exploration)

Launch parallel sub-agents to gather codebase intelligence:

```
Task() → Agent A (discovery-architecture): Architecture snapshot (gkg repo_map)
Task() → Agent B (discovery-patterns):     Pattern search (gkg search_codebase_definitions + get_references)
Task() → Agent C (discovery-constraints):  Constraints (package.json, tsconfig, deps)
Task() → Agent D (discovery-external):     External patterns + Library docs (exa web_search + get_code_context)
```

### Discovery Report Template

Save to `history/<feature>/discovery.md`:

```markdown
# Discovery Report: <Feature Name>

## Architecture Snapshot

- Relevant packages: ...
- Key modules: ...
- Entry points: ...

## Existing Patterns

- Similar implementation: <file> does X using Y pattern
- Reusable utilities: ...
- Naming conventions: ...

## Technical Constraints

- Node version: ...
- Key dependencies: ...
- Build requirements: ...

## External References

- Library docs: ...
- Similar projects: ...
```

## Phase 2: Synthesis (Oracle)

Feed Discovery Report to Oracle for gap analysis:

```
Task(
  subagent_type="oracle-planner",
  description="Phase 2: Synthesis — gap analysis and risk assessment",
  prompt="You are in PHASE 2: SYNTHESIS.
    Feature: <feature>
    Read history/<feature>/discovery.md and produce approach.md.
    Follow Phase 2 instructions in your agent definition."
)
```

Oracle produces:

1. **Gap Analysis** - What exists vs what's needed
2. **Approach Options** - 1-3 strategies with tradeoffs
3. **Risk Assessment** - LOW / MEDIUM / HIGH per component

### Risk Classification

| Level  | Criteria                      | Verification                 |
| ------ | ----------------------------- | ---------------------------- |
| LOW    | Pattern exists in codebase    | Proceed                      |
| MEDIUM | Variation of existing pattern | Interface sketch, type-check |
| HIGH   | Novel or external integration | Spike required               |

### Risk Indicators

```
Pattern exists in codebase? ─── YES → LOW base
                            └── NO  → MEDIUM+ base

External dependency? ─── YES → HIGH
                     └── NO  → Check blast radius

Blast radius >5 files? ─── YES → HIGH
                       └── NO  → MEDIUM
```

Save to `history/<feature>/approach.md`:

```markdown
# Approach: <Feature Name>

## Gap Analysis

| Component | Have | Need | Gap |
| --------- | ---- | ---- | --- |
| ...       | ...  | ...  | ... |

## Recommended Approach

<Description>

### Alternative Approaches

1. <Option A> - Tradeoff: ...
2. <Option B> - Tradeoff: ...

## Risk Map

| Component   | Risk | Reason           | Verification |
| ----------- | ---- | ---------------- | ------------ |
| Stripe SDK  | HIGH | New external dep | Spike        |
| User entity | LOW  | Follows existing | Proceed      |
```

## Phase 3: Verification (Risk-Based)

### For HIGH Risk Items → Create Spike Beads

Spikes are mini-plans executed via MULTI_AGENT_WORKFLOW:

```bash
bd create "Spike: <question to answer>" -t epic -p 0
bd create "Spike: Test X" -t task --blocks <spike-epic>
bd create "Spike: Verify Y" -t task --blocks <spike-epic>
```

### Spike Bead Template

```markdown
# Spike: <specific question>

**Time-box**: 30 minutes
**Output location**: .spikes/<spike-id>/

## Question

Can we <specific technical question>?

## Success Criteria

- [ ] Working throwaway code exists
- [ ] Answer documented (yes/no + details)
- [ ] Learnings captured for main plan

## On Completion

Close with: `bd close <id> --reason "YES: <approach>" or "NO: <blocker>"`
```

### Execute Spikes

Use the MULTI_AGENT_WORKFLOW:

1. `bv --robot-plan` to parallelize spikes
2. `Task()` per spike with time-box
3. Workers write to `.spikes/<feature>/<spike-id>/`
4. Close with learnings: `bd close <id> --reason "<result>"`

### Aggregate Spike Results

```
Task(
  subagent_type="oracle-planner",
  description="Phase 3: Spike Aggregation — update approach with learnings",
  prompt="You are in PHASE 3: SPIKE AGGREGATION.
    Feature: <feature>
    Spike results: <results summary>
    Read history/<feature>/approach.md and update it with validated learnings.
    Follow Phase 3 instructions in your agent definition."
)
```

Update approach.md with validated learnings.

## Phase 4: Decomposition (file-beads skill)

Load the file-beads skill and create beads with embedded learnings:

```bash
skill("file-beads")
```

### Bead Requirements

Each bead MUST include:

- **Spike learnings** embedded in description (if applicable)
- **Reference to .spikes/ code** for HIGH risk items
- **Clear acceptance criteria**
- **File scope** for track assignment

### Example Bead with Learnings

```markdown
# Implement Stripe webhook handler

## Context

Spike bd-12 validated: Stripe SDK works with our Node version.
See `.spikes/billing-spike/webhook-test/` for working example.

## Learnings from Spike

- Must use `stripe.webhooks.constructEvent()` for signature verification
- Webhook secret stored in `STRIPE_WEBHOOK_SECRET` env var
- Raw body required (not parsed JSON)

## Acceptance Criteria

- [ ] Webhook endpoint at `/api/webhooks/stripe`
- [ ] Signature verification implemented
- [ ] Events: `checkout.session.completed`, `invoice.paid`
```

## Phase 5: Validation

### Run bv Analysis

```bash
bv --robot-suggest   # Find missing dependencies
bv --robot-insights  # Detect cycles, bottlenecks
bv --robot-priority  # Validate priorities
```

### Fix Issues

```bash
bd dep add <from> <to>      # Add missing deps
bd dep remove <from> <to>   # Break cycles
bd update <id> --priority X # Adjust priorities
```

### Oracle Final Review

```
Task(
  subagent_type="oracle-planner",
  description="Phase 5: Bead Validation — review completeness and clarity",
  prompt="You are in PHASE 5: BEAD VALIDATION.
    Feature: <feature>
    Read .beads/*.md files, run bv analysis, cross-reference with history/<feature>/approach.md.
    Follow Phase 5 instructions in your agent definition.
    Produce a review report with verdict: READY or NEEDS FIXES."
)
```

## Phase 6: Track Planning

This phase creates an **execution-ready plan** so the orchestrator can spawn workers immediately without re-analyzing beads.

### Step 1: Get Parallel Tracks

```bash
bv --robot-plan 2>/dev/null | jq '.plan.tracks'
```

### Step 2: Assign File Scopes

For each track, determine the file scope based on beads in that track:

```bash
# For each bead, check which files it touches
bd show <bead-id>  # Look at description for file hints
```

**Rules:**

- File scopes must NOT overlap between tracks
- Use glob patterns: `packages/sdk/**`, `apps/server/**`
- If overlap unavoidable, merge into single track

### Step 3: Generate Agent Names

Assign unique adjective+noun names to each track:

- BlueLake, GreenCastle, RedStone, PurpleBear, etc.
- Names are memorable identifiers, NOT role descriptions

### Step 4: Create Execution Plan

Save to `history/<feature>/execution-plan.md`:

```markdown
# Execution Plan: <Feature Name>

Epic: <epic-id>
Generated: <date>

## Tracks

| Track | Agent       | Beads (in order)      | File Scope        |
| ----- | ----------- | --------------------- | ----------------- |
| 1     | BlueLake    | bd-10 → bd-11 → bd-12 | `packages/sdk/**` |
| 2     | GreenCastle | bd-20 → bd-21         | `packages/cli/**` |
| 3     | RedStone    | bd-30 → bd-31 → bd-32 | `apps/server/**`  |

## Track Details

### Track 1: BlueLake - <track-description>

**File scope**: `packages/sdk/**`
**Beads**:

1. `bd-10`: <title> - <brief description>
2. `bd-11`: <title> - <brief description>
3. `bd-12`: <title> - <brief description>

### Track 2: GreenCastle - <track-description>

**File scope**: `packages/cli/**`
**Beads**:

1. `bd-20`: <title> - <brief description>
2. `bd-21`: <title> - <brief description>

### Track 3: RedStone - <track-description>

**File scope**: `apps/server/**`
**Beads**:

1. `bd-30`: <title> - <brief description>
2. `bd-31`: <title> - <brief description>
3. `bd-32`: <title> - <brief description>

## Cross-Track Dependencies

- Track 2 can start after bd-11 (Track 1) completes
- Track 3 has no cross-track dependencies

## Key Learnings (from Spikes)

Embedded in beads, but summarized here for orchestrator reference:

- <learning 1>
- <learning 2>
```

### Validation

Before finalizing, verify:

```bash
# No cycles in the graph
bv --robot-insights 2>/dev/null | jq '.Cycles'

# All beads assigned to tracks
bv --robot-plan 2>/dev/null | jq '.plan.unassigned'
```

## Handoff to Orchestrator

After Phase 6 is complete and `execution-plan.md` is validated, hand off to the `orchestrator` skill to begin execution:

```
skill("orchestrator")
```

The orchestrator will read `history/<feature>/execution-plan.md` and spawn parallel worker agents for each track.

## Output Artifacts

| Artifact          | Location                              | Purpose                            |
| ----------------- | ------------------------------------- | ---------------------------------- |
| Discovery Report  | `history/<feature>/discovery.md`      | Codebase snapshot                  |
| Approach Document | `history/<feature>/approach.md`       | Strategy + risks                   |
| Spike Code        | `.spikes/<feature>/`                  | Reference implementations          |
| Spike Learnings   | Embedded in beads                     | Context for workers                |
| Beads             | `.beads/*.md`                         | Executable work items              |
| Execution Plan    | `history/<feature>/execution-plan.md` | Track assignments for orchestrator |

## Quick Reference

### Tool Selection

| Need               | Tool                                    | Agent/Phase |
| ------------------ | --------------------------------------- | ----------- |
| Codebase structure | `mcp__gkg__repo_map`                    | Agent A (Phase 1) |
| Find definitions   | `mcp__gkg__search_codebase_definitions` | Agent A, B (Phase 1) |
| Find usages        | `mcp__gkg__get_references`              | Agent B (Phase 1) |
| External patterns  | `mcp__exa__web_search_exa`              | Agent D (Phase 1) |
| Library docs       | `mcp__exa__get_code_context_exa`        | Agent D (Phase 1) |
| Gap analysis       | `oracle-planner` agent                  | Phase 2, 3, 5 |
| Create beads       | `skill("file-beads")` + `bd create`     | Phase 4 |
| Validate graph     | `bv --robot-*`                          | Phase 5, 6 |

### Agent → Tool Mapping

| Agent | Required MCP | Required CLI | Called In |
| ----- | ------------ | ------------ | -------- |
| `discovery-architecture` (A) | gkg: `repo_map`, `search_codebase_definitions` | — | Phase 1 |
| `discovery-patterns` (B) | gkg: `search_codebase_definitions`, `get_references` | — | Phase 1 |
| `discovery-constraints` (C) | — (uses built-in Read/Glob) | — | Phase 1 |
| `discovery-external` (D) | exa: `web_search_exa`, `get_code_context_exa` | — | Phase 1 |
| `oracle-planner` | — | `bd show`, `bv --robot-suggest/insights/priority` | Phase 2, 3, 5 |
| Planning orchestrator | — | `bd create/close/update/dep`, `bv --robot-*` | Phase 3, 4, 5, 6 |

### Common Mistakes

- **Skipping discovery** → Plan misses existing patterns
- **No risk assessment** → Surprises during execution
- **No spikes for HIGH risk** → Blocked workers
- **Missing learnings in beads** → Workers re-discover same issues
- **No bv validation** → Broken dependency graph
- **Missing MCP servers** → Agents fail at startup; install gkg and exa first

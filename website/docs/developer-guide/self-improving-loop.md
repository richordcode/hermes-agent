---
sidebar_position: 4
title: "Self-Improving Loop"
description: "How Hermes learns from conversations — background review, skills as procedural memory, and the curator maintenance cycle"
---

# Self-Improving Loop

Hermes implements a **three-tier self-improving system** that turns every conversation into durable learning. Agents don't just answer questions — they capture successful techniques as reusable skills, extract user preferences into memory, and periodically consolidate their skill library to keep it lean and discoverable.

The three tiers operate at different time scales:

| Tier | Trigger | Time Scale | What It Does |
|------|---------|------------|--------------|
| **Background Review** | Every N user turns | Seconds after a turn | Extracts memory + skill updates from the conversation |
| **Skills System** | Agent tool call | During conversations | Creates/patches skills as procedural memory |
| **Curator** | Weekly (configurable) | Days to weeks | Consolidates, prunes, and maintains the skill library |

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HERMES SELF-IMPROVING LOOP                       │
│                                                                     │
│  ┌──────────┐     ┌──────────────┐     ┌──────────────────┐        │
│  │  User    │────▶│  AIAgent     │────▶│  Background      │        │
│  │  Message │     │  Main Loop   │     │  Review Fork     │        │
│  └──────────┘     └──────────────┘     │  (every N turns)  │        │
│                                         │                   │        │
│                                         │  Extraction:      │        │
│                                         │  • Preferences    │        │
│                                         │    → MEMORY.md    │        │
│                                         │  • Techniques     │        │
│                                         │    → skill_manage │        │
│                                         └───────┬───────────┘        │
│                                                 │                   │
│                                          skill_manage()             │
│                                                 │                   │
│                                                 ▼                   │
│                              ┌──────────────────────────┐          │
│                              │  ~/.hermes/skills/       │          │
│                              │  • SKILL.md              │          │
│                              │  • references/           │          │
│                              │  • templates/            │          │
│                              │  • scripts/              │          │
│                              │                          │          │
│                              │  ~/.hermes/skills/       │          │
│                              │    .usage.json (sidecar) │          │
│                              └──────────┬───────────────┘          │
│                                         │                          │
│                                  Every 7 days                      │
│                                         │                          │
│                                         ▼                          │
│                              ┌──────────────────────────┐          │
│                              │  Curator                  │          │
│                              │  • Auto state transitions │          │
│                              │  • LLM umbrella merges    │          │
│                              │  • Archive stale →        │          │
│                              │    .archive/              │          │
│                              │  • Cron job ref rewrite   │          │
│                              │  • Pre-run tar.gz snap    │          │
│                              └──────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Tier 1: Background Review (In-Session Learning)

**Source:** `run_agent.py` → `_spawn_background_review()`

After every `nudge_interval` user turns (default 10), the agent forks a background copy of itself to review the conversation. This runs as a daemon thread — the user never waits for it.

### Trigger Logic

```python
# run_agent.py:11639 — Memory trigger (turn-based)
if (self._memory_nudge_interval > 0
        and self._turns_since_memory >= self._memory_nudge_interval):
    _should_review_memory = True

# run_agent.py:15158 — Skill trigger (iteration-based)
if (self._skill_nudge_interval > 0
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True

# run_agent.py:15175 — Spawn if either fired
if _should_review_memory or _should_review_skills:
    self._spawn_background_review(
        messages_snapshot=list(messages),
        review_memory=_should_review_memory,
        review_skills=_should_review_skills,
    )
```

The review runs **after** the response is delivered to the user, so it never competes for the model's attention during an active task.

### Fork Configuration

```python
# run_agent.py:4180-4190
review_agent = AIAgent(
    model=self.model,            # inherits main session's model
    max_iterations=16,           # small budget — this is a review, not a task
    quiet_mode=True,             # no spinner, no progress output
    enabled_toolsets=["memory", "skills"],  # only these two
    ...
)
review_agent._memory_write_origin = "background_review"
review_agent._memory_nudge_interval = 0    # no recursive review
review_agent._skill_nudge_interval = 0
review_agent.suppress_status_output = True
```

The fork inherits the parent's provider, model, API credentials, and memory store. It writes directly into the shared memory/skill stores — the changes are immediately available to the next main-session turn.

### Review Prompts

Three prompt variants are selected based on which triggers fired:

| Prompt Constant | When Used | Focus |
|-----------------|-----------|-------|
| `_COMBINED_REVIEW_PROMPT` | Both memory + skill triggers fire | Full review of user identity + techniques |
| `_MEMORY_REVIEW_PROMPT` | Only memory trigger fires | Extract user preferences, persona, desires |
| `_SKILL_REVIEW_PROMPT` | Only skill trigger fires | Identify reusable techniques → create/patch skills |

**Key instructions to the review agent:**

- **Memory:** Save facts about the user and durable preferences. "Memory captures who the user is and what the current situation and state of your operations are."
- **Skills:** Capture "how to do this class of task." Four-tier preference: (1) patch a currently-loaded skill, (2) patch an existing umbrella skill, (3) add a support file under an existing umbrella, (4) create a new class-level umbrella.
- **Do NOT capture:** environment-dependent failures, negative claims about tools, session-specific transient errors, or one-off task narratives.

### User-Visible Output

```python
# run_agent.py:4221
self._safe_print("  💾 Self-improvement review: created skill 'python-packaging' · "
                 "updated memory preferences")
```

The summary is compact — one line listing distinct actions taken. Failed reviews log at WARNING and are never surfaced to the user.

---

## Tier 2: Skills System (Procedural Memory)

**Source:** `tools/skill_manager_tool.py` + `tools/skills_tool.py`

Skills are Hermes's **procedural memory** — narrow, actionable documents that capture how to do a specific type of task. They follow Anthropic's Claude Skills format with progressive disclosure: metadata first (name + description), full instructions on demand, linked files when needed.

### Skill Storage

```
~/.hermes/skills/
├── python-packaging/
│   ├── SKILL.md              # YAML frontmatter + markdown body
│   ├── references/
│   │   └── pyproject.toml-examples.md
│   ├── templates/
│   │   └── setup.cfg
│   ├── scripts/
│   │   └── verify-build.sh
│   └── assets/
│       └── logo.png
├── github-pr-review/
│   └── SKILL.md
└── .usage.json               # Telemetry sidecar
```

### Tool Surface: `skill_manage`

Six actions, all dispatched through a single tool:

```python
# tools/skill_manager_tool.py
def skill_manage(action, name, ...):
    if action == "create":
        # Full SKILL.md write + directory scaffold
    elif action == "patch":
        # Targeted find-and-replace (uses fuzzy_match engine)
    elif action == "edit":
        # Full SKILL.md rewrite (major overhauls only)
    elif action == "delete":
        # Remove skill directory (must pass absorbed_into for consolidations)
    elif action == "write_file":
        # Add references/, templates/, scripts/, or assets/ file
    elif action == "remove_file":
        # Remove a supporting file
```

### Provenance Tracking

Not all skills are eligible for curator management. The system distinguishes four provenances:

```python
# tools/skill_usage.py
def is_agent_created(skill_name):
    """Must be neither bundled nor hub-installed."""
    off_limits = _read_bundled_manifest_names() | _read_hub_installed_names()
    return skill_name not in off_limits

def _is_curator_managed_record(record):
    """Only skills explicitly marked by a background review fork."""
    return record.get("created_by") == "agent"
```

The `created_by: "agent"` marker is set **only** when `skill_manage(action="create")` is called from inside a background review fork — never from a user-directed tool call. This prevents the curator from touching skills the user created manually.

### Usage Telemetry

Every skill interaction bumps counters in `~/.hermes/skills/.usage.json`:

```python
# tools/skill_usage.py
def bump_view(skill_name):    # skill_view() was called
    rec["view_count"] += 1
    rec["last_viewed_at"] = now()

def bump_use(skill_name):     # skill was loaded into prompt
    rec["use_count"] += 1
    rec["last_used_at"] = now()

def bump_patch(skill_name):   # skill_manage(patch/edit/write_file)
    rec["patch_count"] += 1
    rec["last_patched_at"] = now()
```

These timestamps are the **sole input** to the curator's auto-transition logic. A skill that was viewed last week but never used or patched has no recent "activity" — the curator will eventually mark it stale.

---

## Tier 3: Curator (Background Maintenance)

**Source:** `agent/curator.py` (~1782 lines) + `agent/curator_backup.py` + `tools/skill_usage.py`

The curator is the **quality-control layer**. It prevents the skill library from degrading into a flat list of hundreds of narrow, near-duplicate one-session-one-skill entries.

:::info Design Philosophy
The goal of the skill collection is a **library of class-level instructions and experiential knowledge**. A collection of hundreds of narrow skills where each captures one session's specific bug is a failure — not a feature. The curator builds umbrellas.
:::

### Triggering

The curator is **not a cron daemon**. It piggy-backs on existing infrastructure:

| Entry Point | Code Location | Behavior |
|-------------|---------------|----------|
| **Gateway** | `gateway/run.py:16198` | Checked every `CURATOR_EVERY` cron ticks |
| **CLI** | `cli.py:10897` | Checked on session start |
| **Manual** | `hermes curator run` | Immediate, bypasses all gates |

All paths go through the same gate:

```python
# agent/curator.py
def should_run_now(now=None):
    if not is_enabled():           # curator.enabled in config.yaml
        return False
    if is_paused():                # hermes curator pause
        return False
    if last_run_at is None:        # first observation → seed + defer
        seed_state()
        return False
    return (now - last_run_at) >= interval_hours  # default: 7 days
```

**First-run behavior:** On a brand-new install, the curator seeds `last_run_at` to "now" and defers the first real pass by one full interval. This gives users time to review their library, pin important skills, or opt out entirely.

### Two-Phase Execution

Each run has two distinct phases:

#### Phase 1: Automatic Transitions (Deterministic, No LLM)

```python
# agent/curator.py: apply_automatic_transitions()
for row in agent_created_report():
    if row.pinned:                        # Pinned → skip entirely
        continue

    anchor = row.last_activity_at or row.created_at
    if anchor <= archive_cutoff:          # Default: 90 days
        archive_skill(name)               # → .archive/
    elif anchor <= stale_cutoff:          # Default: 30 days
        set_state(name, STATE_STALE)
    elif anchor > stale_cutoff and row.state == STATE_STALE:
        set_state(name, STATE_ACTIVE)     # Reactivated
```

**Lifecycle state machine:**

```
active ──(30 days no activity)──▶ stale ──(90 days no activity)──▶ archived
  ▲                                   │
  └──────(new activity detected)──────┘

pinned: fully exempt from all automatic transitions
```

#### Phase 2: LLM Review (Umbrella Consolidation)

The curator spawns a forked `AIAgent` with the same pattern as background review:

```python
# agent/curator.py: _run_llm_review()
review_agent = AIAgent(
    model=_model_name,
    max_iterations=9999,         # High ceiling for large libraries
    quiet_mode=True,
    platform="curator",
    skip_context_files=True,
    skip_memory=True,            # No memory operations during curation
)
review_agent._memory_nudge_interval = 0
review_agent._skill_nudge_interval = 0
```

**The review prompt** (`CURATOR_REVIEW_PROMPT`) instructs the fork to:

1. **Scan candidate skills** — all agent-created skills with usage stats
2. **Identify prefix clusters** — skills sharing domain keywords (e.g., `hermes-config-*`, `gateway-*`, `ollama-*`)
3. **Consolidate using three strategies:**

| Strategy | When | Action |
|----------|------|--------|
| Merge into existing umbrella | One skill in the cluster is already broad enough | Patch umbrella to add sibling's unique insight as labeled section, then `skill_manage(delete, absorbed_into=<umbrella>)` |
| Create new umbrella | No existing member is broad enough | `skill_manage(create)` with class-level coverage, archive narrow siblings |
| Demote to support files | Sibling has narrow-but-valuable session detail | Move into umbrella's `references/`, `templates/`, or `scripts/`, then archive sibling |

**Hard invariants enforced by the prompt:**

- Never touch bundled or hub-installed skills (already filtered out)
- Never delete — maximum destructive action is archive (recoverable)
- Never touch pinned skills
- Usage counters are **not** a reason to skip consolidation — judge on content overlap

### Classification with `absorbed_into`

When the curator archives a skill, it must declare intent. The `absorbed_into` parameter on `skill_manage(action="delete")` is the authoritative signal:

```python
# tools/skill_manager_tool.py: _delete_skill()
def _delete_skill(name, absorbed_into=None):
    if absorbed_into is None:
        # Legacy — accepted but logged as ambiguous
    elif absorbed_into == "":
        # Explicit: truly pruned, no forwarding target
    else:
        # Explicit: content absorbed into umbrella — must exist on disk
        target = _find_skill(absorbed_into)
        if not target:
            return error("umbrella doesn't exist")
```

The curator's classification reconciler distinguishes two outcomes:

- **Consolidations** — content lives on under a different name. The absorbed sibling's directory moves to `.archive/` for safety, but the knowledge persists in the umbrella.
- **Prunings** — archived for staleness/irrelevance, content not preserved elsewhere.

This distinction drives downstream behavior like cron job skill-reference rewriting.

### Cron Job Reference Rewriting

When a cron job references skill `X` and the curator consolidates `X` into umbrella `Y`, the cron job's `skills` array is updated in-place:

```python
# agent/curator.py: _write_run_report() → cron.jobs.rewrite_skill_refs()
cron_rewrites = _rewrite_cron_refs(
    consolidated={"old-skill": "umbrella-skill"},
    pruned=["truly-stale-skill"],
)
```

This keeps scheduled jobs working across consolidation passes without user intervention.

### Backup and Rollback

Before every mutating run, the curator takes a snapshot:

```python
# agent/curator.py: run_curator_review()
from agent import curator_backup
snap = curator_backup.snapshot_skills(reason="pre-curator-run")
```

**What gets backed up** (tar.gz under `~/.hermes/skills/.curator_backups/<timestamp>/`):
- All `SKILL.md` files + directories
- `.usage.json` (usage telemetry)
- `.archive/` (previously archived skills)
- `.curator_state` (last-run pointer)
- `.bundled_manifest` (protection markers)
- `cron/jobs.json` (so rollback restores pre-rewrite state)

**What is excluded:**
- `.curator_backups/` itself (would recurse)
- `.hub/` (managed by the skills hub)

Rollback is available via `hermes curator rollback <snapshot>`. Even the rollback itself is undoable — the current state is snapshotted before restoring.

### Per-Run Reports

Every curator run produces a report under `~/.hermes/logs/curator/{YYYYMMDD-HHMMSS}/`:

| File | Content |
|------|---------|
| `run.json` | Machine-readable: full counts, tool call log, classification results, cron rewrites |
| `REPORT.md` | Human-readable: auto-transition summary, consolidations with rationale, prunings, new skills, state transitions, recovery instructions |
| `cron_rewrites.json` | Only when cron jobs were touched |

Users can inspect `hermes curator status` for a quick summary or read the full report for detail.

---

## Configuration

All three tiers are independently configurable in `~/.hermes/config.yaml`:

```yaml
# ── Background Review ──
skills:
  creation_nudge_interval: 10     # Turns between skill review passes (0 = off)
  guard_agent_created: false      # Security scan on agent-created skills

memory:
  nudge_interval: 10              # Turns between memory review passes (0 = off)

# ── Curator ──
curator:
  enabled: true                   # Master switch
  interval_hours: 168             # 7 days between runs
  min_idle_hours: 2               # Agent must be idle this long
  stale_after_days: 30            # Mark stale if no activity
  archive_after_days: 90          # Archive if still inactive
  backup:
    keep: 5                       # Number of snapshots to retain
```

---

## Design Principles

These principles apply across all three tiers:

### 1. Non-Blocking

Background review and curator both run in daemon threads. The user's active conversation never waits for self-improvement work. If a review fork crashes, the error is logged at DEBUG/WARNING and silently discarded.

### 2. Safe by Default

The maximum destructive action any tier can take is **archival** (moving a skill to `.archive/`). True deletion is never automated. Archives are recoverable via `hermes curator restore <name>` or by manually moving the directory back.

### 3. Source Isolation

The curator's scope is strictly limited by provenance:
- **Bundled skills** (shipped with the repo) → tracked in `.bundled_manifest`, never modified
- **Hub-installed skills** (from [agentskills.io](https://agentskills.io)) → tracked in `.hub/lock.json`, never modified
- **User-created skills** (manual `skill_manage`) → not marked `agent_created`, never auto-curated
- **Agent-created skills** (from background review fork) → `created_by: "agent"`, fully curated

### 4. Snapshot Protection

Every curator run is preceded by a tar.gz snapshot. Rollback is always possible, including rollback of the rollback itself.

### 5. Prompt Cache Aware

Background review forks and curator forks run in their own `AIAgent` instances with their own prompt caches. They never modify the main session's conversation history, messages, or system prompt — preserving Anthropic prefix cache validity across turns.

### 6. Negative Feedback Prevention

All three review prompts explicitly forbid capturing:
- Environment-dependent failures (missing binaries, uninstalled packages)
- Negative claims about tools ("X tool is broken")
- Session-specific transient errors
- One-off task narratives

These would otherwise harden into persistent self-imposed constraints the agent cites against itself for months after the actual problem was fixed.

---

## Related

- [Curator user guide](/docs/user-guide/features/curator.md) — end-user documentation
- [Skills](/docs/user-guide/features/skills.md) — skill system from the user's perspective
- [Agent Loop](/docs/developer-guide/agent-loop.md) — the main conversation loop that hosts review spawns
- [Creating Skills](/docs/developer-guide/creating-skills.md) — authoring bundled skills
- [Memory Provider Plugin](/docs/developer-guide/memory-provider-plugin.md) — pluggable memory backends

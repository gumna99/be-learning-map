# Design: Learning Progress Tracking for be-learning-map

**Date**: 2026-05-28
**Status**: Approved
**Scope**: Extend existing `be-learning-map` skill with progress tracking

## Problem

The skill predicts which BE concepts a feature involves (`map`), teaches them (`explain`), tests understanding (`quiz`), and calibrates accuracy (`calibrate`). But it doesn't track **the user's learning over time** — which concepts have been encountered, how many times, and which ones the user has internalized.

Without this, there's no way to know:
- Which concepts keep appearing but haven't been deeply understood yet
- Which concepts have never been encountered and need deliberate study
- Overall learning coverage across features

## Design Decisions

| Decision | Choice | Reasoning |
|----------|--------|-----------|
| Tracking model | Two-tier: auto "touched" + manual "learned" | "Encountered" and "understood" are different; frequency data reveals which concepts need more practice |
| Storage location | `~/.claude/skills/be-learning-map/references/progress.json` | Learning is personal and cross-repo; belongs with the skill, not with any single codebase |
| Sub-command | Single `/be-learn progress` with optional review flow | One command, minimal cognitive load |
| "Learned" criteria | User can explain "why this was used here" in their own words | Subjective but practical; no automated quiz gate |
| Approach | Extend existing SKILL.md (not a new skill) | All `/be-learn` sub-commands are one learning tool; splitting would fragment the mental model |

## Data Structure — `progress.json`

Stored at `~/.claude/skills/be-learning-map/references/progress.json`.

```json
{
  "version": "1.0",
  "concepts": {
    "transaction-acid": {
      "status": "touched",
      "encounters": [
        { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28", "via": "map" }
      ]
    },
    "dependency-injection": {
      "status": "learned",
      "encounters": [
        { "feature": "005-factor-settings-unit-conversion-ratio", "date": "2026-05-27", "via": "map" },
        { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28", "via": "explain" }
      ],
      "learned_at": { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28" }
    }
  }
}
```

Rules:
- `status`: only `"touched"` or `"learned"`
- `encounters`: auto-appended, deduplicated by (feature + via) pair
- `learned_at`: written only when user manually marks as learned
- Concepts absent from JSON = "untouched" (no need to pre-populate)
- `via`: one of `"map"`, `"explain"`

## Auto-write: Encounter Recording

### In `map` (after Step 4 — save `be-learning-map.json`)

Add Step 5:
- For each **HIGH confidence** match, append an encounter to `progress.json`:
  - `feature`: the feature name
  - `date`: today
  - `via`: `"map"`
- Skip if the same (feature, via) pair already exists
- If `progress.json` doesn't exist, create it with `{ "version": "1.0", "concepts": {} }`
- Do not change `status` if already `"learned"`

### In `explain` (after Step 3 — teach)

Add Step 4:
- Append an encounter for the explained concept:
  - `feature`: current feature (from `be-learning-map.json` if available, else `"standalone"`)
  - `date`: today
  - `via`: `"explain"`
- Same dedup and status rules as above

### Not modified

- `quiz`: testing understanding, not encountering a concept
- `calibrate`: accuracy tool, not a learning event

## Sub-command 5: `progress`

### Flow

```
Step 1 — Read progress.json (create if missing) + be-roadmap.json
Step 2 — Display summary table (three sections: learned / touched / untouched)
Step 3 — If "touched but not learned" concepts exist, ask: review now? (y/n)
Step 4 — If yes: list touched concepts, user inputs numbers of learned ones
Step 5 — Update progress.json: set status to "learned", write learned_at
```

### Terminal Output — Summary Table

```
BE Learning Progress
================================================================

Learned (3/28):
  dependency-injection    2 encounters (005, 007) | learned at 007
  testing-strategy        2 encounters (005, 007) | learned at 007
  design-patterns         2 encounters (005, 007) | learned at 005

Touched but not learned (5/28):
  transaction-acid        1 encounter (007)
  caching                 1 encounter (007)
  service-communication   1 encounter (007)
  observability           1 encounter (007)
  fault-tolerance         1 encounter (007)

Untouched (20/28):
  web-security, auth, ci-cd, ...

================================================================
Progress: 3/28 learned | 5/28 touched | 20/28 untouched

5 concepts touched but not marked as learned. Review now? (y/n)
```

### Terminal Output — Review Mode

```
Review: which concepts have you learned?
Criteria: can you explain "why it was used this way" in your own words?

  1. transaction-acid        — per-group withWriterTransaction
  2. caching                 — Redis cross-service payload store
  3. service-communication   — cupcake writes / madeleine reads
  4. observability           — structured logging per group
  5. fault-tolerance         — partial failure strategy

Enter numbers of learned concepts (comma-separated), or 'skip': 1,3
```

After input: update `progress.json`, display confirmation, show updated counts.

## Files Changed

| File | Change |
|------|--------|
| `SKILL.md` | Add sub-command 5 (progress), modify map Step 4→5, modify explain Step 3→4 |
| `references/progress.json` | New file, auto-created on first encounter |
| `be-learning-map` git repo `SKILL.md` | Sync copy |

No changes to: `be-roadmap.json`, `be-learning-map.json` format, `CLAUDE.md`.

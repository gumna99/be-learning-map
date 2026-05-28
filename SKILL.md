---
name: be-learning
invoke: user
description: |
  BE engineer learning companion. Five sub-commands:
  /be-learning map — predict which concepts a feature involves
  /be-learning explain {id} — teach a concept with real codebase examples
  /be-learning quiz {id} — self-test understanding
  /be-learning calibrate — compare predictions vs actual code
  /be-learning progress — view learning progress and mark concepts as learned
  Scope: Backend repos only (cream, tiramisu, cupcake, madeleine, puff, etc.).
---

# BE Learning

Identify which backend engineering concepts each feature involves, teach them using real codebase examples, and improve matching accuracy over time.

**Scope**: Backend repos only. This skill's 26 concepts are all backend-focused (NestJS, TypeORM, PostgreSQL, etc.). Frontend RDs working in souffle/pudding should not use this skill — the matching results will be poor.

---

## Prerequisites

- Feature directory exists under `specs/` with at least `tasks.md` (produced by `/speckit.tasks`)
- Roadmap JSON at `~/.claude/skills/be-learning-map/references/be-roadmap.json`
- For `calibrate`: the feature has been implemented (source code exists)

---

## Sub-commands

| Command | Purpose | When to use |
|---------|---------|-------------|
| `/be-learning map` | Predict which BE concepts a feature involves | After `/speckit.tasks`, before implementation |
| `/be-learning explain {concept-id}` | Teach a concept using real codebase examples | During or after implementation |
| `/be-learning quiz {concept-id}` | Self-test understanding with codebase questions | After reading an explanation |
| `/be-learning calibrate` | Compare predictions vs actual code, improve matching | After implementation is complete |
| `/be-learning progress` | View learning progress and mark concepts as learned | Anytime, especially after completing a feature |

---

## Sub-command 1: map

Scan `tasks.md` + `plan.md` against `be-roadmap.json` to predict which BE concepts this feature will exercise.

### Flow

**Step 1 — Gather context**

1. Locate the feature directory. If the user didn't specify, look for the most recent `specs/*/tasks.md`.
2. Read `tasks.md` fully. Also read `plan.md` and `spec.md` if they exist (for additional context).
3. Load `be-roadmap.json` from `~/.claude/skills/be-learning-map/references/`.
4. **mint-tea artifacts** (optional but recommended): If `plan.md` references a mint-tea technical design path (e.g., `mint-tea/CarbonX/L2/005-.../`), attempt to read `sequence.plantuml` and `*-api.yaml` from that path. These reveal cross-service interactions (service-communication, async-communication, fault-tolerance) that tasks.md alone may not surface.
5. **Incremental mode**: If `spec_update.diff` exists in the feature directory, this is an incremental change to an existing feature. In this case, focus matching on the **delta** — concepts introduced by the change, not the entire feature. Tag each matched concept as `"source": "new"` (introduced by this change) or `"source": "existing"` (already present in prior implementation) in the output.

**Step 2 — 3-Layer Matching**

For each concept in the roadmap, score it against the feature content using three layers:

**Layer 1 — Keyword scan** (weight: 1 per hit)
- Search all task descriptions, acceptance criteria, and plan content for each concept's `keywords`.
- Case-insensitive, whole-word match preferred but partial match allowed for compound terms.

**Layer 2 — File pattern scan** (weight: 2 per hit)
- Check if any task mentions files matching the concept's `file_patterns` (e.g., `*.repository.ts`, `*.e2e.spec.ts`).
- Also check `plan.md` Project Structure section for matching file paths.

**Layer 3 — Context triggers** (weight: 3 per hit)
- Evaluate each concept's `trigger_context` rules against the combined content.
- These are semantic rules like "task involves creating a new @Injectable service that does one thing" — use judgment, not just regex.
- Also check concept co-occurrence: if concept A is matched and concept B's trigger says "when A is present", score B.

**Scoring**: Sum weighted hits. Threshold:
- Score >= 5: **High confidence** match
- Score 3-4: **Medium confidence** match
- Score 1-2: **Low confidence** (mention but don't emphasize)
- Score 0: Skip

**Step 3 — Output**

Display a terminal report:

```
BE Learning Map: {feature-name}
================================================================

Matched Concepts (by relevance):

HIGH
  1. [testing-strategy] 測試策略 & 品質思維
     Why this feature: TDD workflow (RED → GREEN phases), E2E + unit test files
     Knowledge points to watch:
       - TDD: 先寫測試（RED）→ 實作（GREEN）→ 重構
       - E2E 啟動真實 app + 真實 DB
     Tasks: T001, T002, T005, T009

  2. [di-nestjs] NestJS 依賴注入
     Why this feature: New @Injectable NetWeightPrefillResolver service
     Knowledge points to watch:
       - @Injectable + constructor 注入
       - Module providers/exports 註冊
     Tasks: T003, T004

MEDIUM
  3. [design-patterns] Design Patterns
     ...

LOW
  4. ...

================================================================
Total: {N} concepts matched ({H} high, {M} medium, {L} low)

Run `/be-learning explain {concept-id}` for deep-dive on any concept.

Cross-service notice (if service-communication is HIGH):
  This feature involves cross-service communication ({reasons}).
  Consider running `/be-learning map` in the other repo(s) involved.
```

**Cross-service notice rule**: When `service-communication` is matched at HIGH confidence, append a notice block at the end of the terminal report. Extract the affected services from plan.md or tasks.md (e.g., "cupcake writes → madeleine reads via Redis") and remind the user to run `/be-learning map` in the other repo(s). This notice is omitted when `service-communication` is not HIGH.

**Step 4 — Save predictions**

Write `be-learning-map.json` to the feature's `specs/{feature}/` directory:

```json
{
  "feature": "005-factor-settings-unit-conversion-ratio",
  "generated_at": "2026-05-27T10:00:00Z",
  "source": "tasks.md",
  "matches": [
    {
      "concept_id": "testing-strategy",
      "confidence": "high",
      "score": 12,
      "matched_tasks": ["T001", "T002", "T005"],
      "reasons": ["TDD RED/GREEN phases", "E2E test files", "unit test files"],
      "knowledge_points_highlighted": [
        "TDD: RED → GREEN → REFACTOR",
        "E2E: real app + real DB"
      ]
    }
  ]
}
```

This file is used later by `calibrate` to compare predictions vs reality.

**Step 5 — Record encounters**

For each **HIGH confidence** match, append an encounter to `~/.claude/skills/be-learning-map/references/progress.json`:
- `feature`: the feature name
- `date`: today (YYYY-MM-DD)
- `via`: `"map"`
- Skip if the same (feature, via) pair already exists for that concept
- Do not change `status` if already `"learned"`
- If `progress.json` doesn't exist, create it: `{ "version": "1.0", "concepts": {} }`

---

## Sub-command 2: explain {concept-id}

Teach a single BE concept using real examples from **this codebase**.

### Flow

**Step 1 — Load concept**

1. Load the concept from `be-roadmap.json` by ID.
2. If a `be-learning-map.json` exists in the current feature directory, load it to get feature-specific context (which tasks matched, why).

**Step 2 — Find real examples**

3. Search the codebase for files matching the concept's `file_patterns` and `code_signatures`.
4. Prioritize examples from the current feature's files (if implementation exists), then fall back to elsewhere in the codebase.
5. Select 1-2 concrete code snippets that best illustrate the concept.

**Step 3 — Teach (4-part structure)**

Present the explanation in this order:

```
## {concept.name} — {concept.why}
Depth: {concept.depth} | Pillar: {concept.pillar}

### 1. Why does this matter?
{concept.why — expanded into 2-3 sentences in zh-TW, relating to the user's current feature}

### 2. Show me the code
{1-2 annotated code snippets from the codebase}

  File: src/emission/services/net-weight-prefill-resolver.service.ts
  ```typescript
  // [annotated snippet with inline comments explaining the concept]
  ```

  This demonstrates {concept knowledge point} because {explanation}.

### 3. Knowledge points checklist
{List each knowledge_point with a checkbox}
- [ ] 命名說出意圖
- [ ] 函式只做一件事（SRP）
- [ ] ...

### 4. Go deeper
{concept.resources — books, links}
{concept.tools — relevant tools/frameworks in this project}
```

**Step 4 — Record encounter**

Append an encounter to `~/.claude/skills/be-learning-map/references/progress.json` for the explained concept:
- `feature`: current feature name (from `be-learning-map.json` if available, else `"standalone"`)
- `date`: today (YYYY-MM-DD)
- `via`: `"explain"`
- Same dedup and status rules as `map` Step 5

**Teaching guidelines:**
- Use zh-TW for explanations (matching the user's language preference)
- Keep explanations concise — this is a working engineer, not a classroom lecture
- Always ground explanations in real code from this repo, never hypothetical examples
- Connect the concept to the specific feature when possible
- If the user's depth level is "基礎", avoid jargon and build from first principles
- If "核心" or "進階", assume foundational understanding exists

---

## Sub-command 3: quiz {concept-id}

Generate codebase-grounded questions to verify understanding.

### Flow

**Step 1 — Load concept + find code**

1. Load concept from `be-roadmap.json`.
2. Find 2-3 relevant code locations in the codebase (same search as `explain`).

**Step 2 — Generate questions**

Create 3 questions at escalating difficulty:

```
## Quiz: {concept.name}

### Q1 (Identify)
Look at `src/emission/services/net-weight-prefill-resolver.service.ts:15-30`.
Which SOLID principle does this class demonstrate? Why?

### Q2 (Explain)
In `src/emission/utils/recompute-auto-conversion.ts`, the function `planUcrReconcile()`
is a pure function outside any class. What advantage does this give for testing?

### Q3 (Apply)
If you needed to add a new prefill strategy for a different emission source type,
how would you structure it following the patterns in this codebase?
Hint: look at how NetWeightPrefillResolver is registered and used.
```

**Question types:**
- **Identify**: Point to code, ask what concept/pattern it demonstrates
- **Explain**: Ask why a design choice was made
- **Apply**: Ask how to extend/modify using the same pattern

**Step 3 — Wait for answers**

After presenting questions, wait for the user to answer. Then:
- Provide feedback on each answer (correct/partially correct/incorrect)
- Reference the specific code that supports the correct answer
- If incorrect, give a hint and let the user try again before revealing the answer

---

## Sub-command 4: calibrate

Compare pre-implementation predictions against actual implemented code to improve matching accuracy.

### Prerequisites

- `be-learning-map.json` exists in the feature directory (produced by `map`)
- The feature has been implemented (code changes exist in the repo)

### Flow

**Step 1 — Load predictions**

1. Read `be-learning-map.json` from the feature directory.
2. Read `plan.md` to identify which files were supposed to be created/modified.

**Step 2 — Analyze actual code**

3. For each file listed in `plan.md` Project Structure (MODIFY/NEW files):
   - Read the actual implementation.
   - Scan for all concept indicators: keywords, patterns, code signatures from `be-roadmap.json`.
4. Also check git diff if available (`git diff main...HEAD` or `git log --oneline` for the feature branch) to find files changed that weren't in the plan.

**Step 3 — Compare predictions vs reality**

For each concept in `be-roadmap.json`:
- Was it predicted (in `be-learning-map.json`)? At what confidence?
- Is it actually present in the code? What evidence?

Classify each concept into:

| Category | Meaning |
|----------|---------|
| TRUE POSITIVE | Predicted and found in code |
| FALSE POSITIVE | Predicted but NOT found in code |
| FALSE NEGATIVE | NOT predicted but found in code |
| TRUE NEGATIVE | Not predicted, not found (skip from report) |

**Step 4 — Output calibration report**

```
Calibration Report: {feature-name}
================================================================

Accuracy: {TP}/{TP+FP+FN} = {percentage}%
Precision: {TP}/{TP+FP} (how many predictions were correct)
Recall: {TP}/{TP+FN} (how many actual concepts were caught)

TRUE POSITIVES (correctly predicted):
  [testing-strategy] Score: 12 → Found: TDD pattern in test files
  [di-nestjs] Score: 8 → Found: @Injectable NetWeightPrefillResolver

FALSE POSITIVES (predicted but absent):
  [error-handling] Score: 3 → Not found: no custom exception handling added
  Reason: keyword "error" appeared in spec but no error handling code was needed

FALSE NEGATIVES (missed but present):
  [transaction-acid] Score: 0 → Found: DbTransaction in reconciler service
  Root cause: tasks.md doesn't mention transactions, but reconciler uses them
  Suggested fix: add keywords ["DbTransaction", "reconcile", "bulkInsert"] to concept

================================================================
Suggested roadmap updates: {N} concepts need keyword/trigger adjustments
```

**Step 5 — Suggest roadmap updates**

For each FALSE NEGATIVE, propose specific changes to `be-roadmap.json`:
- New keywords to add
- New file_patterns to add
- New trigger_context rules to add
- New code_signatures to add

Present changes as a diff and ask the user to confirm before applying.

**Step 6 — Apply updates (with confirmation)**

If the user approves, update `be-roadmap.json` with the new keywords/triggers. Add a `calibration_history` entry to track what was learned:

```json
{
  "calibrated_from": "005-factor-settings-unit-conversion-ratio",
  "date": "2026-05-27",
  "changes": ["Added 'DbTransaction' to transaction-acid keywords"]
}
```

---

## Sub-command 5: progress

View learning progress across all features and optionally mark concepts as learned.

### Prerequisites

- `be-roadmap.json` exists (for the full concept list)
- `progress.json` is auto-created on first `map`/`explain` run; if missing, treat all concepts as untouched

### Flow

**Step 1 — Load data**

1. Read `~/.claude/skills/be-learning-map/references/progress.json` (create empty if missing).
2. Read `be-roadmap.json` for the full concept list.

**Step 2 — Display summary**

```
BE Learning Progress
================================================================

Learned ({L}/28):
  {concept-id}    {N} encounters ({feature-list}) | learned at {feature}
  ...

Touched but not learned ({T}/28):
  {concept-id}    {N} encounters ({feature-list})
  ...

Untouched ({U}/28):
  {concept-id}, {concept-id}, ...

================================================================
Progress: {L}/28 learned | {T}/28 touched | {U}/28 untouched

{T} concepts touched but not marked as learned. Review now? (y/n)
```

**Step 3 — Review (if user says yes)**

List touched-but-not-learned concepts with numbers. Ask the user to input which ones they've learned (comma-separated numbers, or `skip`).

Criteria reminder: "Can you explain why it was used this way in your own words?"

```
  1. transaction-acid        — per-group withWriterTransaction
  2. caching                 — Redis cross-service payload store
  3. service-communication   — cupcake writes / madeleine reads

Enter numbers of learned concepts (comma-separated), or 'skip':
```

**Step 4 — Update progress.json**

For each selected concept:
- Set `status` to `"learned"`
- Write `learned_at`: `{ "feature": "{most recent encounter's feature}", "date": "today" }`

Display updated counts as confirmation.

### progress.json schema

Stored at `~/.claude/skills/be-learning-map/references/progress.json`:

```json
{
  "version": "1.0",
  "concepts": {
    "dependency-injection": {
      "status": "learned",
      "encounters": [
        { "feature": "005-factor-settings-unit-conversion-ratio", "date": "2026-05-27", "via": "map" },
        { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28", "via": "explain" }
      ],
      "learned_at": { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28" }
    },
    "transaction-acid": {
      "status": "touched",
      "encounters": [
        { "feature": "007-rma-bulk-create-custom-factor-upsert", "date": "2026-05-28", "via": "map" }
      ]
    }
  }
}
```

Rules:
- `status`: `"touched"` or `"learned"` only
- `encounters`: deduplicated by (feature + via) pair
- Concepts absent from JSON = untouched
- `learned_at` only present when `status` is `"learned"`

---

## Integration with speckit workflow

This skill is **non-invasive** — it does NOT modify any speckit skills. Instead, use it as a companion:

### Recommended workflow

Follows the team's RD development workflow (see `carbonx-spec-hub/docs/rd-development-workflow.md`):

```
Step 1: PM spec.md                              [Spec-Hub]
Step 2: Architect sequence + API contracts       [mint-tea]
Step 3: specify init . --ai claude --force       [Code Repo]
Step 4: /speckit.plan    → plan.md               [Code Repo]
Step 5: /speckit.tasks   → tasks.md              [Code Repo]
        /be-learning map    → be-learning-map.json  ← HERE
        /be-learning explain X → deep-dive          ← HERE
Step 6: /speckit.implement                       [Code Repo]
        /be-learning calibrate                      ← HERE (post-implementation)
        /be-learning progress                       ← HERE (review & mark learned)
```

### Multi-repo usage

When a feature spans multiple repos (e.g., cream BFF + madeleine Domain + souffle MFE):

- Each backend RD runs `/be-learning map` independently **in their own repo**
- The same feature will produce **different** learning maps per repo — this is expected:
  - **BFF (cream)**: likely matches api-design, auth, request-lifecycle
  - **Domain (madeleine)**: likely matches transaction-acid, query-execution, design-patterns
- Frontend repos (souffle, pudding) should **not** use this skill — the 26 concepts are all backend-focused
- Each repo's `specs/{feature}/be-learning-map.json` is independent

### Incremental development

When PM updates an existing spec and provides `spec_update.diff`:

- RD runs `/speckit.plan --context specs/{feature}/spec_update.diff` for incremental planning
- `/be-learning map` detects `spec_update.diff` in the feature directory and switches to **incremental mode**:
  - Focuses on concepts introduced by the **change**, not the entire feature
  - Tags each concept as `"source": "new"` or `"source": "existing"` in the output
  - Only shows `new` concepts in the terminal report (existing ones are collapsed)
- This prevents re-surfacing concepts the engineer already learned from the prior implementation

### Implement-time prompts (optional)

If the user wants brief learning hints during `/speckit.implement`, add the CLAUDE.md snippet. This is opt-in and can be removed by deleting the snippet from CLAUDE.md:

```markdown
## BE Learning Prompts (opt-in)

When implementing a feature via `/speckit.implement`, if a `be-learning-map.json` file exists
in the current feature's `specs/{feature}/` directory, show a brief learning hint at the start
of each task that matches a BE concept. Format:

Learning hint: This task involves [concept name] — [one-sentence why from roadmap].
Run `/be-learning explain {concept-id}` to learn more.

Rules:
- Only show for HIGH confidence matches (not medium/low)
- One hint per task, max — do not interrupt the implementation flow
- Never show hints if `be-learning-map.json` does not exist
- To disable: delete this section from CLAUDE.md
```

---

## Notes

- All explanations use zh-TW to match the user's language preference
- The roadmap JSON is the single source of truth for concept definitions and matching rules
- Calibration is cumulative — each feature calibration improves future predictions
- The skill never modifies speckit workflow files or other team skills
- Senior engineers can skip this skill entirely — it's purely additive
- **Backend only**: The 26 concepts cover NestJS, TypeORM, PostgreSQL, Bull, etc. Frontend concepts (React, MFE, CSS) are not included

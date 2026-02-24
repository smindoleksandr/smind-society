# UDM Benchmark — User Data Model Quality Standard

> **Version:** 1.1.0
> **Last updated:** 2026-02-24
> **Scope:** `user_model_profile_as_items`, `user_model_scheduling_constraints_and_preferences`, `user_data_history`
> **Consumer:** Evaluator job (`evaluator_udm`) — an LLM that grades actual data against these criteria.

---

## Changelog

### v1.1.0 (2026-02-24)
- Added §1.6 (time box overlap ban for `user_data_history`)
- Added §2.4 (key length bounds)
- Added §3.5 (conditional/contingent nature of scheduling constraints must be explicit)
- Added §3.6 (description granularity guidance for `user_data_history`)
- Added §5.5 (timestamps must not be in the future for `user_data_history`)
- Added §6.5 (evidence element schema — required and optional fields)
- Added §6.6 (conflicting evidence must be reconciled)
- Added §7.4 (tag consistency across tables — same taxonomy)
- Added §8.4 (history coverage for past scheduling constraints)
- Refined §5.2 (timezone default behavior — made pass/fail crisper)
- Refined §9.1 (stale constraints — added grace window and Evaluator instructions)
- Added Appendix C (evidence element schema reference)
- Noted §5.4 as intentional overlap with §1.3 scoped to scheduling table

### v1.0.0 (2026-02-20)
- Initial benchmark.

---

## How to Use This Benchmark

Each section defines a **quality dimension** with:
- **RULE**: A concrete, testable criterion.
- **PASS**: What compliance looks like.
- **FAIL**: What a violation looks like, with examples.
- **Severity**: `critical` | `high` | `medium` | `low` — guides issue priority.

The Evaluator should check every row in each table against every applicable rule and file GitHub issues for failures.

**Table abbreviations used below:**
- **PROFILE** = `user_model_profile_as_items`
- **SCHED** = `user_model_scheduling_constraints_and_preferences`
- **HISTORY** = `user_data_history`

---

## 1. Schema Compliance

### 1.1 Primary Key Integrity
**Applies to:** PROFILE, SCHED, HISTORY
**RULE:** Every row must have a non-empty `key`.
**PASS:** `key` is a non-empty string, unique within the table.
**FAIL:** `key` is null, empty string, or whitespace-only.
**Severity:** critical

### 1.2 Required Fields Non-Null
**RULE:** Required fields per schema must not be null or empty:
- PROFILE: `title`
- SCHED: `title`
- HISTORY: `start_ts`, `end_ts`, `description`

**PASS:** All required fields are populated with meaningful content.
**FAIL:** Any required field is null, empty, or whitespace-only.
**Severity:** critical

### 1.3 Time Order Constraint (HISTORY)
**RULE:** `end_ts` must be strictly after `start_ts`.
**PASS:** `end_ts > start_ts` for every row.
**FAIL:** `end_ts <= start_ts`.
**Severity:** critical

### 1.4 Evidence Is Array
**Applies to:** PROFILE, SCHED, HISTORY
**RULE:** `evidence` column must be a JSON array (not null, not an object, not a string).
**PASS:** `evidence` parses as a JSON array (including empty `[]`).
**FAIL:** `evidence` is null, a bare string, a JSON object, or malformed JSON.
**Severity:** critical

### 1.5 Tags Is Array
**Applies to:** PROFILE, SCHED
**RULE:** `tags` column must be a text array with no null elements.
**PASS:** `tags` is a valid Postgres text array (including empty `{}`).
**FAIL:** `tags` is null or contains null elements.
**Severity:** high

### 1.6 No Overlapping Time Boxes (HISTORY)
**RULE:** No two rows in `user_data_history` should have overlapping `[start_ts, end_ts)` intervals, unless they represent genuinely concurrent activities (e.g., "listening to podcast" while "commuting"). If concurrent, descriptions must make this clear.
**PASS:** Time boxes are non-overlapping, or overlap is explicitly justified by concurrent activities in both descriptions.
**FAIL:** Two rows cover `09:00–10:00` and `09:30–11:00` with no indication of concurrency (likely a data entry error or duplicate).
**Severity:** high

---

## 2. Key Quality

### 2.1 Key Format
**RULE:** Keys should be human-readable, snake_case or kebab-case identifiers that hint at the content. They serve as both primary key and semantic anchor.
**PASS:** `fear_of_flying`, `weekly-gym-routine`, `2025-01-15-dentist-appointment`
**FAIL:** `item_1`, `abc123`, `untitled`, `asdf`, UUIDs used as keys.
**Severity:** medium

### 2.2 Key Uniqueness Semantics
**RULE:** Two rows should not represent the same real-world fact, event, or activity. If they do, one should be deleted or they should be merged.
**PASS:** One row per distinct fact/event.
**FAIL:** `fear_of_flying` and `afraid_of_planes` both exist with overlapping content. Or `dentist_jan_2025` and `dentist_appointment_january` covering the same appointment.
**Severity:** high

### 2.3 Key Stability
**RULE:** Keys should not encode volatile information (e.g., current date in a key for a recurring event, sequence numbers that shift).
**PASS:** `weekly_cage_cleaning` for a recurring task.
**FAIL:** `weekly_cage_cleaning_2025_02_15` for a recurring pattern (date makes it seem one-time).
**Severity:** low

### 2.4 Key Length
**RULE:** Keys should be between 3 and 80 characters. Shorter keys lack semantic content; longer keys become unwieldy and likely contain embedded sentences.
**PASS:** `morning_gym_routine` (19 chars)
**FAIL:** `a` (too short), `the_user_prefers_to_go_to_the_gym_in_the_morning_on_mondays_wednesdays_and_fridays` (82 chars — this is a sentence, not a key).
**Severity:** low

---

## 3. Content Quality

### 3.1 Title Clarity
**RULE:** `title` must be a concise, self-explanatory label. A reader should understand what the row is about from the title alone, without reading `summary`.
**PASS:** "Fear of flying", "Weekly gym routine — Mon/Wed/Fri", "Dentist appointment 2025-01-15"
**FAIL:** "Thing", "Note", "Important", "Untitled", "TODO"
**Severity:** high

### 3.2 Summary Informativeness
**RULE:** `summary` (when present) should add context beyond the title. It should contain specific details: dates, frequencies, conditions, quantities, context. If `summary` merely restates the `title`, it adds no value.
**PASS:** Title: "Fear of flying" → Summary: "the user experienced a scary turbulence event on a flight around 2018. Avoids flights when possible; prefers trains for European travel. Will fly if necessary but with significant anxiety."
**FAIL:** Title: "Fear of flying" → Summary: "the user is afraid of flying." (just restates title)
**Severity:** medium

### 3.3 Summary Accuracy (No Editorializing)
**RULE:** Summaries must faithfully represent what the user said. No positive spin, no softening of negative sentiment, no editorializing.
**PASS:** "Position is brittle and tenuous" if that's what the user said.
**FAIL:** "Navigating a dynamic career opportunity" when the user said their position is brittle.
**Severity:** high

### 3.4 Description Specificity (HISTORY)
**RULE:** `description` should describe what actually happened during the time box, with enough detail for a future reader to understand the activity.
**PASS:** "Worked on SMind society architecture — defined Evaluator job spec, wrote benchmark template"
**FAIL:** "Work", "Stuff", "Meeting"
**Severity:** medium

### 3.5 Conditional Nature Must Be Explicit (SCHED)
**RULE:** Per schema, "all events are conditional in some way." If a scheduling constraint is contingent on a condition (e.g., "only if the weather is good", "if the visa is approved"), the condition must be stated in `summary`. Unconditional future appointments should still note their certainty level.
**PASS:** Summary: "Dentist appointment, confirmed." or Summary: "Beach trip — weather permitting, planning to go if Saturday is sunny."
**FAIL:** A vague constraint with `starts_at` set but no summary explaining the conditions or certainty, leaving the Scheduler unable to judge reliability.
**Severity:** medium

### 3.6 History Granularity (HISTORY)
**RULE:** Time boxes in `user_data_history` should represent meaningful activity blocks, not micro-events (< 5 minutes) unless genuinely significant, and not mega-blocks (> 24 hours) unless they represent a single continuous activity (e.g., a multi-day trip). A day of work should be broken into meaningful segments, not stored as a single 10-hour block.
**PASS:** "09:00–12:30 Deep work on project X", "12:30–13:30 Lunch", "13:30–17:00 Meetings"
**FAIL:** A single row "09:00–19:00 Work" for an entire day. Or "14:05–14:07 Checked phone."
**Severity:** low

---

## 4. Table Assignment (Correct Routing)

### 4.1 Profile Items vs Scheduling Constraints
**RULE:** Data must be in the correct table:
- **SCHED**: Future-applicable constraints — appointments with dates, recurring patterns, commitments, conditional future constraints.
- **PROFILE**: Stable background context — facts, preferences, habits, fears, traits, historical incidents that don't impose a future scheduling constraint.

**PASS:** "Dentist appointment on 2025-03-10" → SCHED. "Has two parrots" → PROFILE.
**FAIL:** A dentist appointment stored in PROFILE. A personality trait stored in SCHED.
**Severity:** high

### 4.2 Historical vs Future
**RULE:** Purely historical incidents that impose NO future constraint belong in PROFILE (as background context), NOT in SCHED.
**PASS:** "Moved from Ukraine to Portugal in 2022" → PROFILE.
**FAIL:** "Moved from Ukraine to Portugal in 2022" → SCHED (no future scheduling implication).
**Severity:** high

### 4.3 Profile Items vs User Data History
**RULE:** HISTORY stores time-boxed activity records (what happened in a specific time window). PROFILE stores enduring facts/preferences. A specific gym session goes in history; the fact that the user exercises regularly goes in profile items.
**PASS:** "Gym session 2025-02-10 07:00–08:30" → HISTORY. "Exercises 3x/week, prefers morning sessions" → PROFILE.
**FAIL:** A general preference stored in HISTORY with fabricated timestamps. A specific dated event stored only as a profile item.
**Severity:** high

---

## 5. Timestamp Quality (SCHED)

### 5.1 No Fabricated Precision
**RULE:** Timestamps must not be more precise than what the user provided:
- Date without time → `starts_at` at midnight, summary says "time unknown"
- Year only → `starts_at = YYYY-01-01`, `ends_at = (YYYY+1)-01-01`, summary says "approximate"
- Month+year → `starts_at = YYYY-MM-01`, `ends_at = YYYY-(MM+1)-01`, summary says "approximate"

**PASS:** User said "sometime in March 2025" → `starts_at = 2025-03-01`, `ends_at = 2025-04-01`, summary: "approximate, March 2025"
**FAIL:** User said "sometime in March 2025" → `starts_at = 2025-03-15T10:00:00Z` (fabricated day and time).
**Severity:** high

### 5.2 Timezone Presence
**RULE:** If the user specified or implied a timezone, `timezone` must be populated. For events that are clearly local to the user (e.g., "dentist at 3pm"), default to the user's known timezone (`Europe/Lisbon`). Only leave `timezone` null for events where locality is genuinely ambiguous (e.g., an online meeting with no timezone specified).
**PASS:** `timezone: "Europe/Lisbon"` for "dentist at 3pm tomorrow".
**FAIL:** `timezone: null` for "dentist at 3pm tomorrow" — this is a local event, timezone is known.
**Severity:** medium

### 5.3 Recurring Events: Summary Describes Pattern
**RULE:** For recurring events, the recurrence pattern must be described in the `summary` field since there are no dedicated recurrence columns. The `starts_at`/`ends_at` fields may represent the first occurrence or be null, but the summary must clearly state the recurrence.
**PASS:** Summary: "Weekly gym session, Mon/Wed/Fri mornings. Started January 2025."
**FAIL:** A recurring event with `starts_at` for one occurrence and no mention of recurrence in summary.
**Severity:** high

### 5.4 Time Order for Scheduling Constraints
**RULE:** When both `starts_at` and `ends_at` are present, `ends_at` must be after `starts_at`. _(Note: this mirrors §1.3 but applies to SCHED, which has no DB-level time order constraint.)_
**PASS:** `starts_at < ends_at`.
**FAIL:** `starts_at >= ends_at`.
**Severity:** critical

### 5.5 No Future Timestamps in History
**Applies to:** HISTORY
**RULE:** `start_ts` and `end_ts` in `user_data_history` must not be in the future. History records what happened, not what will happen. (A small tolerance of ~1 hour is acceptable for timezone edge cases.)
**PASS:** All timestamps are in the past or very recent present.
**FAIL:** A history entry with `start_ts` three weeks from now.
**Severity:** high

---

## 6. Evidence Quality

### 6.1 Evidence Present for Non-Obvious Facts
**RULE:** Every item or constraint derived from user conversation should have at least one evidence entry. The only exception is seed data or data so foundational it needs no citation (e.g., the user's name).
**PASS:** `evidence: [{"source": "file:/root/.openclaw/agents/main/sessions/abc123.jsonl#L42", "content": "I'm terrified of flying"}]`
**FAIL:** `evidence: []` for a specific claim like "allergic to shellfish" that must have come from somewhere.
**Severity:** high

### 6.2 Evidence Source Format
**RULE:** The primary `source` field must point to OpenClaw session logs:
- Format: `file:/root/.openclaw/agents/main/sessions/<sessionId>.jsonl#L<line>`
- External IDs (Telegram message_id, etc.) may appear in a secondary field like `legacy_source` but NOT as the primary `source`.

**PASS:** `{"source": "file:/root/.openclaw/agents/main/sessions/sess_abc.jsonl#L15", "content": "quoted text"}`
**FAIL:** `{"source": "telegram:msg_12345"}` as the only source.
**Severity:** high

### 6.3 Evidence Content Field
**RULE:** Each evidence entry should include a `content` field with a direct quote or close paraphrase from the user's message. This makes evidence self-contained.
**PASS:** `{"source": "...", "content": "I usually go to the gym Mon, Wed, and Fri mornings"}`
**FAIL:** `{"source": "..."}` with no `content` field, or `{"content": "user mentioned gym"}` (vague paraphrase).
**Severity:** medium

### 6.4 Evidence Freshness
**RULE:** Evidence should reference real, existing session files. Fabricated session IDs or line numbers are worse than no evidence.
**PASS:** The session file referenced actually exists (Evaluator may spot-check).
**FAIL:** A session ID that follows no known pattern or references a clearly nonexistent file.
**Severity:** high

### 6.5 Evidence Element Schema
**RULE:** Each element in the `evidence` JSON array must be an object. Required fields: `source` (string). Recommended fields: `content` (string — user quote). Optional fields: `legacy_source` (string — external message ID), `extracted_at` (ISO 8601 timestamp — when the evidence was recorded). No other arbitrary fields should be added.
**PASS:** `{"source": "file:...", "content": "I have two parrots"}`
**FAIL:** `"I have two parrots"` (bare string in array instead of object). Or `{"random_field": 42}` (missing `source`).
**Severity:** high

### 6.6 Conflicting Evidence
**RULE:** If a row has multiple evidence entries that contradict each other (e.g., one says "exercises 3x/week" and another says "stopped going to the gym"), the summary must reflect the most recent evidence, and outdated evidence should be annotated or removed. Unresolved contradictions are a data quality failure.
**PASS:** Evidence array ordered chronologically; summary reflects latest state; older contradictory evidence marked with context (e.g., `"content": "[superseded] used to exercise 3x/week"`).
**FAIL:** Two contradictory evidence entries with no indication of which is current; summary picks one arbitrarily.
**Severity:** medium

---

## 7. Tags Quality

### 7.1 Tags Are Meaningful
**Applies to:** PROFILE, SCHED
**RULE:** Tags should be drawn from a coherent taxonomy and provide useful categorization. Lowercase, consistent naming (snake_case or single words).
**PASS:** `{habits, health, fitness}`, `{travel, preferences}`, `{work, career}`
**FAIL:** `{misc}`, `{tag1}`, `{important}`, `{}` on a row that clearly belongs to a category.
**Severity:** medium

### 7.2 No Tag Proliferation
**RULE:** Tags should converge on a manageable set rather than spawning unique tags per item. If the tag set grows beyond ~30 distinct tags across all items, there's likely duplication or over-specificity.
**PASS:** Consistent reuse of tags like `health`, `career`, `family`, `finance`, `travel`.
**FAIL:** 50+ unique tags with many used only once, like `dentist_2025`, `that_flight_thing`.
**Severity:** low

### 7.3 Habits Tag Convention
**RULE:** Per SCHEMA.md, habits are stored as profile items with the `habits` tag. Any habit-related item must include `habits` in its tags array.
**PASS:** "Morning gym routine" with tags `{habits, fitness, health}`.
**FAIL:** "Morning gym routine" with tags `{fitness, health}` but missing `habits`.
**Severity:** medium

### 7.4 Tag Consistency Across Tables
**RULE:** PROFILE and SCHED should use the same tag vocabulary. A concept tagged `fitness` in PROFILE should not be tagged `exercise` in SCHED for the same domain. The Evaluator should collect the distinct tag set from both tables and flag divergent synonyms.
**PASS:** Both tables use `health` for health-related items.
**FAIL:** PROFILE uses `health` and SCHED uses `medical` for the same category of items.
**Severity:** low

---

## 8. Cross-Table Consistency

### 8.1 No Orphaned References
**RULE:** If a profile item references a scheduling constraint (or vice versa) by key, that referenced key must exist.
**PASS:** Profile item summary says "see also: weekly_gym_routine" and that key exists in scheduling constraints.
**FAIL:** Reference to a key that doesn't exist in any table.
**Severity:** medium

### 8.2 No Contradictions Between Tables
**RULE:** Information in PROFILE should not contradict information in SCHED or HISTORY. If the user changed their mind, the old entry should be updated or removed.
**PASS:** Profile says "exercises 3x/week" and scheduling has a matching recurring constraint.
**FAIL:** Profile says "doesn't exercise" but scheduling has active gym appointments.
**Severity:** high

### 8.3 History Aligns with Constraints
**RULE:** HISTORY entries for recurring activities should be consistent with any recurring patterns described in SCHED. Occasional misses are fine, but systematic divergence suggests stale data.
**PASS:** SCHED says "gym Mon/Wed/Fri" and HISTORY shows gym entries roughly matching that pattern.
**FAIL:** SCHED says "daily meditation 6am" but HISTORY has zero meditation entries over months. (Constraint may be aspirational rather than actual — should be flagged for review.)
**Severity:** low

### 8.4 Past Constraints Have History Coverage
**RULE:** If a scheduling constraint's `starts_at` is in the past and it was a one-time event, there should ideally be a corresponding entry in HISTORY recording what happened. This is informational — not all events get logged — but systematic absence of history for past events suggests the history table is under-populated.
**PASS:** SCHED has "dentist 2025-12-01" (past) and HISTORY has a matching entry around that date.
**FAIL:** 10 past one-time events in SCHED, 0 corresponding HISTORY entries. (Flag as potential data gap.)
**Severity:** low (informational)

---

## 9. Data Hygiene

### 9.1 No Stale Scheduling Constraints
**RULE:** One-time scheduling constraints whose date has passed by more than **7 days** should either be removed or moved to HISTORY. A 7-day grace window allows for delayed cleanup. Recurring constraints never become stale by date alone.
**PASS:** Past one-time events cleaned up within ~7 days of passing; only future/recurring constraints remain.
**FAIL:** Multiple one-time constraints dated > 7 days ago still present.
**Severity:** medium

### 9.2 No Duplicate Content
**RULE:** Two rows should not express the same fact, even with different keys. This applies within each table and across tables (per §4 routing rules).
**PASS:** One authoritative row per fact.
**FAIL:** `fear_of_flying` and `flight_anxiety` both describing the same fear with overlapping content.
**Severity:** high

### 9.3 No Empty/Placeholder Rows
**RULE:** Every row should contain real, meaningful data. No placeholder entries, test data, or rows with only a key and defaults.
**PASS:** Every row has substantive content in title/summary/description.
**FAIL:** A row with `title: "test"`, `summary: null`, `tags: {}`, `evidence: []`.
**Severity:** high

---

## 10. Completeness Indicators

> These are NOT hard failures — they are signals for the Evaluator to flag as potential data gaps (which may feed into `smind_data_gap_questions`). The Evaluator should note these but at lower severity.

### 10.1 Life Dimension Coverage
**RULE:** The user model should have at least some items covering major life dimensions: health, career/work, relationships/family, finance, personal development, living situation. Absence of an entire dimension suggests data gaps.
**Severity:** low (informational)

### 10.2 Evidence Coverage Ratio
**RULE:** The percentage of items with non-empty evidence should be tracked. A ratio below 50% suggests systematic evidence omission.
**Severity:** low (informational)

### 10.3 Recency
**RULE:** If all items have evidence pointing to sessions older than 90 days and no recent updates, the model may be going stale.
**Severity:** low (informational)

---

## Appendix A: Evaluation Procedure for Evaluator Job

When evaluating, the Evaluator should:

1. **Scan all rows** in each of the three tables.
2. **Check each row** against all applicable rules from this benchmark.
3. **Group failures** by rule, not by row — one GitHub issue per rule violation pattern, listing all affected rows.
4. **Set issue priority** based on the severity of the violated rule.
5. **Include actionable fix descriptions** — what specifically should change (e.g., "merge rows X and Y", "add evidence to row Z", "move row W from profile to scheduling").
6. **Do NOT modify data** — only file issues. The User Data Maintainer worker handles fixes.

## Appendix B: Severity Mapping to GitHub Issue Labels

| Severity | GitHub Label | Expected Response Time |
|----------|-------------|----------------------|
| critical | `priority: critical` | Must fix before next pipeline run |
| high | `priority: high` | Fix within 1-2 pipeline cycles |
| medium | `priority: medium` | Fix when convenient |
| low | `priority: low` | Nice to have / informational |

## Appendix C: Evidence Element Schema Reference

```json
{
  "source": "file:/root/.openclaw/agents/main/sessions/<sessionId>.jsonl#L<line>",
  "content": "Direct quote or close paraphrase from user message",
  "legacy_source": "telegram:msg_12345",
  "extracted_at": "2025-06-15T14:30:00Z"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `source` | string | **Yes** | OpenClaw session log reference |
| `content` | string | Recommended | User quote making evidence self-contained |
| `legacy_source` | string | Optional | External message ID (Telegram, etc.) |
| `extracted_at` | string (ISO 8601) | Optional | When the evidence was captured |

---

*This benchmark is self-improving. Each run of the Scientist job should review and enhance it.*

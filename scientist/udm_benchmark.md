# UDM Benchmark — User Data & Model Quality Standard

> **Version:** 1.0.0 · **Created:** 2026-02-24 · **Author:** 🔬 Scientist UDM
>
> This benchmark defines what high-quality data looks like in three tables:
> `user_model_profile_as_items`, `user_model_scheduling_constraints_and_preferences`, `user_data_history`.
>
> **Audience:** The Evaluator LLM job consumes this benchmark to grade actual data. Criteria must be specific, concrete, and actionable.

---

## How to Use This Benchmark

The Evaluator should check every row (or a representative sample) against ALL applicable criteria below. Each criterion is either **PASS** or **FAIL** per row. A criterion marked 🔴 is critical (should always be filed as an issue); 🟡 is important; 🟢 is advisory.

---

## 1 · `user_model_profile_as_items`

Schema recap: `key` (PK text), `title` (text, not null), `summary` (text, nullable), `tags` (text[], default `{}`), `evidence` (jsonb array, default `[]`).

### 1.1 Key Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-KEY-1 | **Meaningful key** | 🔴 | `key` must be a human-readable slug that summarizes the item (e.g. `fear-of-public-speaking`, `morning-person`). Keys like `item_1`, `abc123`, UUIDs, or generic labels are failures. |
| P-KEY-2 | **Consistent key format** | 🟡 | Keys must use lowercase `kebab-case`. No spaces, underscores, camelCase, or mixed case. |
| P-KEY-3 | **No semantic duplicates** | 🔴 | Two rows must not describe the same underlying fact. E.g. `likes-coffee` and `coffee-lover` covering the same preference = duplicate. The Evaluator should group suspicious pairs and flag them. |

### 1.2 Title Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-TIT-1 | **Self-contained title** | 🔴 | `title` alone must convey the item's meaning without needing to read `summary`. It should be a concise noun phrase or statement (e.g. "Afraid of public speaking", "Prefers mornings for deep work"). |
| P-TIT-2 | **No filler or meta-language** | 🟡 | Title must not contain meta-commentary like "User says…", "Note:", "TODO", or "Important:". Just the fact. |
| P-TIT-3 | **Reasonable length** | 🟢 | Title should be 3–80 characters. Shorter is likely too vague; longer should be in summary. |

### 1.3 Summary Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-SUM-1 | **Adds value beyond title** | 🟡 | If `summary` is present, it must contain information not already in `title`. A summary that just rephrases the title is wasted space. If there's nothing to add, `null` is preferred over a repetitive summary. |
| P-SUM-2 | **Factual, not speculative** | 🔴 | Summary must state observed/reported facts, not assistant speculation. E.g. ✅ "Mentioned disliking crowded places on 2025-12-01" vs ❌ "Probably introverted based on general patterns". |
| P-SUM-3 | **No stale or contradicted info** | 🔴 | If a summary references a state that has been explicitly updated or contradicted by newer evidence, it must be updated or the row replaced. |

### 1.4 Tags Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-TAG-1 | **At least one tag** | 🔴 | Every row must have ≥1 tag. Empty `tags = '{}'` is a failure — the item is uncategorized and invisible to tag-based queries. |
| P-TAG-2 | **Tags are from a coherent vocabulary** | 🟡 | Tags should be lowercase, singular nouns or short phrases (e.g. `habits`, `values`, `fears`, `preferences`, `health`, `work`, `relationships`). Avoid one-off tags that appear on only one item with no clear semantic meaning. |
| P-TAG-3 | **No redundant tags** | 🟢 | Avoid tag pairs where one strictly subsumes the other on the same row (e.g. `food` + `food-preferences` — pick one). |
| P-TAG-4 | **Habits tag used correctly** | 🟡 | Items tagged `habits` must describe a recurring behavior pattern, not a one-time event or a preference. |

### 1.5 Evidence Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-EVI-1 | **Evidence is a JSON array** | 🔴 | Must be a valid JSON array (enforced by DB constraint, but check for malformed entries). |
| P-EVI-2 | **At least one evidence entry** | 🟡 | Every profile item should have ≥1 evidence entry linking back to where this fact was learned. Items with `[]` evidence are unverifiable. |
| P-EVI-3 | **Evidence entries are structured** | 🟡 | Each evidence entry should contain at minimum a source reference (date, conversation snippet, or document path). Free-text blobs with no structure reduce traceability. |
| P-EVI-4 | **Evidence supports the claim** | 🔴 | The evidence must actually support what the title/summary claims. Mismatched evidence (e.g. evidence about cooking on a row about exercise habits) is a critical failure. |

### 1.6 Semantic & Coverage Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| P-SEM-1 | **Atomic items** | 🔴 | Each row should represent ONE fact/preference/habit. A row that bundles multiple unrelated facts (e.g. "Likes coffee, afraid of dogs, works at Google") must be split. |
| P-SEM-2 | **No orphaned items** | 🟡 | Items should not be completely disconnected from all other items — if an item has unique tags shared by no other row and no evidence, it may be a data entry error. |
| P-SEM-3 | **Appropriate granularity** | 🟡 | Items should be specific enough to be actionable. "Likes food" is too vague. "Prefers spicy Thai food for dinner" is good. |

---

## 2 · `user_model_scheduling_constraints_and_preferences`

Schema recap: `key` (PK text), `title` (text, not null), `summary` (text, nullable), `starts_at` (timestamptz, nullable), `ends_at` (timestamptz, nullable), `timezone` (text, nullable), `tags` (text[], default `{}`), `evidence` (jsonb array, default `[]`).

### 2.1 Key Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-KEY-1 | **Meaningful key** | 🔴 | Same as P-KEY-1. Must be a descriptive slug. |
| S-KEY-2 | **Consistent key format** | 🟡 | Same as P-KEY-2. Lowercase kebab-case. |
| S-KEY-3 | **No semantic duplicates** | 🔴 | Same as P-KEY-3. Two rows must not describe the same constraint/event. |

### 2.2 Title Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-TIT-1 | **Self-contained title** | 🔴 | Same standard as P-TIT-1. |
| S-TIT-2 | **Indicates constraint type** | 🟡 | Title should make clear whether this is a hard commitment ("Dentist appointment"), a recurring pattern ("Gym every MWF"), or a soft preference ("Prefers no meetings before 10am"). |

### 2.3 Temporal Fields Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-TIME-1 | **Fixed events must have timestamps** | 🔴 | If the row describes a specific date/time event (appointment, flight, deadline), `starts_at` must be populated. If it has a known duration, `ends_at` must also be set. |
| S-TIME-2 | **Recurring patterns must NOT rely on starts_at/ends_at** | 🟡 | Recurring patterns (e.g. "Gym every Monday") should encode recurrence in `title`/`summary`, not as a single timestamp. `starts_at`/`ends_at` may optionally mark the recurrence window (first/last occurrence), but the pattern itself must be in text. |
| S-TIME-3 | **ends_at > starts_at** | 🔴 | DB constraint enforced, but logically: if both are set, end must be after start. Zero-duration events are suspicious. |
| S-TIME-4 | **Timezone populated for time-specific events** | 🟡 | If `starts_at` is set, `timezone` should be populated to disambiguate. Null timezone on a timestamped event means the reader must guess. |
| S-TIME-5 | **No stale past events** | 🔴 | One-off events whose `ends_at` is in the past (and not recurring) should be archived or removed. They clutter the scheduling constraint space. Evaluator: flag any non-recurring row with `ends_at` > 30 days in the past. |
| S-TIME-6 | **Vague-date items use summary** | 🟡 | If an event has no exact date ("sometime in March", "after the project ends"), `starts_at` should be null and the vague timing must be described in `summary`. Do NOT fabricate a timestamp for vague dates. |

### 2.4 Tags Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-TAG-1 | **At least one tag** | 🔴 | Same as P-TAG-1. |
| S-TAG-2 | **Distinguishes constraint vs preference** | 🟡 | Tags should include at least one of: `hard-constraint`, `soft-preference`, `recurring`, `one-off`, `deadline`. This enables downstream filtering by the Scheduler. |

### 2.5 Evidence Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-EVI-1 | **Evidence is a JSON array** | 🔴 | Same as P-EVI-1. |
| S-EVI-2 | **At least one evidence entry** | 🟡 | Same as P-EVI-2. |
| S-EVI-3 | **Evidence supports the constraint** | 🔴 | Same as P-EVI-4. |

### 2.6 Semantic Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| S-SEM-1 | **Atomic constraints** | 🔴 | Each row = one constraint or preference. "No meetings before 10am and gym every MWF" must be two rows. |
| S-SEM-2 | **Actionable for scheduling** | 🟡 | Each row must contain enough information for the Scheduler to act on it. A row that says "busy sometimes" with no further detail is useless. |
| S-SEM-3 | **No profile items masquerading as constraints** | 🟡 | If a row is really a personality trait or preference with no scheduling implication (e.g. "Likes the color blue"), it belongs in `user_model_profile_as_items`, not here. |

---

## 3 · `user_data_history`

Schema recap: `key` (PK text), `start_ts` (timestamptz, not null), `end_ts` (timestamptz, not null), `description` (text, not null), `evidence` (jsonb array, default `[]`).

### 3.1 Key Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| H-KEY-1 | **Meaningful key** | 🔴 | Must describe the activity + approximate time. E.g. `gym-session-2025-12-01`, `dentist-visit-2025-11-15`. Pure timestamps or UUIDs fail. |
| H-KEY-2 | **Consistent key format** | 🟡 | Lowercase kebab-case, ideally `{activity}-{date}` pattern. |
| H-KEY-3 | **No duplicates** | 🔴 | No two rows should describe the same real-world event. |

### 3.2 Temporal Fields Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| H-TIME-1 | **Both timestamps required** | 🔴 | `start_ts` and `end_ts` are NOT NULL by schema. Both must be meaningful — not placeholder values like epoch 0 or year 9999. |
| H-TIME-2 | **end_ts > start_ts** | 🔴 | DB constraint. Logically, duration must be positive and reasonable for the described activity. A "gym session" lasting 0.1 seconds or 48 hours is suspicious. |
| H-TIME-3 | **Reasonable duration** | 🟡 | Duration should match the activity type. Flag entries where duration is implausibly short (<1 min for most activities) or implausibly long (>24h for a single activity unless it's travel/sleep). |
| H-TIME-4 | **No future timestamps** | 🔴 | This is a HISTORY table. Both `start_ts` and `end_ts` must be in the past (relative to evaluation time). Future entries belong in `user_model_scheduling_constraints_and_preferences`. |
| H-TIME-5 | **Chronological consistency** | 🟡 | Overlapping time boxes for the same person should be rare and justified (e.g. listening to a podcast while commuting is fine). Flag cases where the same kind of activity overlaps itself (e.g. two "gym sessions" at the same time). |

### 3.3 Description Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| H-DESC-1 | **Describes what happened** | 🔴 | Description must state the activity that occurred. Not a plan, not a wish — what actually happened. |
| H-DESC-2 | **Specific enough to be useful** | 🟡 | "Did stuff" fails. "30 min run in the park" passes. Include relevant details: what, where, with whom (if relevant), outcome. |
| H-DESC-3 | **No scheduling language** | 🟡 | Descriptions should not say "planned to" or "should have" — this is history, not intentions. |
| H-DESC-4 | **Reasonable length** | 🟢 | 10–500 characters is typical. Very short descriptions lack context; very long ones may be pasting raw data. |

### 3.4 Evidence Quality

| # | Criterion | Severity | Rule |
|---|-----------|----------|------|
| H-EVI-1 | **Evidence is a JSON array** | 🔴 | Same as P-EVI-1. |
| H-EVI-2 | **Evidence links to source** | 🟡 | Each entry should reference where this history was captured (conversation date, upload, integration). |

---

## 4 · Cross-Table Quality

These criteria apply across all three tables.

| # | Criterion | Severity | Applies To | Rule |
|---|-----------|----------|------------|------|
| X-1 | **No data in wrong table** | 🔴 | All | Profile facts → `profile_as_items`. Scheduling constraints → `scheduling_constraints_and_preferences`. Past events → `user_data_history`. Misplaced rows must be flagged. |
| X-2 | **Consistent evidence format** | 🟡 | All | Evidence entries across all three tables should follow a consistent internal structure (same keys, same style). |
| X-3 | **Key uniqueness across tables** | 🟢 | All | While not a DB constraint, ideally keys should not collide across tables to avoid confusion in cross-table references. |
| X-4 | **Temporal coherence** | 🟡 | scheduling + history | If a scheduling constraint describes a past one-off event, a corresponding history entry should exist (and vice versa — a history entry for a planned event implies the constraint was fulfilled). Flag orphaned pairs. |
| X-5 | **No empty tables** | 🟡 | All | A completely empty table (0 rows) for an active user is a coverage gap worth flagging. |

---

## 5 · Evaluation Scoring Guide

For the Evaluator:

1. **Per-row evaluation:** Check each row against all applicable criteria for its table.
2. **Severity-based issue filing:**
   - 🔴 Critical failures → File a GitHub issue immediately. Title: `[UDM] {table}: {criterion ID} — {short description}`. Label: `udm`, `critical`.
   - 🟡 Important failures → Group related failures into a single issue per table. Label: `udm`, `important`.
   - 🟢 Advisory — Note in issue body but don't create standalone issues.
3. **Batch duplicates:** If the same criterion fails on 5+ rows, file ONE issue listing all affected keys, not one per row.
4. **Positive signal:** If a table passes all criteria, note that in the evaluation summary — no issue needed.

---

## Appendix A · Evidence Schema Recommendation

While the DB only enforces "must be a JSON array", the recommended structure for each evidence entry is:

```json
{
  "source": "conversation | upload | integration | manual",
  "date": "ISO-8601 date when this evidence was captured",
  "detail": "Brief quote or reference"
}
```

The Evaluator should flag evidence entries that are bare strings, empty objects, or lack any identifiable source reference.

---

## Appendix B · Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-02-24 | Initial benchmark created from schema analysis. 44 criteria across 3 tables + cross-table checks. |

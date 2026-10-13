# results

Part of the **Outcome** section. The final, rolled-up outcome of one submitted session. Dashboards, leaderboards and hiring decisions read from here.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the result |
| `session_id` | UUID | NOT NULL, UNIQUE, FOREIGN KEY → `assessment_sessions.id` | The session this result is for. UNIQUE makes it 1:1. |
| `total_marks` | NUMERIC(8,2) | NULL | Sum of `marks_awarded` over all session tasks. `NULL` while `CALCULATING`. |
| `max_marks` | NUMERIC(8,2) | NOT NULL, CHECK `>= 0` | Sum of `max_marks` over all session tasks |
| `percentage` | NUMERIC(5,2) | NULL, CHECK between `0` and `100` | `total_marks ÷ max_marks × 100`, not below 0. `NULL` while `CALCULATING`. |
| `status` | VARCHAR(20) | NOT NULL, default `CALCULATING`, CHECK in (`CALCULATING`, `READY_FOR_REVIEW`, `PUBLISHED`, `INVALIDATED`) | Lifecycle of the result |
| `section_scores` | JSONB | NULL | Score per section, pre-calculated for dashboards |
| `integrity_summary` | JSONB | NULL | Rolled-up proctoring counts |
| `integrity_flagged` | BOOLEAN | NOT NULL, default `false` | `true` when an integrity limit was crossed |
| `calculated_at` | TIMESTAMPTZ | NULL | When the totals were last calculated |

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `CALCULATING` | Session submitted. Grading is still running. |
| `READY_FOR_REVIEW` | Grading is done. Waiting for a person to review (for example manual or flagged answers) before publishing. |
| `PUBLISHED` | Final. Visible to the participant if the assessment allows it. |
| `INVALIDATED` | Discarded (for example confirmed cheating). Excluded from rankings. |

Transitions: `CALCULATING → READY_FOR_REVIEW → PUBLISHED`. `READY_FOR_REVIEW` or `PUBLISHED → INVALIDATED`. Only HR can publish and only ADMIN can invalidate.

## Pass or fail

There is no pass column. Pass is derived: `percentage >= assessments.passing_percentage`.

## `section_scores` JSONB

Keyed by `assessment_sections.id`, with the title copied in for display.

| Key (per section) | Type | Meaning |
|-------------------|------|---------|
| `title` | string | Section title |
| `score` | number | Marks earned in the section |
| `max` | number | Maximum marks of the section |

## `integrity_summary` JSONB

Counts rolled up from `proctoring_events` when the session ends.

| Key | Meaning |
|-----|---------|
| `tab_switches` | Number of `TAB_SWITCH` events |
| `fullscreen_exits` | Number of `EXIT_FULLSCREEN` events |
| `copy_paste_attempts` | Number of `COPY_PASTE` events |
| `face_not_detected` | Number of `FACE_NOT_DETECTED` events |
| `multiple_faces` | Number of `MULTIPLE_FACES` events |
| `audio_anomalies` | Number of `AUDIO_ANOMALY` events |

## `integrity_flagged`

Set to `true` when a count crosses the limit set in `assessments.settings` (for example `tab_switches` above `max_tab_switches`). It gives HR a quick filter: show only unflagged candidates, ordered by score.

## Sample payload

```json
{
  "id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234",
  "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
  "total_marks": 33.75,
  "max_marks": 53.0,
  "percentage": 63.68,
  "status": "PUBLISHED",
  "section_scores": {
    "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81": { "title": "Core Java MCQ",        "score": 14.25, "max": 23 },
    "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92": { "title": "Coding Challenge",     "score": 12.0,  "max": 20 },
    "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3": { "title": "Database Fundamentals", "score": 7.5,   "max": 10 }
  },
  "integrity_summary": {
    "tab_switches": 2,
    "fullscreen_exits": 0,
    "copy_paste_attempts": 1,
    "face_not_detected": 0,
    "multiple_faces": 0,
    "audio_anomalies": 0
  },
  "integrity_flagged": false,
  "calculated_at": "2026-10-21T05:43:10Z"
}
```

## Sample rows

```json
[
  {
    "id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "total_marks": 33.75,
    "max_marks": 53.0,
    "percentage": 63.68,
    "status": "PUBLISHED",
    "section_scores": {
      "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81": { "title": "Core Java MCQ",        "score": 14.25, "max": 23 },
      "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92": { "title": "Coding Challenge",     "score": 12.0,  "max": 20 },
      "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3": { "title": "Database Fundamentals", "score": 7.5,   "max": 10 }
    },
    "integrity_summary": { "tab_switches": 2, "fullscreen_exits": 0, "copy_paste_attempts": 1, "face_not_detected": 0, "multiple_faces": 0, "audio_anomalies": 0 },
    "integrity_flagged": false,
    "calculated_at": "2026-10-21T05:43:10Z"
  },
  {
    "id": "91d9e0f1-a2b3-4c45-9d67-e8f9a0b1c345",
    "session_id": "51f60718-293a-4b16-8db7-e8091a2b3c4d",
    "total_marks": 9.0,
    "max_marks": 53.0,
    "percentage": 16.98,
    "status": "INVALIDATED",
    "section_scores": {
      "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81": { "title": "Core Java MCQ",        "score": 9.0, "max": 23 },
      "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92": { "title": "Coding Challenge",     "score": 0.0, "max": 20 },
      "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3": { "title": "Database Fundamentals", "score": 0.0, "max": 10 }
    },
    "integrity_summary": { "tab_switches": 7, "fullscreen_exits": 2, "copy_paste_attempts": 0, "face_not_detected": 0, "multiple_faces": 1, "audio_anomalies": 0 },
    "integrity_flagged": true,
    "calculated_at": "2026-10-21T05:24:00Z"
  }
]
```

## Relationships

- One-to-one with `assessment_sessions` (`session_id` → `assessment_sessions.id`).
- One result has many `skill_scores`.

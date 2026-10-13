# assessment_sessions

Part of the **Delivery** section. One row is one live attempt at an assessment. This is the record that proctoring monitors and that tasks, answers and the result hang off.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the session |
| `assignment_id` | UUID | NOT NULL, FOREIGN KEY → `assignments.id` | The assignment this attempt belongs to |
| `attempt_no` | INTEGER | NOT NULL, CHECK `>= 1` | 1 for the first attempt, 2 for the second, and so on |
| `started_at` | TIMESTAMPTZ | NOT NULL | When the participant clicked Start |
| `deadline_at` | TIMESTAMPTZ | NOT NULL, CHECK `deadline_at > started_at` | The server-side end time of the attempt |
| `submitted_at` | TIMESTAMPTZ | NULL | When the attempt ended. `NULL` while it is still running. |
| `submit_type` | VARCHAR(30) | NULL, CHECK in (`MANUAL`, `AUTO_SUBMIT`, `PROCTORING_VIOLATION`, `ADMIN_FORCED`) | How the attempt ended. `NULL` while it is still running. |
| `status` | VARCHAR(20) | NOT NULL, default `ACTIVE`, CHECK in (`ACTIVE`, `PAUSED`, `SUBMITTED`) | Current state of the session |

**Composite constraint:** `UNIQUE (assignment_id, attempt_no)`.

**Consistency rule:** `submitted_at` and `submit_type` are both `NULL` while `status` is `ACTIVE` or `PAUSED`, and both set when `status` is `SUBMITTED`.

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `ACTIVE` | The participant is taking the test. |
| `PAUSED` | The session is frozen (for example a network drop). The timer rules for pauses are decided by the application. |
| `SUBMITTED` | The attempt is locked. Answers can no longer change. |

## Possible values of `submit_type`

| Value | Meaning |
|-------|---------|
| `MANUAL` | The participant clicked Submit. |
| `AUTO_SUBMIT` | Time ran out and the server submitted it. |
| `PROCTORING_VIOLATION` | The system ended the attempt because the integrity limit was crossed (for example `max_tab_switches` in `assessments.settings`). |
| `ADMIN_FORCED` | Staff ended it from the dashboard. |

## Timer rule

`deadline_at` is set by the server when the session starts: `started_at + assessments.duration_minutes`. The browser timer is only a display. A late save after `deadline_at` is rejected and the session is auto-submitted.

## What happens at session start

1. The session row is created.
2. The server creates the `assessment_session_tasks` rows: fixed tasks are copied from `assessment_tasks`, and pooled tasks are picked at random from each pool in `section_pool_rules`.
3. One empty `assessment_session_tasks_answers` row is created for each session task.

## Sample payload

```json
{
  "id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
  "assignment_id": "30c3d4e5-f607-4183-8ca4-b5d6e7f8091a",
  "attempt_no": 1,
  "started_at": "2026-10-21T04:30:12Z",
  "deadline_at": "2026-10-21T06:00:12Z",
  "submitted_at": "2026-10-21T05:41:37Z",
  "submit_type": "MANUAL",
  "status": "SUBMITTED"
}
```

## Sample rows

```json
[
  {
    "id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "assignment_id": "30c3d4e5-f607-4183-8ca4-b5d6e7f8091a",
    "attempt_no": 1,
    "started_at": "2026-10-21T04:30:12Z",
    "deadline_at": "2026-10-21T06:00:12Z",
    "submitted_at": "2026-10-21T05:41:37Z",
    "submit_type": "MANUAL",
    "status": "SUBMITTED"
  },
  {
    "id": "51f60718-293a-4b16-8db7-e8091a2b3c4d",
    "assignment_id": "32c3d4e5-f607-4183-8ca4-b5d6e7f8093c",
    "attempt_no": 1,
    "started_at": "2026-10-21T05:00:00Z",
    "deadline_at": "2026-10-21T06:30:00Z",
    "submitted_at": "2026-10-21T05:22:48Z",
    "submit_type": "PROCTORING_VIOLATION",
    "status": "SUBMITTED"
  }
]
```

## Relationships

- Many sessions belong to one assignment (`assignment_id` → `assignments.id`).
- One session has many `assessment_session_tasks`.
- One session has one `results` row.
- One session has many `proctoring_events`.

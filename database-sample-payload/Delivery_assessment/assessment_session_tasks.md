# assessment_session_tasks

Part of the **Delivery** section. One row is one task given to one participant in one session. It holds the **task information**: which task, where it came from, its position and its locked marking. The participant's answer lives in `assessment_session_tasks_answers`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the session task |
| `session_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_sessions.id`, ON DELETE CASCADE | The session this task belongs to |
| `task_id` | UUID | NOT NULL, FOREIGN KEY → `tasks.id` | The question-bank task that was given |
| `section_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_sections.id` | The section the task appears in |
| `pool_rule_id` | UUID | NULL, FOREIGN KEY → `section_pool_rules.id` | The pool the task was randomly picked from. `NULL` for fixed tasks. |
| `assessment_task_id` | UUID | NULL, FOREIGN KEY → `assessment_tasks.id` | The fixed task row it was copied from. `NULL` for pooled tasks. |
| `display_order` | INTEGER | NOT NULL, CHECK `>= 1` | Position of the task within its section, as this participant saw it |
| `marks` | NUMERIC(6,2) | NOT NULL, CHECK `>= 0` | Marks for a correct answer. Copied from the pool or fixed task at session start. |
| `negative_marks` | NUMERIC(6,2) | NOT NULL, default `0`, CHECK `>= 0` | Marks deducted for a wrong answer. Stored as a positive number. Copied at session start. |

**Constraints:**

- `UNIQUE (session_id, task_id)`. A task appears at most once per session.
- `CHECK ((pool_rule_id IS NULL) <> (assessment_task_id IS NULL))`. Exactly one source is set.

## Why each less obvious attribute exists

| Attribute | Reason |
|-----------|--------|
| `pool_rule_id` | Records why this task was chosen. Admins can group results by pool, for example to see whether the "hard Java" pool is too difficult. |
| `assessment_task_id` | Records which fixed row the task came from. |
| `display_order` | Questions can be shuffled per participant. When someone reports "question 3 is broken", this identifies which task it was for them. |
| `marks`, `negative_marks` | Snapshot. If an admin changes marking later, running and finished sessions are not affected. |

## How rows are created

- **Fixed tasks** (`FIXED` or `MIXED` sections): one row per `assessment_tasks` row, with `assessment_task_id` set.
- **Pooled tasks** (`POOL` or `MIXED` sections): for each pool, `question_count` tasks are picked at random from published tasks matching the pool's skill, type and difficulty, excluding tasks already in this session. `pool_rule_id` is set.
- Rows are created once at session start and never change afterwards.

## Sample payload

```json
{
  "id": "40d4e5f6-0718-4294-9db5-c6e7f8091a2b",
  "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
  "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b",
  "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
  "pool_rule_id": "6a7b8c9d-0e1f-4a2b-8c3d-4e5f6a7b8c9d",
  "assessment_task_id": null,
  "display_order": 1,
  "marks": 1.0,
  "negative_marks": 0.25
}
```

## Sample rows

```json
[
  {
    "id": "40d4e5f6-0718-4294-9db5-c6e7f8091a2b",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b",
    "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
    "pool_rule_id": "6a7b8c9d-0e1f-4a2b-8c3d-4e5f6a7b8c9d",
    "assessment_task_id": null,
    "display_order": 1,
    "marks": 1.0,
    "negative_marks": 0.25
  },
  {
    "id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c",
    "section_id": "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92",
    "pool_rule_id": "8c9d0e1f-2a3b-4c4d-8e5f-6a7b8c9d0e1f",
    "assessment_task_id": null,
    "display_order": 1,
    "marks": 20.0,
    "negative_marks": 0.0
  },
  {
    "id": "42f60718-293a-4b16-8ce8-f9091a2b3c4d",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f",
    "section_id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
    "pool_rule_id": null,
    "assessment_task_id": "a0b1c2d3-e4f5-4a6b-8c7d-9e0f1a2b3c4d",
    "display_order": 1,
    "marks": 2.0,
    "negative_marks": 0.5
  }
]
```

The three rows are a sample. A full session has one row per task, for example 12 in this assessment.

## Relationships

- Many session tasks belong to one session (`session_id` → `assessment_sessions.id`).
- Many session tasks reference one task (`task_id` → `tasks.id`).
- Many session tasks reference one section (`section_id` → `assessment_sections.id`).
- Optional link to a pool (`pool_rule_id` → `section_pool_rules.id`) or a fixed task (`assessment_task_id` → `assessment_tasks.id`).
- One-to-one with `assessment_session_tasks_answers`.
- One-to-many with `submissions`.

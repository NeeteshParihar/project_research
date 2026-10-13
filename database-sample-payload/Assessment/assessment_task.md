# assessment_tasks

Part of the **Assessment Setup** section. Holds the **fixed** tasks of a section: tasks chosen in advance, so every participant assigned this assessment gets them. Used by `FIXED` and `MIXED` sections only. `POOL` sections have no rows here.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the row |
| `section_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_sections.id`, ON DELETE CASCADE | Section the task belongs to |
| `task_id` | UUID | NOT NULL, FOREIGN KEY → `tasks.id` | The question-bank task |
| `display_order` | INTEGER | NOT NULL, CHECK `>= 1` | Position of the task within the section |
| `marks` | NUMERIC(6,2) | NOT NULL, CHECK `>= 0` | Marks for answering this task correctly |
| `negative_marks` | NUMERIC(6,2) | NOT NULL, default `0`, CHECK `>= 0` | Marks deducted for a wrong answer. Stored as a positive number. |

**Composite constraints (suggested):**

- `UNIQUE (section_id, task_id)`. The same task cannot be added twice to a section.
- `UNIQUE (section_id, display_order)`. Two tasks cannot share a position.

## Rules

- Only add a task whose `status` is `PUBLISHED`. The task's exact version is pinned: because published tasks are immutable (edits create a new row), `task_id` always points to the version the creator chose.
- Marks are set here, per section, not on the task. The same task can be worth different marks in different assessments.
- The ordering rule for a `MIXED` section is fixed tasks first, in `display_order`, then random picks from the pool rules.
- Rows are locked once the assessment is `PUBLISHED`.

## Sample payload

```json
{
  "id": "a0b1c2d3-e4f5-4a6b-8c7d-9e0f1a2b3c4d",
  "section_id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
  "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f",
  "display_order": 1,
  "marks": 2.0,
  "negative_marks": 0.5
}
```

## Sample rows

```json
[
  {
    "id": "a0b1c2d3-e4f5-4a6b-8c7d-9e0f1a2b3c4d",
    "section_id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
    "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f",
    "display_order": 1,
    "marks": 2.0,
    "negative_marks": 0.5
  },
  {
    "id": "b1c2d3e4-f5a6-4b7c-9d8e-0f1a2b3c4d5e",
    "section_id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
    "task_id": "d2e3f4a5-b6c7-4d8e-8f9a-1b2c3d4e5f60",
    "display_order": 2,
    "marks": 2.0,
    "negative_marks": 0.5
  },
  {
    "id": "c2d3e4f5-a6b7-4c8d-8e9f-1a2b3c4d5e6f",
    "section_id": "5e6f7081-92a3-4db4-a5e6-f708192a3b4c",
    "task_id": "e3f4a5b6-c7d8-4e9f-9a0b-2c3d4e5f6071",
    "display_order": 1,
    "marks": 5.0,
    "negative_marks": 1.0
  },
  {
    "id": "d3e4f5a6-b7c8-4d9e-9f0a-2b3c4d5e6f70",
    "section_id": "5e6f7081-92a3-4db4-a5e6-f708192a3b4c",
    "task_id": "f4a5b6c7-d8e9-4f0a-8b1c-3d4e5f607182",
    "display_order": 2,
    "marks": 20.0,
    "negative_marks": 0.0
  }
]
```

## Relationships

- Many assessment tasks belong to one section (`section_id` → `assessment_sections.id`).
- Many assessment tasks reference one question-bank task (`task_id` → `tasks.id`).
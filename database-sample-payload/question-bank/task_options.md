# task_options

Part of the **Question Bank** section. Stores the answer choices of an **MCQ** task. Each option is one row.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the option. Participants' answers refer to this id. |
| `task_id` | UUID | NOT NULL, FOREIGN KEY → `tasks.id`, ON DELETE CASCADE | The MCQ task the option belongs to |
| `option_text` | TEXT | NOT NULL | The text of the choice |
| `is_correct` | BOOLEAN | NOT NULL, default `false` | Whether this option is a correct answer |
| `display_order` | INTEGER | NOT NULL, CHECK `>= 1` | Fixed position of the option |

**Composite constraint (suggested):** `UNIQUE (task_id, display_order)`.

## Rules

- Only for tasks with `task_type = MCQ`.
- A task needs at least 2 options and at least 1 correct option. This is checked when the task is submitted for review.
- If `tasks.content.allow_multiple` is `false`, exactly one option is correct. If `true`, one or more are correct.
- Options of a `PUBLISHED` task cannot change. A fix creates a new task version, and its options are copied.

## Why `display_order` exists

- If `tasks.content.randomize_options` is `true`, options are shuffled for each participant and `display_order` is ignored.
- If it is `false`, options appear in `display_order`. This is needed for choices like "All of the above" (must be last) or numbers that read badly when shuffled (10, 20, 30, 40).

## Grading

`is_correct` is used for auto-grading. The participant's answer stores option ids, and the backend compares the selected set with the set of options where `is_correct = true`. For multi-answer questions, the answer is correct only when the selected set matches the correct set exactly.

## Sample payload

```json
{
  "id": "81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1",
  "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b",
  "option_text": "final",
  "is_correct": true,
  "display_order": 1
}
```

## Sample rows

Options for "Which keyword prevents a class from being subclassed in Java?" and for "Which SQL clause filters groups after aggregation?".

```json
[
  { "id": "81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1", "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b", "option_text": "final",    "is_correct": true,  "display_order": 1 },
  { "id": "81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f2", "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b", "option_text": "static",   "is_correct": false, "display_order": 2 },
  { "id": "81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f3", "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b", "option_text": "abstract", "is_correct": false, "display_order": 3 },
  { "id": "81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f4", "task_id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b", "option_text": "private",  "is_correct": false, "display_order": 4 },

  { "id": "82a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1", "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f", "option_text": "WHERE",    "is_correct": false, "display_order": 1 },
  { "id": "82a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f2", "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f", "option_text": "GROUP BY", "is_correct": false, "display_order": 2 },
  { "id": "82a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f3", "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f", "option_text": "HAVING",   "is_correct": true,  "display_order": 3 },
  { "id": "82a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f4", "task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f", "option_text": "ORDER BY", "is_correct": false, "display_order": 4 }
]
```

## Relationships

- Many options belong to one MCQ task (`task_id` → `tasks.id`).
- Logical link: `assessment_session_tasks_answers.answer.selected_option_ids` holds ids from this table.

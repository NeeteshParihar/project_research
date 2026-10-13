# skill_scores

Part of the **Outcome** section. Marks per skill for one result, so a recruiter sees what the candidate is good at, not just a total. One row per skill per result.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the row |
| `result_id` | UUID | NOT NULL, FOREIGN KEY → `results.id`, ON DELETE CASCADE | The result this score belongs to |
| `skill_id` | UUID | NOT NULL, FOREIGN KEY → `skills.id` | The skill being scored |
| `marks` | NUMERIC(8,2) | NOT NULL | Marks earned in this skill. Can be below 0 if negative marks exceed earned marks. |
| `max_marks` | NUMERIC(8,2) | NOT NULL, CHECK `> 0` | Maximum marks available in this skill |
| `percentage` | NUMERIC(5,2) | NOT NULL, CHECK between `0` and `100` | `marks ÷ max_marks × 100`, not below 0 |

**Composite constraint (suggested):** `UNIQUE (result_id, skill_id)`.

## How the rows are calculated

When a result is calculated, session tasks are grouped by skill:

- **Pooled task:** counts toward the pool's single `skill_id`.
- **Fixed task:** counts toward every skill in the task's `skills` array, with its full marks for each.

Because a fixed task with several skills counts once per skill, the sum of skill marks can exceed `results.total_marks`. When every task has one skill, as in the sample, they match.

## How skill scores are used

1. **Granular hiring:** a 55% candidate may be strong in the one skill the role needs.
2. **Benchmarks:** screening rules such as "overall at least 60% and SQL at least 70%".
3. **Profiles:** combining a person's skill scores across assessments over time.
4. **Learning and development:** low scores point to training for internal staff.

## Sample payload

```json
{
  "id": "a1e0f1a2-b3c4-4d56-8e78-f9a0b1c2d3a1",
  "result_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234",
  "skill_id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b",
  "marks": 6.75,
  "max_marks": 13.0,
  "percentage": 51.92
}
```

## Sample rows

The four skills of the sample result. Marks add up to 33.75, matching `results.total_marks`.

```json
[
  { "id": "a1e0f1a2-b3c4-4d56-8e78-f9a0b1c2d3a1", "result_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234", "skill_id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b", "marks": 6.75,  "max_marks": 13.0, "percentage": 51.92 },
  { "id": "a1e0f1a2-b3c4-4d56-8e78-f9a0b1c2d3a2", "result_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234", "skill_id": "22b2c3d4-e5f6-4071-9b8c-0d1e2f3a4b5c", "marks": 7.5,   "max_marks": 10.0, "percentage": 75.0 },
  { "id": "a1e0f1a2-b3c4-4d56-8e78-f9a0b1c2d3a3", "result_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234", "skill_id": "33c3d4e5-f607-4182-8c9d-1e2f3a4b5c6d", "marks": 12.0,  "max_marks": 20.0, "percentage": 60.0 },
  { "id": "a1e0f1a2-b3c4-4d56-8e78-f9a0b1c2d3a4", "result_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234", "skill_id": "44d4e5f6-0718-4293-9dae-2f3a4b5c6d7e", "marks": 7.5,   "max_marks": 10.0, "percentage": 75.0 }
]
```

## Relationships

- Many skill scores belong to one result (`result_id` → `results.id`).
- Many skill scores reference one skill (`skill_id` → `skills.id`).

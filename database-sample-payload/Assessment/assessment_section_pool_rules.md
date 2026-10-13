# section_pool_rules

Part of the **Assessment Setup** section. A section can have **many pools** (Pool 1, Pool 2, ...). Each row in this table is one pool. A pool tests **one skill**, asks **one task type** at **one difficulty**, and marks **every task in it equally**.

Example: Pool 1 = "2 tasks, Java, MCQ, HARD, 4 marks each, -1 for a wrong answer". Both tasks get identical marking.

When a participant starts a session, the backend goes through each pool of a `POOL` or `MIXED` section, picks `question_count` random tasks that match the pool's skill, type and difficulty, and copies them into that participant's session. This is what makes every participant's questions different.

## What changed from the earlier version

| Attribute | Before | Now |
|-----------|--------|-----|
| `skills_filter` (JSONB with many tags) | present | **Removed**, replaced by `skill_id` |
| `skill_id` | not present | **Added**: exactly one skill per pool |
| `difficulty` | optional (`NULL` = any) | **Required**: exactly one difficulty per pool |
| `task_type` | required | Unchanged, exactly one type per pool |
| Marking | `marks_per_question`, `negative_marks` | Unchanged, applied evenly to every task in the pool |

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the pool |
| `section_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_sections.id`, ON DELETE CASCADE | Section this pool belongs to |
| `skill_id` | UUID | NOT NULL, FOREIGN KEY → `skills.id` | The single skill this pool tests |
| `task_type` | VARCHAR(20) | NOT NULL, CHECK in (`MCQ`, `CODING`, `SUBJECTIVE`) | The single type of task this pool asks |
| `difficulty` | VARCHAR(10) | NOT NULL, CHECK in (`EASY`, `MEDIUM`, `HARD`) | The single difficulty of every task in this pool |
| `question_count` | INTEGER | NOT NULL, CHECK `>= 1` | How many tasks to pick for this pool |
| `marks_per_question` | NUMERIC(6,2) | NOT NULL, CHECK `>= 0` | Marks for each correct task. Same for every task in the pool. |
| `negative_marks` | NUMERIC(6,2) | NOT NULL, default `0`, CHECK `>= 0` | Marks deducted for each wrong answer. Stored as a positive number. Same for every task in the pool. |
| `display_order` | INTEGER | NOT NULL, CHECK `>= 1` | Order of this pool within the section (Pool 1, Pool 2, ...) |

**Composite constraint (suggested):** `UNIQUE (section_id, display_order)`.

**No `assessment_id` column.** The assessment is reached through the section. This keeps the data normalised and avoids a pool pointing at a different assessment than its section.

## Possible values of `task_type`

| Value | Meaning |
|-------|---------|
| `MCQ` | Multiple choice (single or multiple correct options) |
| `CODING` | Coding problem graded by test cases |
| `SUBJECTIVE` | Free-text answer, graded manually or by AI |

## Possible values of `difficulty`

`EASY`, `MEDIUM`, `HARD`

## Marking rules

- Every task picked from a pool gets the same `marks_per_question` and `negative_marks`. Different marks mean a different pool.
- Pool maximum marks = `question_count × marks_per_question`.
- Section maximum marks (pools only) = sum of the pool maximums.
- When a task is picked, the pool's marks and the pool's `id` are copied into the session task. Later edits to the pool cannot change a running session.

## How the backend uses a pool

1. Filter `tasks` where `status = PUBLISHED`, `task_type` matches, `difficulty` matches, and the task is tagged with the pool's skill.
2. Exclude tasks already picked for this session, so two pools cannot give the same task twice.
3. Pick `question_count` tasks at random.
4. Create one session task per pick with the pool's marking.

If fewer matching tasks exist than `question_count`, the assessment must not be publishable. Check this at publish time.

## Sample payload

```json
{
  "id": "6a7b8c9d-0e1f-4a2b-8c3d-4e5f6a7b8c9d",
  "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
  "skill_id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b",
  "task_type": "MCQ",
  "difficulty": "EASY",
  "question_count": 5,
  "marks_per_question": 1.0,
  "negative_marks": 0.25,
  "display_order": 1
}
```

## Sample rows

Section "Core Java MCQ" with three pools, then one pool in each of two other sections.

```json
[
  {
    "id": "6a7b8c9d-0e1f-4a2b-8c3d-4e5f6a7b8c9d",
    "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
    "skill_id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b",
    "task_type": "MCQ",
    "difficulty": "EASY",
    "question_count": 5,
    "marks_per_question": 1.0,
    "negative_marks": 0.25,
    "display_order": 1
  },
  {
    "id": "7b8c9d0e-1f2a-4b3c-9d4e-5f6a7b8c9d0e",
    "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
    "skill_id": "22b2c3d4-e5f6-4071-9b8c-0d1e2f3a4b5c",
    "task_type": "MCQ",
    "difficulty": "MEDIUM",
    "question_count": 5,
    "marks_per_question": 2.0,
    "negative_marks": 0.5,
    "display_order": 2
  },
  {
    "id": "e5f6a7b8-c9d0-4e1f-8a2b-3c4d5e6f7081",
    "section_id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
    "skill_id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b",
    "task_type": "MCQ",
    "difficulty": "HARD",
    "question_count": 2,
    "marks_per_question": 4.0,
    "negative_marks": 1.0,
    "display_order": 3
  },
  {
    "id": "8c9d0e1f-2a3b-4c4d-8e5f-6a7b8c9d0e1f",
    "section_id": "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92",
    "skill_id": "33c3d4e5-f607-4182-8c9d-1e2f3a4b5c6d",
    "task_type": "CODING",
    "difficulty": "MEDIUM",
    "question_count": 1,
    "marks_per_question": 20.0,
    "negative_marks": 0.0,
    "display_order": 1
  },
  {
    "id": "9d0e1f2a-3b4c-4d5e-9f6a-7b8c9d0e1f2a",
    "section_id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
    "skill_id": "44d4e5f6-0718-4293-9dae-2f3a4b5c6d7e",
    "task_type": "MCQ",
    "difficulty": "MEDIUM",
    "question_count": 3,
    "marks_per_question": 2.0,
    "negative_marks": 0.5,
    "display_order": 1
  }
]
```

Worked marks for section "Core Java MCQ":

| Pool | Skill | Type | Difficulty | Count | Marks each | Pool max |
|------|-------|------|------------|-------|-----------|----------|
| 1 | Java | MCQ | EASY | 5 | 1 | 5 |
| 2 | Collections | MCQ | MEDIUM | 5 | 2 | 10 |
| 3 | Java | MCQ | HARD | 2 | 4 | 8 |
| | | | | | **Section max** | **23** |

## Relationships

- Many pools belong to one section (`section_id` → `assessment_sections.id`).
- Many pools reference one skill (`skill_id` → `skills.id`).
- Logical link (not a foreign key): the pool's skill is matched against the skill tags on `tasks`.
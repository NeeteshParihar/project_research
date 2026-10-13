# tasks

Part of the **Question Bank** section. A task is one question (MCQ, coding or subjective) that can be used in many assessments, either picked from a pool or added as a fixed task.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of this task version |
| `task_type` | VARCHAR(20) | NOT NULL, CHECK in (`MCQ`, `CODING`, `SUBJECTIVE`) | Kind of question |
| `title` | VARCHAR(200) | NOT NULL | Short name used in lists and search |
| `description` | TEXT | NULL | Short plain-text summary for authors and reviewers. Not shown to participants. |
| `difficulty` | VARCHAR(10) | NOT NULL, CHECK in (`EASY`, `MEDIUM`, `HARD`) | Difficulty level |
| `default_marks` | NUMERIC(6,2) | NULL, CHECK `>= 0` | Suggested starting marks in the fixed-task picker. **Not used in scoring.** See the note below. |
| `skills` | JSONB | NOT NULL, default `'[]'`, CHECK `jsonb_typeof(skills) = 'array'` | Array of `skills.id` values this task tests |
| `content` | JSONB | NOT NULL | The question itself. Shape depends on `task_type`. |
| `is_predefined` | BOOLEAN | NOT NULL, default `false` | `true` for platform-provided library tasks, `false` for tasks written by the organisation |
| `status` | VARCHAR(20) | NOT NULL, default `DRAFT`, CHECK in (`DRAFT`, `IN_REVIEW`, `PUBLISHED`, `RETIRED`) | Lifecycle state |
| `version` | INTEGER | NOT NULL, default `1`, CHECK `>= 1` | Version number in the chain |
| `previous_task_id` | UUID | NULL, FOREIGN KEY → `tasks.id` (self-reference) | The version this one replaces. `NULL` for version 1. |
| `created_by` | UUID | NOT NULL, FOREIGN KEY → `users.id` | The CREATOR who wrote this version |

Suggested index: a GIN index on `skills`, so pool picks can filter quickly.

## Possible values of `status`

| Value | Meaning | Who moves it |
|-------|---------|--------------|
| `DRAFT` | Being written. Fully editable. | CREATOR |
| `IN_REVIEW` | Submitted for approval. | CREATOR (`task:submit_review`) |
| `PUBLISHED` | Live. Can be picked by pools and added to assessments. **Locked.** | ADMIN (`task:publish`) |
| `RETIRED` | Not used in new assessments. Kept for history. | ADMIN (`task:retire`) |

Transitions: `DRAFT → IN_REVIEW → PUBLISHED → RETIRED`. A reviewer can send `IN_REVIEW → DRAFT` with changes requested.

## Versioning rule

A published task is immutable, because sessions and answers point at it. To fix a typo or add a test case:

1. Copy it into a new row with `status = DRAFT`, `version` + 1, and `previous_task_id` = the old row.
2. Edit and publish the new row.
3. Retire the old row when it is no longer wanted.

Existing assessments still point at the old row until they are re-versioned. Pools only pick `PUBLISHED` tasks, so they get the new version automatically.

## `skills` JSONB

An array of `skills.id` strings: `["11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b"]`. A pool picks a task when `skills` contains the pool's `skill_id`. A task can carry several skills. In results, a pooled task counts only toward its pool's skill, while a fixed task counts toward every skill it carries.

## `content` JSONB: shape by task type

`MCQ`

| Key | Type | Meaning |
|-----|------|---------|
| `question_text` | string | The question |
| `allow_multiple` | boolean | `true` if several options can be correct |
| `randomize_options` | boolean | `true` to shuffle options per participant. When `false`, `task_options.display_order` is used. |

`CODING`

| Key | Type | Meaning |
|-----|------|---------|
| `problem_statement` | string | The problem |
| `input_format` | string | How input is given |
| `output_format` | string | Expected output format |
| `constraints` | string | Limits on the input |
| `time_limit_ms` | integer | Time limit per test case |
| `memory_limit_mb` | integer | Memory limit per test case |
| `allowed_languages` | array of strings | Languages the editor offers |
| `starter_code` | object | Starting code per language |

`SUBJECTIVE`

| Key | Type | Meaning |
|-----|------|---------|
| `question_text` | string | The question |
| `max_words` | integer | Answer length limit |
| `rubric` | array of strings | Points a good answer covers, for evaluators and AI |

## Note on `default_marks`

Marks are set per pool (`section_pool_rules.marks_per_question`) and per fixed task (`assessment_tasks.marks`). Nothing in the scoring reads `default_marks`, so it is only a convenience. You said earlier it might not be needed. I recommend dropping it. It is documented here because it is still in the ERD.

## Note on `is_predefined`

I read this as library tasks supplied by the platform versus tasks the organisation wrote. Predefined tasks should be copy-only: creators clone them into their own editable task instead of editing them. Tell me if you meant something else.

## Sample payload

```json
{
  "id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b",
  "task_type": "MCQ",
  "title": "Java: preventing subclassing",
  "description": "Tests knowledge of the final keyword on classes.",
  "difficulty": "EASY",
  "default_marks": 1.0,
  "skills": ["11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b"],
  "content": {
    "question_text": "Which keyword prevents a class from being subclassed in Java?",
    "allow_multiple": false,
    "randomize_options": true
  },
  "is_predefined": false,
  "status": "PUBLISHED",
  "version": 1,
  "previous_task_id": null,
  "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
}
```

## Sample rows

```json
[
  {
    "id": "5b6c7d8e-9f01-4a23-8b45-6c7d8e9f0a1b",
    "task_type": "MCQ",
    "title": "Java: preventing subclassing",
    "description": "Tests knowledge of the final keyword on classes.",
    "difficulty": "EASY",
    "default_marks": 1.0,
    "skills": ["11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b"],
    "content": {
      "question_text": "Which keyword prevents a class from being subclassed in Java?",
      "allow_multiple": false,
      "randomize_options": true
    },
    "is_predefined": false,
    "status": "PUBLISHED",
    "version": 1,
    "previous_task_id": null,
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  },
  {
    "id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c",
    "task_type": "CODING",
    "title": "Two Sum: indices",
    "description": "Find two numbers that add up to a target. A brute force passes the small cases but fails the large ones.",
    "difficulty": "MEDIUM",
    "default_marks": 20.0,
    "skills": ["33c3d4e5-f607-4182-8c9d-1e2f3a4b5c6d"],
    "content": {
      "problem_statement": "Given an array of integers and a target, print the indices of the two numbers that add up to the target.",
      "input_format": "First line: N. Second line: N integers. Third line: target.",
      "output_format": "Two space-separated indices (0-based), smaller first.",
      "constraints": "2 <= N <= 1000000; exactly one valid pair exists.",
      "time_limit_ms": 2000,
      "memory_limit_mb": 256,
      "allowed_languages": ["java", "python", "cpp"],
      "starter_code": { "java": "import java.util.*;\npublic class Main {\n  public static void main(String[] args) {\n  }\n}" }
    },
    "is_predefined": true,
    "status": "PUBLISHED",
    "version": 1,
    "previous_task_id": null,
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  },
  {
    "id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f",
    "task_type": "MCQ",
    "title": "SQL: filtering groups",
    "description": "Tests the difference between WHERE and HAVING.",
    "difficulty": "MEDIUM",
    "default_marks": 2.0,
    "skills": ["44d4e5f6-0718-4293-9dae-2f3a4b5c6d7e"],
    "content": {
      "question_text": "Which SQL clause filters groups after aggregation?",
      "allow_multiple": false,
      "randomize_options": true
    },
    "is_predefined": false,
    "status": "PUBLISHED",
    "version": 1,
    "previous_task_id": null,
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  },
  {
    "id": "7d8e9f0a-1b23-4c45-8d67-8e9f0a1b2c3d",
    "task_type": "MCQ",
    "title": "SQL: filtering groups",
    "description": "Tests the difference between WHERE and HAVING. Wording clarified.",
    "difficulty": "MEDIUM",
    "default_marks": 2.0,
    "skills": ["44d4e5f6-0718-4293-9dae-2f3a4b5c6d7e"],
    "content": {
      "question_text": "Which SQL clause is used to filter rows AFTER GROUP BY has aggregated them?",
      "allow_multiple": false,
      "randomize_options": true
    },
    "is_predefined": false,
    "status": "DRAFT",
    "version": 2,
    "previous_task_id": "c1d2e3f4-a5b6-4c7d-9e8f-0a1b2c3d4e5f",
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  }
]
```

## Relationships

- One task has many `task_options` (MCQ only).
- One task has many `test_cases` (CODING only).
- Many tasks are tagged with many `skills` (logical link through the `skills` JSONB).
- A task points to its previous version (`previous_task_id` → `tasks.id`).
- Many tasks are created by one user (`created_by` → `users.id`).
- One task is used by many `assessment_tasks` and `assessment_session_tasks` rows.

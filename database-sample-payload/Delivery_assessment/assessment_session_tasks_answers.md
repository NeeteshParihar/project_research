# assessment_session_tasks_answers

Part of the **Delivery** section. Holds the **answer values** for a session task: what the participant answered, how it was graded, and the score. There is exactly one row per session task (1:1) and it stores only the **latest** answer. Each save overwrites the previous one. For coding tasks, every run is kept in `submissions`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the answer row |
| `session_task_id` | UUID | NOT NULL, UNIQUE, FOREIGN KEY → `assessment_session_tasks.id`, ON DELETE CASCADE | The session task this answers. UNIQUE makes the relationship 1:1. |
| `answer` | JSONB | NULL | The participant's latest answer. Shape depends on the task type. `NULL` until first saved. |
| `answer_status` | VARCHAR(20) | NOT NULL, default `UNATTEMPTED`, CHECK in (`UNATTEMPTED`, `SAVED`, `MARKED_FOR_REVIEW`) | State of the question from the participant's side |
| `eval_status` | VARCHAR(25) | NOT NULL, default `PENDING_EVALUATION`, CHECK in (`PENDING_EVALUATION`, `NEEDS_MANUAL_REVIEW`, `COMPLETED`, `ERROR`) | State of grading for this answer |
| `marks_awarded` | NUMERIC(6,2) | NULL | Marks earned. Can be negative because of negative marking. `NULL` until graded. |
| `max_marks` | NUMERIC(6,2) | NOT NULL, CHECK `>= 0` | Maximum marks possible. Equal to `assessment_session_tasks.marks`. |
| `llm_evaluation` | JSONB | NULL | AI analysis of the answer. Advisory only. |
| `evaluated_by` | UUID | NULL, FOREIGN KEY → `users.id` | The EVALUATOR who reviewed it manually. `NULL` when only graded automatically. |
| `feedback` | TEXT | NULL | Note from a human reviewer |

## Possible values of `answer_status`

| Value | Meaning |
|-------|---------|
| `UNATTEMPTED` | No answer saved yet. |
| `SAVED` | An answer is saved (auto-save or explicit). |
| `MARKED_FOR_REVIEW` | The participant flagged it to revisit. An answer may or may not be present. |

## Possible values of `eval_status`

| Value | Meaning |
|-------|---------|
| `PENDING_EVALUATION` | Not graded yet. This is the default, including during the session. |
| `NEEDS_MANUAL_REVIEW` | Auto-grading is done or not possible, and a human must grade it (for example subjective answers). |
| `COMPLETED` | Final marks are set. |
| `ERROR` | Grading failed (for example the code runner crashed). It can be retried. |

## `answer` JSONB: shape by task type

| Task type | Shape |
|-----------|-------|
| `MCQ` | `{ "selected_option_ids": ["<task_options.id>", ...] }` |
| `CODING` | `{ "language": "java", "code": "..." }` |
| `SUBJECTIVE` | `{ "text": "..." }` |

Options are stored by `task_options.id`, not by letter, so shuffled options can never be mis-graded.

## `llm_evaluation` JSONB

| Key | Type | Meaning |
|-----|------|---------|
| `ai_score` | number | AI's score for the answer |
| `max_ai_score` | number | Scale of `ai_score` |
| `reasoning` | string | AI's explanation |
| `time_complexity` | string | (coding only) estimated complexity |

The AI result never sets `marks_awarded` by itself. Marks come from auto-grading or from the human evaluator.

## Grading rules

- **MCQ:** correct → `marks`. Wrong → `-negative_marks`. Unattempted → 0.
- **CODING:** `marks × score_percent / 100`, using the score of the latest submission.
- **SUBJECTIVE:** set by the evaluator, or by a reviewed AI score.

## Sample payload

```json
{
  "id": "70b7c8d9-e0f1-4a23-9b45-c6d7e8f9a012",
  "session_task_id": "40d4e5f6-0718-4294-9db5-c6e7f8091a2b",
  "answer": { "selected_option_ids": ["81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1"] },
  "answer_status": "SAVED",
  "eval_status": "COMPLETED",
  "marks_awarded": 1.0,
  "max_marks": 1.0,
  "llm_evaluation": null,
  "evaluated_by": null,
  "feedback": null
}
```

## Sample rows

```json
[
  {
    "id": "70b7c8d9-e0f1-4a23-9b45-c6d7e8f9a012",
    "session_task_id": "40d4e5f6-0718-4294-9db5-c6e7f8091a2b",
    "answer": { "selected_option_ids": ["81a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1"] },
    "answer_status": "SAVED",
    "eval_status": "COMPLETED",
    "marks_awarded": 1.0,
    "max_marks": 1.0,
    "llm_evaluation": null,
    "evaluated_by": null,
    "feedback": null
  },
  {
    "id": "71b7c8d9-e0f1-4a23-9b45-c6d7e8f9a123",
    "session_task_id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
    "answer": {
      "language": "java",
      "code": "import java.util.*;\npublic class Main { public static void main(String[] a) { /* nested loops */ } }"
    },
    "answer_status": "SAVED",
    "eval_status": "COMPLETED",
    "marks_awarded": 12.0,
    "max_marks": 20.0,
    "llm_evaluation": {
      "ai_score": 6,
      "max_ai_score": 10,
      "reasoning": "Correct logic but nested loops fail the large-input cases.",
      "time_complexity": "O(N^2)"
    },
    "evaluated_by": "e0f1a2b3-c4d5-4e6f-8a7b-9c0d1e2f3a4b",
    "feedback": "Logic is right. Ask how this could be done in one pass with a hash map."
  },
  {
    "id": "72b7c8d9-e0f1-4a23-9b45-c6d7e8f9a234",
    "session_task_id": "42f60718-293a-4b16-8ce8-f9091a2b3c4d",
    "answer": { "selected_option_ids": ["82a1b2c3-d4e5-4f60-8a71-92b3c4d5e6f1"] },
    "answer_status": "MARKED_FOR_REVIEW",
    "eval_status": "COMPLETED",
    "marks_awarded": -0.5,
    "max_marks": 2.0,
    "llm_evaluation": null,
    "evaluated_by": null,
    "feedback": null
  }
]
```

## Relationships

- One-to-one with `assessment_session_tasks` (`session_task_id` → `assessment_session_tasks.id`).
- Many answers are reviewed by one user (`evaluated_by` → `users.id`).

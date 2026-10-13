# submissions

Part of the **Delivery** section. For **coding tasks**, every time the participant runs or submits code, one row is added here. A task can have many submissions (1:many), and none are overwritten. The latest one drives the grade stored in `assessment_session_tasks_answers`.

MCQ and subjective tasks do not use this table.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the submission |
| `session_task_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_session_tasks.id`, ON DELETE CASCADE | The session task this code was submitted for. Not unique, so many per task. |
| `language` | VARCHAR(30) | NOT NULL | Programming language used (for example `java`, `python`) |
| `source_code` | TEXT | NOT NULL | The code exactly as submitted |
| `status` | VARCHAR(25) | NOT NULL, default `QUEUED`, CHECK in (`QUEUED`, `PROCESSING`, `ACCEPTED`, `WRONG_ANSWER`, `TIME_LIMIT_EXCEEDED`, `COMPILATION_ERROR`, `RUNTIME_ERROR`, `INTERNAL_ERROR`) | Execution status of this run |
| `verdict` | VARCHAR(10) | NULL, CHECK in (`PASS`, `FAIL`, `PARTIAL`) | Simple outcome for UI colouring. `NULL` until finished. |
| `visible_passed` | INTEGER | NULL, CHECK `>= 0` | Visible test cases passed |
| `visible_total` | INTEGER | NULL, CHECK `>= 0` | Visible test cases run |
| `hidden_passed` | INTEGER | NULL, CHECK `>= 0` | Hidden test cases passed. `NULL` if only visible cases were run. |
| `hidden_total` | INTEGER | NULL, CHECK `>= 0` | Hidden test cases run. `NULL` if only visible cases were run. |
| `score_percent` | NUMERIC(5,2) | NULL, CHECK between `0` and `100` | Weighted percentage from test case weights. `NULL` until finished. |
| `test_results` | JSONB | NULL | Per-test-case details of this run |
| `submitted_at` | TIMESTAMPTZ | NOT NULL, default `now()` | When the code was sent |

Suggested index: `(session_task_id, submitted_at DESC)`, so the latest submission is found quickly.

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `QUEUED` | Waiting for the code runner. |
| `PROCESSING` | Running. |
| `ACCEPTED` | All test cases passed. |
| `WRONG_ANSWER` | Ran, but at least one test case gave a wrong output. |
| `TIME_LIMIT_EXCEEDED` | Too slow on at least one test case. |
| `COMPILATION_ERROR` | The code did not compile. |
| `RUNTIME_ERROR` | The code crashed (for example a null pointer). |
| `INTERNAL_ERROR` | The platform's runner failed. Not the participant's fault. Can be retried. |

## Possible values of `verdict`

| Value | Meaning |
|-------|---------|
| `PASS` | All test cases passed. |
| `FAIL` | No test case passed, or the code did not run. |
| `PARTIAL` | Some, but not all, test cases passed. |

## `test_results` JSONB

| Key | Type | Meaning |
|-----|------|---------|
| `execution_time_ms` | integer | Longest run time |
| `memory_used_mb` | number | Peak memory |
| `cases` | array of objects | One per test case: `test_case_id`, `status` (`PASS`, `FAIL`, `TLE`, `ERROR`), and, for **visible** cases only, `actual_output` |
| `error` | string or null | Compiler or runtime message |

Hidden test case inputs and expected outputs are never put in `test_results`, so the participant cannot read them.

## How this relates to the answer row

- `submissions` = history of every run, for audit and replay.
- `assessment_session_tasks_answers` = one row with the final answer and marks.
- Marks use the **latest** submission's `score_percent`.

## Sample payload

```json
{
  "id": "82c8d9e0-f1a2-4b34-8c56-d7e8f9a0b123",
  "session_task_id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
  "language": "java",
  "source_code": "import java.util.*;\npublic class Main { public static void main(String[] a) { /* nested loops */ } }",
  "status": "TIME_LIMIT_EXCEEDED",
  "verdict": "PARTIAL",
  "visible_passed": 3,
  "visible_total": 3,
  "hidden_passed": 3,
  "hidden_total": 5,
  "score_percent": 60.0,
  "test_results": {
    "execution_time_ms": 2000,
    "memory_used_mb": 48.2,
    "cases": [
      { "test_case_id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b1", "status": "PASS", "actual_output": "0 1" },
      { "test_case_id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b4", "status": "PASS" },
      { "test_case_id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b7", "status": "TLE" }
    ],
    "error": null
  },
  "submitted_at": "2026-10-21T05:20:40Z"
}
```

## Sample rows (three attempts at the same task)

```json
[
  {
    "id": "80c8d9e0-f1a2-4b34-8c56-d7e8f9a0b123",
    "session_task_id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
    "language": "java",
    "source_code": "public class Main { void solve( { } }",
    "status": "COMPILATION_ERROR",
    "verdict": "FAIL",
    "visible_passed": 0,
    "visible_total": 3,
    "hidden_passed": null,
    "hidden_total": null,
    "score_percent": 0.0,
    "test_results": { "execution_time_ms": 0, "memory_used_mb": 0, "cases": [], "error": "Main.java:1: error: ')' expected" },
    "submitted_at": "2026-10-21T04:58:10Z"
  },
  {
    "id": "81c8d9e0-f1a2-4b34-8c56-d7e8f9a0b123",
    "session_task_id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
    "language": "java",
    "source_code": "/* first working version */",
    "status": "WRONG_ANSWER",
    "verdict": "PARTIAL",
    "visible_passed": 2,
    "visible_total": 3,
    "hidden_passed": null,
    "hidden_total": null,
    "score_percent": null,
    "test_results": { "execution_time_ms": 120, "memory_used_mb": 41.0, "cases": [], "error": null },
    "submitted_at": "2026-10-21T05:10:02Z"
  },
  {
    "id": "82c8d9e0-f1a2-4b34-8c56-d7e8f9a0b123",
    "session_task_id": "41e5f607-1829-4a05-9bd7-e8091a2b3c4d",
    "language": "java",
    "source_code": "/* final version, nested loops */",
    "status": "TIME_LIMIT_EXCEEDED",
    "verdict": "PARTIAL",
    "visible_passed": 3,
    "visible_total": 3,
    "hidden_passed": 3,
    "hidden_total": 5,
    "score_percent": 60.0,
    "test_results": { "execution_time_ms": 2000, "memory_used_mb": 48.2, "cases": [], "error": null },
    "submitted_at": "2026-10-21T05:20:40Z"
  }
]
```

## Relationships

- Many submissions belong to one session task (`session_task_id` → `assessment_session_tasks.id`).

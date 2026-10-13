# test_cases

Part of the **Question Bank** section. Stores the inputs and expected outputs used to grade a **CODING** task. Each row is one test case.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the test case |
| `task_id` | UUID | NOT NULL, FOREIGN KEY → `tasks.id`, ON DELETE CASCADE | The coding task the test case belongs to |
| `input` | TEXT | NOT NULL | Text fed to the program on standard input |
| `expected_output` | TEXT | NOT NULL | Output the program must print |
| `is_hidden` | BOOLEAN | NOT NULL, default `false` | `false` = shown to the participant as an example. `true` = never shown. |
| `weight` | INTEGER | NOT NULL, default `1`, CHECK `>= 1` | Importance of this case in the score |
| `is_performance` | BOOLEAN | NOT NULL, default `false` | `true` if the case exists to test speed or memory, not just correctness |

## Rules

- Only for tasks with `task_type = CODING`. A coding task needs at least one visible and one hidden case before it can be submitted for review.
- Test cases of a `PUBLISHED` task cannot change. A fix creates a new task version, and its test cases are copied.
- Hidden cases are never sent to the browser, and their inputs and outputs never appear in `submissions.test_results`.

## `weight`: partial scoring

Cases are not equally important, so each has a weight. The score of a submission is:

`score_percent = (sum of weights of passed cases ÷ sum of weights of all cases) × 100`

## `is_performance`: what and why

A brute-force solution can pass small inputs but time out on huge ones. Performance cases use very large inputs, so they separate a correct but slow solution from an efficient one. When one fails with a time limit, the participant is told "your logic is correct but too slow" instead of "wrong answer". They are usually hidden and carry a higher weight.

## Sample payload

```json
{
  "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b1",
  "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c",
  "input": "4\n2 7 11 15\n9",
  "expected_output": "0 1",
  "is_hidden": false,
  "weight": 2,
  "is_performance": false
}
```

## Sample rows

The task "Two Sum: indices". Total weight = 3×2 + 3×2 + 2×4 = 20. The large inputs of the performance cases are shortened here with `...`; the real rows hold the full text.

```json
[
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b1", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "4\n2 7 11 15\n9",   "expected_output": "0 1", "is_hidden": false, "weight": 2, "is_performance": false },
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b2", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "3\n3 2 4\n6",        "expected_output": "1 2", "is_hidden": false, "weight": 2, "is_performance": false },
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b3", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "2\n3 3\n6",          "expected_output": "0 1", "is_hidden": false, "weight": 2, "is_performance": false },

  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b4", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "5\n-1 -2 -3 -4 -5\n-8", "expected_output": "2 4", "is_hidden": true, "weight": 2, "is_performance": false },
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b5", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "4\n0 4 3 0\n0",          "expected_output": "0 3", "is_hidden": true, "weight": 2, "is_performance": false },
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b6", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "6\n1 5 9 13 17 21\n30",  "expected_output": "2 5", "is_hidden": true, "weight": 2, "is_performance": false },

  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b7", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "100000\n1 2 3 ...\n199999",  "expected_output": "99998 99999", "is_hidden": true, "weight": 4, "is_performance": true },
  { "id": "e1b1c2d3-e4f5-4a67-8b89-c0d1e2f3a4b8", "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c", "input": "1000000\n1 2 3 ...\n1999999", "expected_output": "999998 999999", "is_hidden": true, "weight": 4, "is_performance": true }
]
```

## Relationships

- Many test cases belong to one coding task (`task_id` → `tasks.id`).
- Logical link: `submissions.test_results.cases[].test_case_id` refers to ids in this table.

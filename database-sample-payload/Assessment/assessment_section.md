# assessment_sections

Part of the **Assessment Setup** section. A section is a group of tasks inside an assessment (for example "Core Java MCQ" or "Coding Challenge"). `selection_mode` decides where the section's tasks come from.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the section |
| `assessment_id` | UUID | NOT NULL, FOREIGN KEY → `assessments.id`, ON DELETE CASCADE | Assessment this section belongs to |
| `title` | VARCHAR(150) | NOT NULL | Section name shown to the candidate |
| `instructions` | TEXT | NULL | Instructions shown at the start of the section |
| `selection_mode` | VARCHAR(10) | NOT NULL, CHECK in (`FIXED`, `POOL`, `MIXED`) | Where the section's tasks come from |
| `generated_by` | VARCHAR(10) | NOT NULL, default `MANUAL`, CHECK in (`MANUAL`, `AI`) | Who built this section |
| `generation_rationale` | TEXT | NULL | Why the AI built the section this way. Only filled when `generated_by = AI`. |
| `display_order` | INTEGER | NOT NULL, CHECK `>= 1` | Position of the section in the assessment |

**Composite constraint (suggested):** `UNIQUE (assessment_id, display_order)`. Two sections cannot take the same position in one assessment.

## Possible values of `selection_mode`

| Value | Tasks come from | Rows used |
|-------|-----------------|-----------|
| `FIXED` | A hand-picked list. Every participant gets the same tasks. | `assessment_tasks` |
| `POOL` | Random picks from the question bank at session start, based on rules. | `section_pool_rules` |
| `MIXED` | Both: the fixed tasks first, then random picks to fill the rest. | `assessment_tasks` and `section_pool_rules` |

A `FIXED` section must have at least one `assessment_tasks` row and no pool rules. A `POOL` section must have at least one pool rule and no fixed tasks. A `MIXED` section must have both. The application enforces this when publishing.

## Possible values of `generated_by`

| Value | Meaning |
|-------|---------|
| `MANUAL` | Built by a person (CREATOR or ADMIN). |
| `AI` | Built by the AI, for example from a participant's resume in a `PERSONALISED` assessment. |

## Sample payload

```json
{
  "id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
  "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "title": "Core Java MCQ",
  "instructions": "Answer all questions. Each wrong answer carries negative marking.",
  "selection_mode": "POOL",
  "generated_by": "MANUAL",
  "generation_rationale": null,
  "display_order": 1
}
```

## Sample rows

```json
[
  {
    "id": "2b3c4d5e-6f70-4a81-92b3-c4d5e6f70a81",
    "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "title": "Core Java MCQ",
    "instructions": "Answer all questions. Each wrong answer carries negative marking.",
    "selection_mode": "POOL",
    "generated_by": "MANUAL",
    "generation_rationale": null,
    "display_order": 1
  },
  {
    "id": "3c4d5e6f-7081-4b92-a3c4-d5e6f7081b92",
    "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "title": "Coding Challenge",
    "instructions": "Solve the problem in your preferred language. Hidden test cases are used for scoring.",
    "selection_mode": "POOL",
    "generated_by": "MANUAL",
    "generation_rationale": null,
    "display_order": 2
  },
  {
    "id": "4d5e6f70-8192-4ca3-b4d5-e6f708192ca3",
    "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "title": "Database Fundamentals",
    "instructions": "Two compulsory questions followed by random ones.",
    "selection_mode": "MIXED",
    "generated_by": "MANUAL",
    "generation_rationale": null,
    "display_order": 3
  },
  {
    "id": "5e6f7081-92a3-4db4-a5e6-f708192a3b4c",
    "assessment_id": "6f5e4d3c-2b1a-4f0e-9d8c-7b6a5f4e3d2c",
    "title": "Resume-Based Questions",
    "instructions": "These questions are based on the projects and skills in your resume.",
    "selection_mode": "FIXED",
    "generated_by": "AI",
    "generation_rationale": "Resume lists 3 years of Java and Spring Boot, plus PostgreSQL work. Selected medium-difficulty Java, Spring and SQL tasks matching those skills.",
    "display_order": 1
  }
]
```

## Relationships

- Many sections belong to one assessment (`assessment_id` → `assessments.id`).
- One section has many `section_pool_rules` (for `POOL` and `MIXED`).
- One section has many `assessment_tasks` (for `FIXED` and `MIXED`).
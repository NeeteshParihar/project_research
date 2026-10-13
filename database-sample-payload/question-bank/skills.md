# skills

Part of the **Question Bank** section. The master list of skills the platform tests (Java, SQL, Communication and so on). It connects question authoring, random pool picking and result reporting.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the skill |
| `name` | VARCHAR(100) | NOT NULL, UNIQUE (case-insensitive) | Skill name. Implement with a unique index on `lower(name)`. |
| `category` | VARCHAR(50) | NOT NULL | Group the skill belongs to, for filtering in the UI |

## Suggested values of `category`

`category` is free text so new groups need no schema change. Suggested starting values:

| Value | Examples |
|-------|----------|
| `LANGUAGE` | Java, Python |
| `FRAMEWORK` | Spring Boot, React |
| `DATABASE` | SQL, PostgreSQL |
| `CONCEPT` | Algorithms, Collections, System Design |
| `SOFT_SKILL` | Communication, Problem Solving |

## How skills are used

1. **Tagging:** a CREATOR adds skill ids to a task's `skills` array.
2. **Generation:** each pool in `section_pool_rules` has one `skill_id`. The backend picks tasks whose `skills` contain it.
3. **Reporting:** after grading, marks are summed per skill into `skill_scores` (per skill, a candidate scored 75%, 51%, and so on).

## Rules

- Skills are referenced by id, so renaming a skill is safe and changes nothing in tasks, pools or results.
- A skill used by any task, pool or score cannot be deleted. The ERD has no `status` column, so skills cannot be retired. Say so if you want one.

## Sample payload

```json
{
  "id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b",
  "name": "Java",
  "category": "LANGUAGE"
}
```

## Sample rows

```json
[
  { "id": "11a1b2c3-d4e5-4f60-8a7b-9c0d1e2f3a4b", "name": "Java",        "category": "LANGUAGE" },
  { "id": "22b2c3d4-e5f6-4071-9b8c-0d1e2f3a4b5c", "name": "Collections", "category": "CONCEPT" },
  { "id": "33c3d4e5-f607-4182-8c9d-1e2f3a4b5c6d", "name": "Algorithms",  "category": "CONCEPT" },
  { "id": "44d4e5f6-0718-4293-9dae-2f3a4b5c6d7e", "name": "SQL",         "category": "DATABASE" }
]
```

## Relationships

- Many pools reference one skill (`section_pool_rules.skill_id` → `skills.id`).
- Many skill scores reference one skill (`skill_scores.skill_id` → `skills.id`).
- Logical link: `tasks.skills` holds skill ids.

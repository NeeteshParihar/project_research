# departments

Part of the **Organisation** section. A department groups job roles (e.g. Engineering, Sales).

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the department |
| `name` | VARCHAR(100) | NOT NULL, UNIQUE | Department name; no two departments can share a name |
| `status` | VARCHAR(20) | NOT NULL, default `ACTIVE`, CHECK in (`ACTIVE`, `INACTIVE`) | Current state of the department |

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `ACTIVE` | Department is in use. It can be selected for new job roles. |
| `INACTIVE` | Department is disabled. It is kept for history but cannot be selected for new job roles. |

## Sample payload

```json
{
  "id": "3f8a2b1c-6d4e-4a7b-9c10-1e2f3a4b5c6d",
  "name": "Engineering",
  "status": "ACTIVE"
}
```

## Sample rows

```json
[
  {
    "id": "3f8a2b1c-6d4e-4a7b-9c10-1e2f3a4b5c6d",
    "name": "Engineering",
    "status": "ACTIVE"
  },
  {
    "id": "7b9c4d2e-1a3f-4e5b-8d6c-2f1e0a9b8c7d",
    "name": "Sales",
    "status": "ACTIVE"
  },
  {
    "id": "c2d1e0f9-8a7b-4c6d-9e5f-4a3b2c1d0e9f",
    "name": "Legacy Support",
    "status": "INACTIVE"
  }
]
```

## Relationships

- One department has many `job_roles` (`job_roles.department_id` → `departments.id`).
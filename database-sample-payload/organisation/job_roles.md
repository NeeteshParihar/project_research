# job_roles

Part of the **Organisation** section. A job role belongs to one department (e.g. "Backend Engineer" in Engineering).

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the job role |
| `department_id` | UUID | NOT NULL, FOREIGN KEY → `departments.id` | Department this role belongs to |
| `name` | VARCHAR(150) | NOT NULL | Job role name |
| `status` | VARCHAR(20) | NOT NULL, default `ACTIVE`, CHECK in (`ACTIVE`, `INACTIVE`, `ARCHIVED`) | Current state of the job role |
| `is_open_for_hiring` | BOOLEAN | NOT NULL, default `false` | Whether the role is currently open for hiring (API field: `isOpenForHiring`) |
| `number_of_positions` | INTEGER | NOT NULL, default `0`, CHECK `>= 0` | Number of open positions for the role (API field: `numberOfPositions`) |

**Composite constraint:** `UNIQUE (department_id, name)`. The same role name cannot repeat within one department, but different departments can use the same name.

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `ACTIVE` | Role is in use. It can be linked to assessments and participants. |
| `INACTIVE` | Role is temporarily disabled. Existing links stay, but it cannot be selected for new assessments or participants. |
| `ARCHIVED` | Role is permanently retired. Kept only for history and reporting. |

`status` describes the lifecycle of the role record. `is_open_for_hiring` is separate: it says whether hiring is happening right now. An `ACTIVE` role can have `is_open_for_hiring = false`.

## Sample payload

```json
{
  "id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
  "department_id": "3f8a2b1c-6d4e-4a7b-9c10-1e2f3a4b5c6d",
  "name": "Backend Engineer",
  "status": "ACTIVE",
  "is_open_for_hiring": true,
  "number_of_positions": 5
}
```

## Sample rows

```json
[
  {
    "id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "department_id": "3f8a2b1c-6d4e-4a7b-9c10-1e2f3a4b5c6d",
    "name": "Backend Engineer",
    "status": "ACTIVE",
    "is_open_for_hiring": true,
    "number_of_positions": 5
  },
  {
    "id": "1a2b3c4d-5e6f-4a7b-8c9d-0e1f2a3b4c5d",
    "department_id": "3f8a2b1c-6d4e-4a7b-9c10-1e2f3a4b5c6d",
    "name": "QA Engineer",
    "status": "ACTIVE",
    "is_open_for_hiring": false,
    "number_of_positions": 0
  },
  {
    "id": "5e6f7a8b-9c0d-4e1f-a2b3-c4d5e6f7a8b9",
    "department_id": "7b9c4d2e-1a3f-4e5b-8d6c-2f1e0a9b8c7d",
    "name": "Sales Executive",
    "status": "INACTIVE",
    "is_open_for_hiring": false,
    "number_of_positions": 0
  },
  {
    "id": "b0c1d2e3-f4a5-4b6c-8d7e-9f0a1b2c3d4e",
    "department_id": "7b9c4d2e-1a3f-4e5b-8d6c-2f1e0a9b8c7d",
    "name": "Telecaller",
    "status": "ARCHIVED",
    "is_open_for_hiring": false,
    "number_of_positions": 0
  }
]
```

## Relationships

- Many job roles belong to one department (`job_roles.department_id` → `departments.id`).
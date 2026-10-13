# audit_logs

Part of the **Monitoring** section. An append-only record of what **staff** did in the system: who changed what, when, and how it looked before and after. It does not record participant activity (that is `proctoring_events`).

## Why this table exists

- **Accountability:** if a score changes from 40 to 95 or an assessment is deleted before a hiring drive, you can see which user did it.
- **Compliance:** enterprise clients (banks, large companies) expect a tamper-resistant history of system changes.
- **Debugging:** developers can see which configuration changed recently.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the log entry |
| `user_id` | UUID | NULL, FOREIGN KEY → `users.id` | Who did it. `NULL` when the system did it (for example an automatic expiry). |
| `action` | VARCHAR(60) | NOT NULL | What happened, as an uppercase code (see below) |
| `entity_type` | VARCHAR(40) | NOT NULL | Kind of record that changed |
| `entity_id` | UUID | NOT NULL | The `id` of the record that changed |
| `old_value` | JSONB | NULL | The changed fields before the action. `NULL` for creations. |
| `new_value` | JSONB | NULL | The changed fields after the action. `NULL` for deletions. |
| `created_at` | TIMESTAMPTZ | NOT NULL, default `now()` | When the action happened |

## Rules

- **Append-only:** the application never updates or deletes rows. Revoke `UPDATE` and `DELETE` on this table for the application's database user.
- `entity_id` is not a foreign key, because it points to many different tables and the record may later be deleted.
- Store only the fields that changed, not whole rows. Never store secrets such as `password_hash`.
- Suggested indexes: `(entity_type, entity_id, created_at)` and `(user_id, created_at)`.
- The ERD has no `ip_address` or `user_agent` columns, so they are not included. They are common in audit logs and can be added later if wanted.

## Suggested values of `entity_type`

`ASSESSMENT`, `TASK`, `SKILL`, `SECTION_POOL_RULE`, `ASSIGNMENT`, `PARTICIPANT`, `RESULT`, `USER`, `ROLE`, `JOB_ROLE`, `DEPARTMENT`

## Suggested values of `action`

| Group | Actions |
|-------|---------|
| Assessments | `ASSESSMENT_CREATED`, `ASSESSMENT_UPDATED`, `ASSESSMENT_PUBLISHED`, `ASSESSMENT_ARCHIVED` |
| Pools and tasks | `SECTION_POOL_RULE_CREATED`, `SECTION_POOL_RULE_UPDATED`, `SECTION_POOL_RULE_DELETED`, `TASK_PUBLISHED`, `TASK_RETIRED` |
| Assignments | `ASSIGNMENT_CREATED`, `ASSIGNMENT_CANCELLED` |
| Results | `RESULT_PUBLISHED`, `RESULT_INVALIDATED`, `ANSWER_MANUALLY_GRADED` |
| Access | `USER_CREATED`, `USER_STATUS_CHANGED`, `USER_ROLE_ASSIGNED`, `USER_ROLE_REMOVED`, `ROLE_PERMISSIONS_CHANGED` |
| Organisation | `JOB_ROLE_UPDATED`, `DEPARTMENT_UPDATED` |

## Sample payload

```json
{
  "id": "b1f1a2b3-c4d5-4e67-8f89-a0b1c2d3e4f1",
  "user_id": "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
  "action": "ASSESSMENT_PUBLISHED",
  "entity_type": "ASSESSMENT",
  "entity_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "old_value": { "status": "DRAFT" },
  "new_value": { "status": "PUBLISHED" },
  "created_at": "2026-10-19T09:30:00Z"
}
```

## Sample rows

```json
[
  {
    "id": "b1f1a2b3-c4d5-4e67-8f89-a0b1c2d3e4f1",
    "user_id": "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
    "action": "ASSESSMENT_PUBLISHED",
    "entity_type": "ASSESSMENT",
    "entity_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "old_value": { "status": "DRAFT" },
    "new_value": { "status": "PUBLISHED" },
    "created_at": "2026-10-19T09:30:00Z"
  },
  {
    "id": "b1f1a2b3-c4d5-4e67-8f89-a0b1c2d3e4f2",
    "user_id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "action": "JOB_ROLE_UPDATED",
    "entity_type": "JOB_ROLE",
    "entity_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "old_value": { "is_open_for_hiring": false, "number_of_positions": 0 },
    "new_value": { "is_open_for_hiring": true, "number_of_positions": 5 },
    "created_at": "2026-10-19T10:05:00Z"
  },
  {
    "id": "b1f1a2b3-c4d5-4e67-8f89-a0b1c2d3e4f3",
    "user_id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "action": "RESULT_PUBLISHED",
    "entity_type": "RESULT",
    "entity_id": "90d9e0f1-a2b3-4c45-9d67-e8f9a0b1c234",
    "old_value": { "status": "READY_FOR_REVIEW" },
    "new_value": { "status": "PUBLISHED" },
    "created_at": "2026-10-22T08:15:00Z"
  },
  {
    "id": "b1f1a2b3-c4d5-4e67-8f89-a0b1c2d3e4f4",
    "user_id": "9a8b7c6d-5e4f-4a3b-8c2d-1e0f9a8b7c6d",
    "action": "RESULT_INVALIDATED",
    "entity_type": "RESULT",
    "entity_id": "91d9e0f1-a2b3-4c45-9d67-e8f9a0b1c345",
    "old_value": { "status": "READY_FOR_REVIEW" },
    "new_value": { "status": "INVALIDATED" },
    "created_at": "2026-10-22T09:40:00Z"
  }
]
```

## Relationships

- Many log entries are written by one user (`user_id` → `users.id`), or by the system when `NULL`.
- Logical link: `entity_type` and `entity_id` point to any record.

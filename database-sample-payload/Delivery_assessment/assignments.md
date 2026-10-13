# assignments

Part of the **Delivery** section. An assignment is the permission slip: "participant X may take assessment Y between these dates, this many times". The participant's real attempts are stored in `assessment_sessions`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the assignment |
| `assessment_id` | UUID | NOT NULL, FOREIGN KEY → `assessments.id` | The assessment being assigned. It must be `PUBLISHED`. |
| `participant_id` | UUID | NOT NULL, FOREIGN KEY → `participants.id` | The participant who may take it |
| `assigned_by` | UUID | NOT NULL, FOREIGN KEY → `users.id` | The staff user who created the assignment |
| `start_at` | TIMESTAMPTZ | NOT NULL | Earliest moment the participant can start |
| `deadline_at` | TIMESTAMPTZ | NOT NULL, CHECK `deadline_at > start_at` | Last moment the participant can start |
| `max_attempts` | INTEGER | NOT NULL, CHECK `>= 1` | Attempts allowed for this participant. Copied from `assessments.max_attempts`, and can be overridden. |
| `status` | VARCHAR(20) | NOT NULL, default `INVITED`, CHECK in (`INVITED`, `IN_PROGRESS`, `COMPLETED`, `EVALUATED`, `EXPIRED`, `CANCELLED`) | Lifecycle of the assignment |

**Composite constraint:** `UNIQUE (assessment_id, participant_id)`. UQ means a unique constraint. The same assessment cannot be assigned twice to the same participant. Extra tries are extra attempts, not extra assignments.

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `INVITED` | Created and link sent. The participant has not started. |
| `IN_PROGRESS` | At least one session has started. |
| `COMPLETED` | The participant has used all attempts or submitted the final one. Waiting for grading. |
| `EVALUATED` | Grading is finished and a result exists. |
| `EXPIRED` | `deadline_at` passed without a start. |
| `CANCELLED` | Revoked by staff. The link stops working. |

Transitions: `INVITED → IN_PROGRESS → COMPLETED → EVALUATED`. `INVITED → EXPIRED`. `INVITED` or `IN_PROGRESS → CANCELLED`.

## Start check

When the participant clicks Start, the backend allows a new session only if all of these hold:

1. The assignment belongs to the logged-in participant.
2. `status` is `INVITED` or `IN_PROGRESS`.
3. The current time is between `start_at` and `deadline_at`.
4. Existing sessions are fewer than `max_attempts`.
5. No session is currently `ACTIVE` or `PAUSED`.

## Sample payload

```json
{
  "id": "30c3d4e5-f607-4183-8ca4-b5d6e7f8091a",
  "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "participant_id": "10a1b2c3-d4e5-4f61-8a72-93b4c5d6e7f8",
  "assigned_by": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
  "start_at": "2026-10-20T04:00:00Z",
  "deadline_at": "2026-10-27T18:29:59Z",
  "max_attempts": 1,
  "status": "EVALUATED"
}
```

## Sample rows

```json
[
  {
    "id": "30c3d4e5-f607-4183-8ca4-b5d6e7f8091a",
    "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "participant_id": "10a1b2c3-d4e5-4f61-8a72-93b4c5d6e7f8",
    "assigned_by": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "start_at": "2026-10-20T04:00:00Z",
    "deadline_at": "2026-10-27T18:29:59Z",
    "max_attempts": 1,
    "status": "EVALUATED"
  },
  {
    "id": "31c3d4e5-f607-4183-8ca4-b5d6e7f8092b",
    "assessment_id": "6f5e4d3c-2b1a-4f0e-9d8c-7b6a5f4e3d2c",
    "participant_id": "20b2c3d4-e5f6-4072-9b83-a4c5d6e7f809",
    "assigned_by": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "start_at": "2026-10-22T04:00:00Z",
    "deadline_at": "2026-10-29T18:29:59Z",
    "max_attempts": 1,
    "status": "INVITED"
  },
  {
    "id": "32c3d4e5-f607-4183-8ca4-b5d6e7f8093c",
    "assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "participant_id": "21b2c3d4-e5f6-4072-9b83-a4c5d6e7f90a",
    "assigned_by": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "start_at": "2026-10-20T04:00:00Z",
    "deadline_at": "2026-10-27T18:29:59Z",
    "max_attempts": 1,
    "status": "EVALUATED"
  }
]
```

## Relationships

- Many assignments belong to one assessment (`assessment_id` → `assessments.id`).
- Many assignments belong to one participant (`participant_id` → `participants.id`).
- Many assignments are created by one user (`assigned_by` → `users.id`).
- One assignment has many `assessment_sessions` (one per attempt).

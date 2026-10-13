# participants

Part of the **Delivery** section. A participant is a candidate profile that can be assigned assessments. The person's name and email live in `users`, so every participant is linked to exactly one user account (role `PARTICIPANT`).

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the participant |
| `user_id` | UUID | NOT NULL, UNIQUE, FOREIGN KEY → `users.id` | Login account that holds the person's name and email. One user can be at most one participant. |
| `participant_type` | VARCHAR(20) | NOT NULL, CHECK in (`INTERNAL`, `EXTERNAL`, `CAMPUS`) | What kind of candidate this is |
| `reference_no` | VARCHAR(100) | NULL | ID from the organisation's external system (employee ID, ATS applicant ID, university roll number) |
| `job_role_id` | UUID | NOT NULL, FOREIGN KEY → `job_roles.id` | Job role the participant is being assessed for |
| `status` | VARCHAR(20) | NOT NULL, default `ACTIVE`, CHECK in (`ACTIVE`, `INACTIVE`) | Whether the participant can be given new assignments |

## Possible values of `participant_type`

| Value | Meaning |
|-------|---------|
| `INTERNAL` | An existing employee (promotion, upskilling) |
| `EXTERNAL` | A new job applicant |
| `CAMPUS` | A university student (campus hiring) |

The type drives data-retention rules, for example deleting `EXTERNAL` data after a set period.

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `ACTIVE` | Can be assigned assessments |
| `INACTIVE` | Kept for history. No new assignments. |

## Notes

- `reference_no` is used to send results back to the client's HR system. It is optional because not every participant comes from an external system. A suggested constraint is `UNIQUE (reference_no, participant_type)` where `reference_no IS NOT NULL`.
- The ERD has no name or email on `participants`, so identity is read through `user_id`. This is why `user_id` is required.
- Department is reached through `job_roles.department_id`. There is no department column here.

## Sample payload

```json
{
  "id": "10a1b2c3-d4e5-4f61-8a72-93b4c5d6e7f8",
  "user_id": "f1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5c",
  "participant_type": "EXTERNAL",
  "reference_no": "ATS-2026-0412",
  "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
  "status": "ACTIVE"
}
```

## Sample rows

```json
[
  {
    "id": "10a1b2c3-d4e5-4f61-8a72-93b4c5d6e7f8",
    "user_id": "f1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5c",
    "participant_type": "EXTERNAL",
    "reference_no": "ATS-2026-0412",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "status": "ACTIVE"
  },
  {
    "id": "20b2c3d4-e5f6-4072-9b83-a4c5d6e7f809",
    "user_id": "a2b3c4d5-e6f7-4081-9bac-1d2e3f4a5b6d",
    "participant_type": "CAMPUS",
    "reference_no": "ROLL-CS-2026-117",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "status": "ACTIVE"
  },
  {
    "id": "21b2c3d4-e5f6-4072-9b83-a4c5d6e7f90a",
    "user_id": "b3c4d5e6-f708-4192-8cbd-2e3f4a5b6c7e",
    "participant_type": "INTERNAL",
    "reference_no": "EMP-20418",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "status": "ACTIVE"
  }
]
```

## Relationships

- One-to-one with `users` (`user_id` → `users.id`).
- Many participants belong to one job role (`job_role_id` → `job_roles.id`).
- One participant has many `assignments`.
- One participant has many `participant_assessment_data` rows (one per personalised assessment).

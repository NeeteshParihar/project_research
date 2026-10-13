# assessments

## changes suggested
1. add evaluater in assessment
2. sketch out exact assessment settings

Part of the **Assessment Setup** section. One row is one version of an assessment (the "master" definition). Sections, pool rules and fixed tasks hang off it.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of this assessment version |
| `title` | VARCHAR(200) | NOT NULL | Assessment name shown to admins and candidates |
| `type` | VARCHAR(20) | NOT NULL, CHECK in (`GENERALISED`, `PERSONALISED`) | How tasks are chosen. See values below. |
| `job_role_id` | UUID | NOT NULL, FOREIGN KEY → `job_roles.id` | Job role this assessment is meant for |
| `duration_minutes` | INTEGER | NOT NULL, CHECK `> 0` | Time allowed per attempt, in minutes |
| `passing_percentage` | NUMERIC(5,2) | NOT NULL, CHECK between `0` and `100` | Minimum percentage a candidate needs to pass |
| `max_attempts` | INTEGER | NOT NULL, default `1`, CHECK `>= 1` | Default number of attempts a participant gets |
| `settings` | JSONB | NOT NULL, default `'{}'`, CHECK `jsonb_typeof(settings) = 'object'` | Proctoring, candidate-experience and security configuration |
| `status` | VARCHAR(20) | NOT NULL, default `DRAFT`, CHECK in (`DRAFT`, `PUBLISHED`, `ARCHIVED`) | Lifecycle state |
| `version` | INTEGER | NOT NULL, default `1`, CHECK `>= 1` | Version number in the chain |
| `previous_assessment_id` | UUID | NULL, FOREIGN KEY → `assessments.id` (self-reference) | The version this one was created from. `NULL` for version 1. |
| `created_by` | UUID | NOT NULL, FOREIGN KEY → `users.id` | User who created this version |

## Possible values of `type`

| Value | Meaning |
|-------|---------|
| `GENERALISED` | One assessment for many participants. Sections use pool rules, and tasks are randomly picked per participant at session start. |
| `PERSONALISED` | Built for one participant (for example from an uploaded resume). Sections use fixed tasks chosen when the assessment is generated. |

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `DRAFT` | Being built. Sections, pool rules and tasks can change. Cannot be assigned to participants. |
| `PUBLISHED` | Live. Can be assigned. **Locked**: sections, pool rules and fixed tasks can no longer change. |
| `ARCHIVED` | Retired. No new assignments. Kept for history and past results. |

Allowed transitions: `DRAFT → PUBLISHED → ARCHIVED`. A `DRAFT` can also be deleted.

## Versioning rule

A published assessment is immutable. To change it:

1. Copy it into a new row with `status = DRAFT`, `version = old version + 1`, and `previous_assessment_id` = the old row's `id`.
2. Edit the new draft and publish it.
3. Archive the old version when it is no longer needed.

Participants who already started keep the version they were assigned.

## `settings` JSONB: what it holds

Keys are optional and can grow over time without schema changes. A missing key means the feature is off.

| Group | Key | Type | Meaning |
|-------|-----|------|---------|
| Proctoring | `webcam_required` | boolean | Webcam must be on |
| Proctoring | `microphone_required` | boolean | Microphone must be on |
| Proctoring | `max_tab_switches` | integer | Tab switches allowed before the session is flagged |
| Proctoring | `force_fullscreen` | boolean | Candidate must stay in fullscreen |
| Proctoring | `id_verification` | boolean | ID check before starting |
| Candidate experience | `allow_calculator` | boolean | Show a built-in calculator |
| Candidate experience | `show_results_immediately` | boolean | Show the score right after submit |
| Candidate experience | `default_language` | string | Default coding language in the editor |

## Sample payload

```json
{
  "id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
  "title": "Backend Engineer Screening",
  "type": "GENERALISED",
  "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
  "duration_minutes": 90,
  "passing_percentage": 60.0,
  "max_attempts": 1,
  "settings": {
    "webcam_required": true,
    "microphone_required": false,
    "max_tab_switches": 3,
    "force_fullscreen": true,
    "id_verification": false,
    "allow_calculator": false,
    "show_results_immediately": false,
    "default_language": "java"
  },
  "status": "PUBLISHED",
  "version": 1,
  "previous_assessment_id": null,
  "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
}
```

## Sample rows (version chain and a personalised assessment)

```json
[
  {
    "id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "title": "Backend Engineer Screening",
    "type": "GENERALISED",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "duration_minutes": 90,
    "passing_percentage": 60.0,
    "max_attempts": 1,
    "settings": { "webcam_required": true, "max_tab_switches": 3, "force_fullscreen": true },
    "status": "ARCHIVED",
    "version": 1,
    "previous_assessment_id": null,
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  },
  {
    "id": "8a9b0c1d-2e3f-4a5b-9c6d-7e8f9a0b1c2d",
    "title": "Backend Engineer Screening",
    "type": "GENERALISED",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "duration_minutes": 100,
    "passing_percentage": 65.0,
    "max_attempts": 1,
    "settings": { "webcam_required": true, "max_tab_switches": 2, "force_fullscreen": true },
    "status": "DRAFT",
    "version": 2,
    "previous_assessment_id": "4c1d2e3f-5a6b-4c7d-8e9f-0a1b2c3d4e5f",
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  },
  {
    "id": "6f5e4d3c-2b1a-4f0e-9d8c-7b6a5f4e3d2c",
    "title": "Backend Engineer - Resume Based (Ananya Iyer)",
    "type": "PERSONALISED",
    "job_role_id": "9d1e2f3a-4b5c-4d6e-8f70-a1b2c3d4e5f6",
    "duration_minutes": 60,
    "passing_percentage": 50.0,
    "max_attempts": 1,
    "settings": { "webcam_required": true, "force_fullscreen": true },
    "status": "PUBLISHED",
    "version": 1,
    "previous_assessment_id": null,
    "created_by": "d4e5f6a7-b8c9-4d0e-9f1a-2b3c4d5e6f70"
  }
]
```

## Relationships

- Many assessments belong to one job role (`job_role_id` → `job_roles.id`).
- Many assessments are created by one user (`created_by` → `users.id`).
- One assessment has many `assessment_sections`.
- An assessment points to its previous version (`previous_assessment_id` → `assessments.id`).
# participant_assessment_data

Part of the **Delivery** section. Used for **`PERSONALISED` assessments only**. It stores what a participant submitted (their resume) and the progress of the AI building their assessment from it.

Flow: an ADMIN or HR creates a `PERSONALISED` assessment shell (`DRAFT`) for the participant, and an invite link goes out. The participant uploads a resume. The AI parses it and builds sections and fixed tasks. Once the assessment is published, an assignment is created.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the row |
| `assessment_id` | UUID | NOT NULL, FOREIGN KEY → `assessments.id` | The personalised assessment being built for this participant |
| `participant_id` | UUID | NOT NULL, FOREIGN KEY → `participants.id` | The participant the assessment is built for |
| `invite_token` | VARCHAR(128) | NOT NULL, UNIQUE | Hash of the one-time token in the resume-upload link. The plain token is never stored. |
| `token_expires_at` | TIMESTAMPTZ | NOT NULL | When the invite link stops working |
| `resume_url` | VARCHAR(500) | NULL | Storage location of the uploaded resume. `NULL` until uploaded. |
| `parsed_resume` | JSONB | NULL | Structured data the parser extracted from the resume |
| `generation_status` | VARCHAR(20) | NOT NULL, default `PENDING`, CHECK in (`PENDING`, `PARSING`, `GENERATING`, `READY`, `FAILED`) | Progress of building the assessment |
| `generation_meta` | JSONB | NULL | Technical details of the generation run |
| `resume_submitted_at` | TIMESTAMPTZ | NULL | When the participant uploaded the resume |
| `created_at` | TIMESTAMPTZ | NOT NULL, default `now()` | When the row was created |

**Composite constraint:** `UNIQUE (assessment_id, participant_id)`.

## Possible values of `generation_status`

| Value | Meaning |
|-------|---------|
| `PENDING` | Invite sent. Waiting for the resume. |
| `PARSING` | Resume received. Being parsed. |
| `GENERATING` | AI is building sections and picking tasks. |
| `READY` | Sections and fixed tasks are built. The assessment can be reviewed and published. |
| `FAILED` | Parsing or generation failed. See `generation_meta.error`. Can be retried. |

Transitions: `PENDING → PARSING → GENERATING → READY`. Any step can go to `FAILED`, and `FAILED → PARSING` on retry.

## JSONB contents

`parsed_resume`

| Key | Type | Meaning |
|-----|------|---------|
| `skills` | array of strings | Skills found in the resume |
| `total_experience_years` | number | Total experience |
| `roles` | array of objects | Past roles (`title`, `company`, `years`) |
| `education` | array of objects | Degrees (`degree`, `institution`, `year`) |

`generation_meta`

| Key | Type | Meaning |
|-----|------|---------|
| `generator_version` | string | Version of the generation logic |
| `attempts` | integer | How many runs were made |
| `started_at`, `finished_at` | timestamp strings | Timing of the last run |
| `error` | string or null | Failure reason when `FAILED` |

## Sample payload

```json
{
  "id": "60a6b7c8-d9e0-4f12-8a34-b5c6d7e8f901",
  "assessment_id": "6f5e4d3c-2b1a-4f0e-9d8c-7b6a5f4e3d2c",
  "participant_id": "20b2c3d4-e5f6-4072-9b83-a4c5d6e7f809",
  "invite_token": "9f2c1e8a4b7d6035c1a9e2f84d7b60a3c5e1f8294b7d6a03c1e9f2a84d7b6051",
  "token_expires_at": "2026-10-20T18:29:59Z",
  "resume_url": "s3://assessment-resumes/2026/10/20b2c3d4-resume.pdf",
  "parsed_resume": {
    "skills": ["Java", "Spring Boot", "PostgreSQL", "Docker"],
    "total_experience_years": 3,
    "roles": [
      { "title": "Software Engineer", "company": "Acme Tech", "years": 3 }
    ],
    "education": [
      { "degree": "B.Tech Computer Science", "institution": "State Engineering College", "year": 2023 }
    ]
  },
  "generation_status": "READY",
  "generation_meta": {
    "generator_version": "v1",
    "attempts": 1,
    "started_at": "2026-10-18T09:12:04Z",
    "finished_at": "2026-10-18T09:12:41Z",
    "error": null
  },
  "resume_submitted_at": "2026-10-18T09:11:50Z",
  "created_at": "2026-10-17T10:00:00Z"
}
```

## Relationships

- Many rows belong to one assessment (`assessment_id` → `assessments.id`).
- Many rows belong to one participant (`participant_id` → `participants.id`).

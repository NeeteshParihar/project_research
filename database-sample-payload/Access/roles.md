# roles

Part of the **Access** section (flat, permission-based RBAC). A role is a named bundle of permissions. Users get roles through `user_roles`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the role |
| `code` | VARCHAR(30) | NOT NULL, UNIQUE, CHECK `code = upper(code)` | Stable machine-readable key used in code (e.g. `HR`). Never changes. |
| `name` | VARCHAR(100) | NOT NULL | Human-readable label shown in the UI |
| `permissions` | JSONB | NOT NULL, default `'[]'`, CHECK `jsonb_typeof(permissions) = 'array'` | Array of permission strings in `<resource>:<action>` format |

## Possible values of `code`

`ADMIN`, `HR`, `CREATOR`, `EVALUATOR`, `PARTICIPANT`

## Permission string format

`<resource>:<action>`, all lowercase. Some actions carry a scope suffix that restricts the data the permission reaches:

- `_own`: only records belonging to the logged-in user.
- `_assigned`: only records assigned to the logged-in user.

The backend checks that the role's `permissions` array contains the required string. If it does not, it returns `403 Forbidden`.

## Permission catalogue

| Resource | Permissions |
|----------|-------------|
| Organisation | `department:read`, `department:manage`, `job_role:read`, `job_role:manage`, `job_role:update_hiring` |
| Users and access | `user:read`, `user:manage`, `role:read`, `role:assign` |
| Question bank | `task:read`, `task:create`, `task:update`, `task:submit_review`, `task:publish`, `task:retire`, `skill:read`, `skill:manage` |
| Assessments | `assessment:read`, `assessment:read_assigned`, `assessment:create`, `assessment:update`, `assessment:publish`, `assessment:archive` |
| Participants and assignments | `participant:read`, `participant:manage`, `assignment:read`, `assignment:create`, `assignment:cancel` |
| Delivery | `session:attempt_own`, `session:read`, `session:read_assigned` |
| Evaluation | `evaluation:read_assigned`, `evaluation:grade_manual` |
| Outcome | `result:read`, `result:read_assigned`, `result:read_own`, `result:publish`, `result:invalidate` |
| Monitoring | `proctoring:read`, `proctoring:read_assigned`, `audit_log:read` |

`manage` means create, update and deactivate for that resource.

---

## Role boundaries (suggested)

Design rules behind the boundaries:

1. **Separation of duties.** The people who write questions (CREATOR) are not the people who grade candidates (EVALUATOR), and neither of them decides hiring outcomes (HR).
2. **Least privilege.** Each role gets only what its job needs.
3. **Admin runs the platform, not the hiring.** ADMIN does not grade candidates, so no one can both configure the system and change a score unnoticed. Every ADMIN action is also written to `audit_logs`.
4. **PARTICIPANT is the most restricted role.** It can reach only its own session and its own published result.

### 1. ADMIN: platform owner

**Purpose:** set up and govern the platform.

| Area | Can do | Cannot do |
|------|--------|-----------|
| Organisation | Create and edit departments and job roles | n/a |
| Users and access | Create, edit and deactivate users, assign roles | n/a |
| Question bank | Read everything, publish and retire tasks, manage skills | Author tasks day to day (CREATOR's job) |
| Assessments | Read, publish and archive assessments | n/a |
| Participants | Read participants and assignments | Create participants (HR's job) |
| Outcome | Read all results, invalidate a result (for example confirmed cheating) | Publish results (HR's job) |
| Monitoring | Read proctoring events and audit logs | n/a |
| Evaluation | Nothing | Grade candidates manually |

```json
[
  "department:read", "department:manage",
  "job_role:read", "job_role:manage", "job_role:update_hiring",
  "user:read", "user:manage",
  "role:read", "role:assign",
  "task:read", "task:publish", "task:retire",
  "skill:read", "skill:manage",
  "assessment:read", "assessment:publish", "assessment:archive",
  "participant:read", "assignment:read",
  "session:read",
  "result:read", "result:invalidate",
  "proctoring:read", "audit_log:read"
]
```

### 2. HR: hiring owner

**Purpose:** run the hiring process. Brings candidates in, assigns assessments and publishes outcomes.

| Area | Can do | Cannot do |
|------|--------|-----------|
| Organisation | Read departments and job roles, update `is_open_for_hiring` and `number_of_positions` on job roles | Create or delete departments and job roles |
| Question bank | Nothing | Read or edit tasks (keeps questions confidential) |
| Assessments | Read assessments | Create, edit or publish them |
| Participants | Create, edit and deactivate participants | n/a |
| Assignments | Create, view and cancel assignments | n/a |
| Delivery | Read sessions | n/a |
| Outcome | Read all results, publish results | Invalidate results, change scores |
| Monitoring | Read proctoring events | Read audit logs |
| Users and access | Nothing | Manage users or roles |

```json
[
  "department:read",
  "job_role:read", "job_role:update_hiring",
  "assessment:read",
  "participant:read", "participant:manage",
  "assignment:read", "assignment:create", "assignment:cancel",
  "session:read",
  "result:read", "result:publish",
  "proctoring:read"
]
```

### 3. CREATOR: content author

**Purpose:** build the question bank and assessments.

| Area | Can do | Cannot do |
|------|--------|-----------|
| Question bank | Read, create and update tasks (own drafts), submit tasks for review | Publish or retire tasks (needs ADMIN) |
| Skills | Read skills | Manage skills |
| Assessments | Read, create and update assessments (sections, pool rules) | Publish or archive them |
| Participants and results | Nothing | See any candidate, session or result data |
| Users and access | Nothing | Manage users or roles |

Four-eyes principle: the CREATOR drafts, an ADMIN publishes. A published task or assessment is immutable, so changes create a new version.

```json
[
  "task:read", "task:create", "task:update", "task:submit_review",
  "skill:read",
  "assessment:read", "assessment:create", "assessment:update"
]
```

### 4. EVALUATOR: grader / reviewer

**Purpose:** review and manually grade answers that need human judgment (subjective, AI-assisted or disputed answers).

| Area | Can do | Cannot do |
|------|--------|-----------|
| Delivery | Read sessions assigned to them | See sessions not assigned to them |
| Evaluation | Read assigned evaluations, grade manually, leave feedback | n/a |
| Assessments | Read assessments they are assigned to | Edit assessments |
| Outcome | Read results of assigned sessions | Publish or invalidate results |
| Monitoring | Read proctoring events of assigned sessions | Read audit logs |
| Question bank | Nothing | Edit tasks |
| Participants | Nothing | Read participant contact details |

```json
[
  "assessment:read_assigned",
  "session:read_assigned",
  "evaluation:read_assigned", "evaluation:grade_manual",
  "result:read_assigned",
  "proctoring:read_assigned"
]
```

### 5. PARTICIPANT: candidate

**Purpose:** take assessments assigned to them.

| Area | Can do | Cannot do |
|------|--------|-----------|
| Delivery | Start and attempt their own session, save answers, submit | Open anyone else's session |
| Assessments | Read the assessment assigned to them | See pool rules or other assessments |
| Outcome | Read their own result, only after it is `PUBLISHED` | See results before publishing or anyone else's |
| Everything else | Nothing | Reach any admin, HR, creator or evaluator function |

```json
[
  "assessment:read_assigned",
  "session:attempt_own",
  "result:read_own"
]
```

---

## Role comparison

| Permission area | ADMIN | HR | CREATOR | EVALUATOR | PARTICIPANT |
|-----------------|:-----:|:--:|:-------:|:---------:|:-----------:|
| Manage departments / job roles | ✅ | ❌ | ❌ | ❌ | ❌ |
| Update job role hiring fields | ✅ | ✅ | ❌ | ❌ | ❌ |
| Manage users and roles | ✅ | ❌ | ❌ | ❌ | ❌ |
| Create / edit tasks | ❌ | ❌ | ✅ | ❌ | ❌ |
| Publish / retire tasks | ✅ | ❌ | ❌ | ❌ | ❌ |
| Create / edit assessments | ❌ | ❌ | ✅ | ❌ | ❌ |
| Publish / archive assessments | ✅ | ❌ | ❌ | ❌ | ❌ |
| Manage participants | ❌ | ✅ | ❌ | ❌ | ❌ |
| Create assignments | ❌ | ✅ | ❌ | ❌ | ❌ |
| Attempt assessment | ❌ | ❌ | ❌ | ❌ | ✅ (own) |
| Grade manually | ❌ | ❌ | ❌ | ✅ (assigned) | ❌ |
| Read results | ✅ all | ✅ all | ❌ | ✅ assigned | ✅ own, published |
| Publish results | ❌ | ✅ | ❌ | ❌ | ❌ |
| Invalidate results | ✅ | ❌ | ❌ | ❌ | ❌ |
| Read proctoring events | ✅ | ✅ | ❌ | ✅ assigned | ❌ |
| Read audit logs | ✅ | ❌ | ❌ | ❌ | ❌ |

## Sample payload

```json
{
  "id": "e1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b",
  "code": "EVALUATOR",
  "name": "Evaluator",
  "permissions": [
    "assessment:read_assigned",
    "session:read_assigned",
    "evaluation:read_assigned",
    "evaluation:grade_manual",
    "result:read_assigned",
    "proctoring:read_assigned"
  ]
}
```

## Relationships

- Many-to-many with `users` through `user_roles`.
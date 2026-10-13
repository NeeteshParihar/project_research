# user_roles

Part of the **Access** section. Join table that gives users their roles. One user can hold several roles (for example CREATOR and EVALUATOR), and one role is held by many users.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `user_id` | UUID | NOT NULL, FOREIGN KEY → `users.id`, part of composite PRIMARY KEY | The user receiving the role |
| `role_id` | UUID | NOT NULL, FOREIGN KEY → `roles.id`, part of composite PRIMARY KEY | The role being granted |

**Composite constraint:** `PRIMARY KEY (user_id, role_id)`. The same role cannot be granted twice to the same user.

## Effective permissions

A user's effective permissions are the union of the `permissions` arrays of all their roles.

Example: a user with CREATOR and EVALUATOR roles can create tasks and also grade assigned answers.

## Role combination guidance (suggested)

| Combination | Allowed? | Reason |
|-------------|----------|--------|
| ADMIN + any other staff role | Allowed | Convenience for small teams, but each action is still audited |
| CREATOR + EVALUATOR | Discouraged | Breaks separation of duties: the same person writes and grades |
| PARTICIPANT + any staff role | Not allowed | A candidate must never hold staff authority |

These rules are enforced in application logic, not in the database.

## Sample payload

```json
{
  "user_id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
  "role_id": "e1a2b3c4-d5e6-4f70-8a9b-0c1d2e3f4a5b"
}
```

## Sample rows

```json
[
  {
    "user_id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "role_id": "<role_id of HR>"
  },
  {
    "user_id": "b8c9d0e1-f2a3-4b4c-9d5e-6f7a8b9c0d1e",
    "role_id": "<role_id of CREATOR>"
  },
  {
    "user_id": "c9d0e1f2-a3b4-4c5d-8e6f-7a8b9c0d1e2f",
    "role_id": "<role_id of PARTICIPANT>"
  }
]
```

## Relationships

- Many `user_roles` rows belong to one user (`user_id` → `users.id`).
- Many `user_roles` rows belong to one role (`role_id` → `roles.id`).
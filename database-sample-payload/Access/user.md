# users

Part of the **Access** section. A user is an account that can log in. Staff (ADMIN, HR, CREATOR, EVALUATOR) and candidates who log in (PARTICIPANT) are all stored here. What each account can do is decided by its roles in `user_roles`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the user |
| `name` | VARCHAR(150) | NOT NULL | Full name of the user |
| `email` | VARCHAR(255) | NOT NULL, UNIQUE, stored lowercase | Login identifier. No two users can share an email. |
| `password_hash` | VARCHAR(255) | NOT NULL | Hashed password (for example bcrypt or argon2). The plain password is never stored. |
| `status` | VARCHAR(20) | NOT NULL, default `ACTIVE`, CHECK in (`ACTIVE`, `INACTIVE`, `SUSPENDED`) | Current state of the account |

## Possible values of `status`

| Value | Meaning |
|-------|---------|
| `ACTIVE` | Account can log in normally. |
| `INACTIVE` | Account is disabled (for example the person left the organisation). Login is blocked and history is kept. |
| `SUSPENDED` | Account is temporarily blocked by an admin (for example suspected compromise). It can be reactivated. |

## Sample payload

```json
{
  "id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
  "name": "Neha Sharma",
  "email": "neha.sharma@example.com",
  "password_hash": "$2b$12$KIXQy1m0Zr7Qh0pJ3Y5wOe8vVx9uFhZkq1m2nBvC3dE4fG5hI6jK7",
  "status": "ACTIVE"
}
```

## Sample rows

```json
[
  {
    "id": "a7b8c9d0-e1f2-4a3b-8c4d-5e6f7a8b9c0d",
    "name": "Neha Sharma",
    "email": "neha.sharma@example.com",
    "password_hash": "$2b$12$KIXQy1m0Zr7Qh0pJ3Y5wOe8vVx9uFhZkq1m2nBvC3dE4fG5hI6jK7",
    "status": "ACTIVE"
  },
  {
    "id": "b8c9d0e1-f2a3-4b4c-9d5e-6f7a8b9c0d1e",
    "name": "Rahul Verma",
    "email": "rahul.verma@example.com",
    "password_hash": "$2b$12$Lm0n1o2p3q4r5s6t7u8v9eAbCdEfGhIjKlMnOpQrStUvWxYz0123",
    "status": "INACTIVE"
  },
  {
    "id": "c9d0e1f2-a3b4-4c5d-8e6f-7a8b9c0d1e2f",
    "name": "Ananya Iyer",
    "email": "ananya.iyer@example.com",
    "password_hash": "$2b$12$Zy9x8w7v6u5t4s3r2q1p0oNmLkJiHgFeDcBaZyXwVuTsRqPoNmLk",
    "status": "SUSPENDED"
  }
]
```

## Notes

- A participant who logs in has a `users` row with the PARTICIPANT role. It links to the `participants` table 1:1 through `participants.user_id` (unique).
- Staff accounts are not linked to `participants`.

## Relationships

- Many-to-many with `roles` through `user_roles`.
- One-to-one (optional) with `participants` through `participants.user_id`.

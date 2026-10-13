# Database Sample Payloads

This directory stores representative, realistic sample payload data (JSON schema & mock records) for all 23 tables in the Codility-style Assessment Engine ERD (`updated_erd_v8.png`), aligned with architectural decisions documented in `debate.md` and `deabte-answers.md`.

## ERD Sections Overview

1. **`organisation-payload/`**
   - `departments`
   - `job_roles`

2. **`access-payload/`** (or `users-payload/`)
   - `users`
   - `roles`
   - `user_roles`

3. **`question-bank-payload/`**
   - `tasks`
   - `task_options`
   - `skills`
   - `test_cases`

4. **`assessment-setup-payload/`**
   - `assessments`
   - `assessment_sections`
   - `section_pool_rules`
   - `assessment_tasks`

5. **`delivery-payload/`**
   - `participants`
   - `participant_assessment_data`
   - `assignments`
   - `assessment_sessions`
   - `answers`
   - `submissions`

6. **`outcome-payload/`**
   - `results`
   - `skill_scores`

7. **`monitoring-payload/`**
   - `audit_logs`
   - `proctoring_events`

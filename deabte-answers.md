
# Section - 4 ( ERD-Assesment-section ) 

Based on the context provided in your proposal, your design approach for generating dynamic, randomized assessments using task pooling is excellent. Here is the architectural reasoning for the specific fields in your ERD to help you finalize the implementable version.

### A. `assessments` (or `assessment_versions`)

**1. Why we need a `previous_version_id` here:**
In a production-grade assessment platform (like HackerRank or Codility), once an assessment is published and candidates start taking it, **it must become immutable**. If an admin wants to change a question or update a setting, you shouldn't overwrite the existing record, as that would invalidate the results of candidates who already took it.
*   **Audit Trail & Lineage:** `previous_version_id` allows you to create a linked list of changes (e.g., `v1 -> v2 -> v3`).
*   **Analytics:** It allows you to track the history of the test and compare metrics across versions (e.g., *"Did the pass rate drop when we introduced version 3 compared to version 2?"*).
*   **Rollbacks:** It gives admins the ability to easily revert to a previous, known-good state if the new version has issues.

**2. What are we going to store in the `settings` JSONB? (Is it feasible?)**
*   **Feasibility:** Yes, using PostgreSQL's `JSONB` for settings is not just feasible, it is the industry standard for this exact use case. `JSONB` is highly optimized, can be indexed (using GIN indexes), and prevents your table from becoming bloated with dozens of columns.
*   **What to store inside it:** You should store any configuration that doesn't need to be strictly queried in relational joins.
    *   **Proctoring Settings:** `{"webcam_required": true, "microphone_required": false, "max_tab_switches": 3, "force_fullscreen": true, "id_verification": true}`
    *   **Candidate Experience:** `{"allow_calculator": true, "show_results_immediately": false, "default_language": "python"}`
    *   **Security/Access:** `{"ip_whitelist": ["192.168.1.1"], "browser_restriction": "safe_exam_browser"}`
*   **Why JSON over columns?** Assessment platforms constantly add new features (e.g., AI gaze tracking). If you use JSONB, you don't need to run a database schema migration (`ALTER TABLE`) every time you add a new toggle switch to the admin UI. Additionally, un-proctored tests won't have a bunch of empty `NULL` columns.

---

### B. `section_pool_rules`

**1. Why are we not storing `assessment_id` here too (for optimized indexing)?**
*   **Normalization (3NF):** A pool rule belongs directly to a **Section**, and the **Section** belongs to an **Assessment**. Storing `assessment_id` inside the pool rule table is redundant data (denormalization).
*   **Database Performance:** You might think adding `assessment_id` helps you skip a `JOIN` and makes fetching faster, but relational databases (like PostgreSQL) are incredibly highly optimized for joining on indexed Foreign Keys. The query `SELECT rules FROM section_pool_rules JOIN assessment_sections ON ... WHERE assessment_id = X` will execute in milliseconds.
*   **Data Integrity:** If you store `assessment_id` in the rule, you risk anomalies. What if the `assessment_id` in the rule contradicts the `assessment_id` of the section it belongs to?
*   **Verdict for Implementation:** Keep it normalized. Do not add `assessment_id` to `section_pool_rules`. Rely on standard indexing for the `section_id` foreign key.

**2. What are we going to store in the `skill_filter` JSON?**
As you mentioned in `debate.md`, the goal is to create randomized tasks for mass assignments. The `skill_filter` is the **query configuration** that your backend will use to pull random questions from the global question bank when a candidate clicks "Start".
*   **What to store inside it:**
    *   **Target Skills:** `{"must_have_skills": ["java", "spring-boot"]}`
    *   **Difficulty Constraints:** `{"difficulty_range": {"min": 2, "max": 4}}` or `{"difficulty": ["medium", "hard"]}`
    *   **Question Types:** `{"allowed_types": ["coding", "mcq_multiple_correct"]}`
    *   **Tag Exclusions:** `{"exclude_tags": ["deprecated", "internal_only"]}`
*   **How it is used in the flow:** When the `assessment_session` is created (Step 6 in your flow), your backend reads this `skill_filter` JSON, translates it into an SQL query against the `tasks` table, applies a `ORDER BY RANDOM() LIMIT X` (where X is the number of questions needed for that section), and assigns those resulting task IDs to the candidate's `session_tasks`.


# Section -5  ( delivery section of ERD ) 

Here is a clear-cut explanation of the attributes in the Delivery section of your ERD. I have aligned these explanations with how enterprise assessment platforms (like HackerRank, Codility, or Mercer|Mettl) handle data for mass-scale, randomized testing.

---
### A. Participants
**1. Why `dept` (Department) attribute?**
In an enterprise context, you assess people for specific roles or teams (e.g., "Engineering", "Sales", "Campus Hiring 2026"). Storing the department helps admins filter and aggregate analytics later (e.g., "What is the average score of the Engineering department vs. the Sales department?").

**2. Why `reference_no` and what is it?**
This is a **crucial** field for integrations. It stores the unique ID from the client's external system (like an Employee ID, an Applicant Tracking System (ATS) ID from Workday/Greenhouse, or a University Roll Number). When the assessment is done, your system uses this `reference_no` to send the score back to the correct candidate profile in the client's HR system.

**3. `participant_type`?**
This categorizes the candidate to determine retention policies or permissions.
*   **Values:** `INTERNAL` (existing employee for promotion), `EXTERNAL` (new job applicant), `CAMPUS` (university student). For example, GDPR rules might require you to delete `EXTERNAL` candidate data after 6 months, but keep `INTERNAL` data forever.

---

### B. Assignments
**1. How `status` will look alike:**
This tracks the lifecycle of the invitation sent to the participant:
*   `SCHEDULED` / `INVITED`: The email link was sent, but the candidate hasn't clicked it.
*   `IN_PROGRESS`: The candidate has clicked start.
*   `COMPLETED`: The test was submitted successfully.
*   `EVALUATED`: The backend engine/evaluator has finished grading.
*   `EXPIRED`: The deadline passed before they attempted it.
*   `CANCELLED`: The admin revoked the link.

**2. What does `UQ` stand for?**
It stands for **Unique Constraint**. It means the combination of attributes marked with UQ must be strictly unique in the database. For example, `UQ(participant_id, assessment_version_id)` ensures you cannot accidentally assign the exact same test to the exact same candidate twice.

--- 
### C. assessment_sessions
**1. What is `submit_type`?**
This tells you exactly *how* the candidate's test ended, which is vital for resolving disputes:
*   `MANUAL`: Candidate explicitly clicked the "Submit Test" button.
*   `AUTO_SUBMIT`: Time ran out, and the system forced the submission.
*   `PROCTORING_VIOLATION`: The system auto-terminated the test because the candidate was caught cheating.
*   `ADMIN_FORCED`: An admin manually ended their session from the dashboard.

**2. `Status`?**
Similar to Assignment status, but specific to the live test instance:
*   `ACTIVE`: The candidate is currently typing in the test window.
*   `PAUSED`: If network drops and the test freezes.
*   `SUBMITTED`: Test is locked, awaiting grading.

---

### D. assessment_session_tasks (Answers / session tasks)
*This table maps the specific, randomized question assigned to the candidate.*

**1. Why `pool_rule_id`?**
Because questions are generated randomly, you need to track *why* this specific question was given to this candidate. If an admin says, *"Candidates are failing too much,"* you can group by `pool_rule_id` to see if a specific rule (e.g., "Hard Java Questions") is generating questions that are too difficult, rather than blaming the whole assessment.

**2. Why `display_order`?**
If questions are shuffled, Candidate A sees "Question X" as #1, but Candidate B sees it as #5. If a candidate emails support saying, *"Question 3 was broken,"* you need `display_order` to know which question was actually #3 on their specific screen.

**3. Why store `marks` and `negative_marks` if they are configured by section?**
**Immutability.** If a candidate starts a test, their grading rubric must be locked in. If an admin edits the Assessment Section configuration to change negative marking from -1 to -2 *while* candidates are taking the test, it shouldn't affect active sessions. Copying the marks into the session task guarantees the evaluation engine knows the exact rules at the moment the session was created.

**4. Why `answer_json` and what will it look like?**
A fixed text column won't work because different task types have entirely different answer structures:
*   **MCQ:** `{"selected_options": ["A", "C"]}`
*   **Coding:** `{"code": "def solve(): return 1", "language": "python", "theme": "dark"}`
*   **Subjective:** `{"text": "My essay answer..."}`

**5. `answer_status`:**
Tracks the state of the individual question:
*   `UNATTEMPTED`: Candidate never saw it.
*   `SAVED`: Autosaved, but candidate hasn't locked it in.
*   `MARKED_FOR_REVIEW`: Candidate clicked a "Flag" icon to revisit it later.

**6. `eval_status`:**
Tracks the backend grading process for this question:
*   `PENDING_EVALUATION`: Waiting in the background queue.
*   `COMPLETED`: Scored successfully.
*   `ERROR`: The code compilation engine crashed or timed out.

**7. `llm_evaluation_jsonb`:**
If you use AI to grade subjective answers or analyze code quality:
*   `{"ai_score": 8, "reasoning": "Good logic but missed boundary conditions", "time_complexity": "O(N)"}`

**8. Why is `feedback` here?**
This is for human reviewers. If a recruiter or technical interviewer manually reviews the candidate's code after the test, they can leave a note here (e.g., *"Messy code structure, ask about this in the interview"*).

---

### E. assessment_session_task_submission
*In coding assessments, candidates don't just submit once. They click "Run Code" multiple times to test their logic. This table stores EVERY run.*

**1. `status`? What will it define?**
This is the execution status of that specific code run (Standard Competitive Programming statuses):
*   `QUEUED` -> `PROCESSING` -> `ACCEPTED`
*   `WRONG_ANSWER`
*   `TIME_LIMIT_EXCEEDED` (Code took too long)
*   `COMPILATION_ERROR` (Syntax error)
*   `RUNTIME_ERROR` (e.g., Null pointer exception)

**2. `verdict`?**
A simple, high-level outcome for UI coloring: `PASS`, `FAIL`, or `PARTIAL` (e.g., passed 3 out of 5 hidden test cases).

**3. What is `test_results` JSONB and why is it here (since similar attributes are in session_tasks)?**
*   `session_tasks` stores the **final, overall result** for the question.
*   `assessment_session_task_submission` stores the **detailed logs of a single execution run**.
*   **What it looks like:** `{"test_cases_passed": 2, "test_cases_failed": 3, "memory_used_mb": 12.5, "execution_time_ms": 140, "details": [{"case": 1, "status": "PASS"}, {"case": 2, "status": "FAIL", "output": "Index Out of Bounds"}]}`
*   **Why we need both:** A candidate might hit "Run" 10 times. You need to store all 10 submissions in this table to show them their history or catch brute-forcing. But when calculating their final grade, the system just looks at the final score saved in `session_tasks`.

# Section - 6  ( Acces section ERD )  


Excellent, moving on to the **Access** section of the ERD. This section dictates who can log into the system and what they are allowed to do.

Here is the breakdown for `users` and `roles`.

### 1. `users` vs. `participants`: Why and how are they related?

In an enterprise assessment platform, you must distinguish between the people **administering** the system and the people **taking** the tests.

*   **`users` table:** This represents internal accounts with login credentials to the admin dashboard (e.g., HR Managers, Technical Interviewers, Content Creators, System Admins).
*   **`participants` table:** This represents the candidate profile taking a specific test.

**How and why they are related (usually an optional `user_id` foreign key in the `participants` table):**

1.  **External Candidates (No relation):** If you invite a college student to take a coding test via email, they just get a magic link. They do **not** need a dashboard account. A `participant` record is created for them, but `user_id` is left `NULL`.
2.  **Internal Employees (Linked):** If a company uses your platform for internal promotions or upskilling, an existing employee (who already has a `user` account to log in) might be assigned an assessment. In this case, `participant.user_id` points to their `users.id`. This prevents data duplication and links their test results directly to their permanent employee profile.
3.  **Candidate Portals (Linked):** If you eventually build a "Candidate Portal" where job seekers can log in to view their past test scores across multiple applications, they will be given a restricted `user` account (Role: Candidate), which links to their `participant` records.

---

### 2. `roles` and `permissions`: What it looks like and how it works

This is the core of **Role-Based Access Control (RBAC)**. You create a Role (e.g., "HR Admin"), attach Permissions to that role, and then assign the Role to a User.

**What the `permissions` attribute looks like:**
If stored as a `JSONB` array or a PostgreSQL `TEXT[]` array, permissions are usually formatted as **`<resource>:<action>`**. This makes them highly granular and easy for your backend APIs to parse.

Examples of permission strings:
*   `assessment:create`
*   `assessment:read`
*   `assessment:edit`
*   `task_bank:read`
*   `task_bank:write`
*   `results:view`
*   `candidate:invite`
*   `evaluation:grade_manual`

**How it looks in the Database (JSONB Example):**
For an **"Interviewer"** role, the `permissions` JSONB column might look like this:
```json
[
  "assessment:read", 
  "results:view", 
  "evaluation:grade_manual"
]
```
*(Notice they cannot create assessments or invite candidates).*

For a **"Content Creator"** role:
```json
[
  "task_bank:read", 
  "task_bank:write", 
  "assessment:create",
  "assessment:edit"
]
```
*(Notice they cannot view candidate results or grades).*

**How it works in practice:**
When User X tries to click the "Edit Question" button, your backend server intercepts the request. It looks at User X's assigned role, checks the `permissions` JSON array, and asks: *"Does this array contain `task_bank:write`?"* If yes, the save goes through. If no, the server throws a `403 Forbidden` error.


# Section - 7 

Perfect. The **Outcome** section is what your clients (HRs, Recruiters, and Hiring Managers) care about the most. This data powers the dashboards, leaderboards, and hiring decisions.

Here is the breakdown for `results` and `skill_scores`.

### A. Results
*This table stores the final, rolled-up outcome of a completed assessment session.*

**1. `status`?**
This tracks the lifecycle of the final report itself:
*   `CALCULATING`: The test is submitted, and background workers (or LLMs) are actively grading it.
*   `READY_FOR_REVIEW`: Grading is done, but it requires a human recruiter/interviewer to look at it before a final hiring decision is made.
*   `PUBLISHED`: The result is finalized. (If your platform allows candidates to see their scores, this status makes it visible to them).
*   `INVALIDATED` / `REJECTED`: The test was discarded (usually because the candidate was caught cheating).

**2. `section_score` JSONB**
Assessments have a dynamic number of sections (e.g., Section 1: Aptitude, Section 2: Coding, Section 3: Communication). Instead of creating complex SQL joins just to render the recruiter's dashboard, you pre-calculate and store the breakdown here.
*   **What it looks like:**
    `{"aptitude": {"score": 15, "max": 20}, "coding": {"score": 40, "max": 50}}`
*   **Why it's useful:** The frontend UI can instantly parse this JSON to draw pie charts or bar graphs showing the candidate's strengths without hammering the database.

**3. `integrity_summary` JSONB**
During a test, your proctoring engine might generate hundreds of individual event logs (e.g., "candidate looked away at minute 4", "candidate looked away at minute 7"). Recruiters don't have time to read every log. This JSONB stores the rolled-up summary.
*   **What it looks like:**
    `{"tab_switches_count": 5, "copy_paste_attempts": 2, "multiple_faces_detected": 1}`
*   **Why it's useful:** It gives the hiring manager a 5-second snapshot to decide if they should investigate further.

**4. `integrity_flagged` (Boolean)**
A simple `TRUE` or `FALSE` column.
*   **Why it's there:** If a candidate crosses a threshold (e.g., more than 3 tab switches), the backend sets this to `TRUE`. On the recruiter's dashboard, this puts a big red **⚠ Flagged** icon next to the candidate's name. It allows recruiters to instantly filter the leaderboard: `SELECT * FROM results WHERE integrity_flagged = FALSE ORDER BY score DESC`.

---

### B. How the `skill_scores` will be used
In a modern hiring platform, a raw total score (e.g., "85/100") is not enough information. You need to know *what* the candidate is actually good at. Because you linked Tasks (Questions) to Skills in the earlier stages, the backend can now calculate a score per skill.

**How it is used by the system and clients:**
1.  **Granular Hiring Decisions:** A candidate might score 70/100 overall. Without skill scores, they might be rejected. But with skill scores, a recruiter might see: `{"React": 100%, "Node.js": 95%, "SQL": 10%}`. If the job is strictly a Frontend React role, they will hire this person despite the low total score.
2.  **Global Candidate Profiling:** If a candidate takes three different assessments over two years, your platform can aggregate their `skill_scores` to build a global profile (e.g., "This candidate is a Level 4 Python Developer").
3.  **Internal Mobility (L&D):** For internal employee testing, if an employee gets a low score in the "Cloud Architecture" skill, the HR system can automatically recommend or assign training courses for that specific gap.
4.  **Benchmarking:** Companies can set strict thresholds. E.g., *"To pass the screening round, candidate must score >70% overall, AND >80% on the 'System Design' skill."*




# Section - 8 ( Monitoring )

Great! The **Monitoring** section is critical for maintaining the security, integrity, and compliance of your platform. Enterprise clients demand strict proctoring and audit trails.

Here is the breakdown for `proctoring_events` and `audit_logs`.

---

### A. proctoring_events
*This table logs every single suspicious action a candidate takes during their live test session.*

**1. `event_type` -> What is it?**
This is an Enum/String that categorizes the specific violation or anomaly detected by the browser or AI proctoring engine.
*   **Examples:**
    *   `TAB_SWITCH`: Candidate clicked away to another browser tab.
    *   `WINDOW_RESIZE` / `EXIT_FULLSCREEN`: Candidate exited fullscreen mode.
    *   `COPY_PASTE`: Candidate pasted text into the code editor.
    *   `FACE_NOT_DETECTED`: The webcam couldn't find a face (candidate walked away).
    *   `MULTIPLE_FACES`: The webcam detected two people looking at the screen.
    *   `AUDIO_ANOMALY`: The microphone picked up talking/voices.

**2. What is `details` JSONB?**
Because every `event_type` is completely different, they require different metadata. You can't make a flat column for all of them. `details` stores the context of *how* the violation happened.
*   **Examples:**
    *   For a `TAB_SWITCH`: `{"duration_away_seconds": 15}`
    *   For `COPY_PASTE`: `{"pasted_character_count": 450, "snippet": "public static void main..."}`
    *   For `MULTIPLE_FACES`: `{"confidence_score": 98.5, "snapshot_url": "s3://bucket/evidence/img_45.jpg"}`
    *   For `AUDIO_ANOMALY`: `{"decibel_level": 75, "duration_seconds": 4}`

---

### B. audit_logs
*This table tracks **Admin/User activity**, NOT candidate activity. It records "Who did what, when, and to what?"*

**1. Explain why this table exists:**
*   **Security & Accountability:** If a candidate's score is mysteriously changed from 40 to 95, or an entire assessment gets deleted right before a mass hiring drive, you need to know exactly which Admin/HR user did it.
*   **SOC2 Compliance:** Enterprise clients (like banks or large tech companies) will not buy your software unless you can prove you have a strict, immutable log of all system changes.
*   **Debugging:** If the system is misbehaving, developers can check the audit log to see what configuration was recently changed by a user.

**2. Explain each attribute's purpose and what it looks like:**

Assuming standard audit log attributes, here is how they function:

*   **`actor_id` (UUID):**
    *   **Purpose:** Identifies *who* performed the action. Points to the `users` table.
    *   **Looks like:** `550e8400-e29b-41d4-a716-446655440000` (or `NULL` if done by the system automatically).
*   **`action` / `event_name` (String):**
    *   **Purpose:** A clear, system-defined string of what happened.
    *   **Looks like:** `ASSESSMENT_PUBLISHED`, `CANDIDATE_SCORE_UPDATED`, `USER_ROLE_CHANGED`, `SECTION_POOL_RULE_DELETED`.
*   **`entity_type` (String):**
    *   **Purpose:** The type of database record that was altered.
    *   **Looks like:** `assessment`, `result`, `user`, `role`.
*   **`entity_id` (UUID):**
    *   **Purpose:** The exact ID of the record that was altered, so you can track the history of a specific object.
*   **`old_values` (JSONB) & `new_values` (JSONB):**
    *   **Purpose:** The most important columns. They show the exact "Before" and "After" states of the change.
    *   **Looks like:**
        *   `old_values`: `{"passing_score": 60, "max_attempts": 1}`
        *   `new_values`: `{"passing_score": 75, "max_attempts": 2}`
*   **`ip_address` (String) & `user_agent` (String):**
    *   **Purpose:** Security tracking. If an admin account was hacked, you can see the action came from an unknown IP address or a different browser.
    *   **Looks like:** IP: `192.168.1.45`, User Agent: `Mozilla/5.0 (Windows NT 10.0; Win64; x64)...`
*   **`created_at` (Timestamp):**
    *   **Purpose:** The exact millisecond the action occurred.


# Section - 9 ( Question Section ERD ) 

This is the final piece of the puzzle! The **Question Bank (Tasks)** section is the engine that drives the whole platform. It needs to be flexible enough to support Multiple Choice, Coding, and Subjective questions all in the same schema.

Here is the breakdown for the Question Bank entities.

---

### A. Tasks
**1. `skills` JSON?**
*   **What it is:** A denormalized array of skill tags attached to the question (e.g., `["Java", "Spring Boot", "API"]`).
*   **Why JSON instead of a join table?** If you are running complex, randomized "Pool Rules" to fetch thousands of questions rapidly, scanning a JSONB column (using PostgreSQL GIN indexes) is often faster and cleaner than doing multi-table joins on a `task_skills` mapping table.

**2. `content` JSON?**
*   **What it is:** The actual question data. Because different task types have entirely different structures, a single JSON column is the cleanest design.
*   **Looks like:**
    *   *For Coding:* `{"problem_statement": "Write a function to...", "constraints": "1 <= N <= 10^5", "input_format": "...", "output_format": "..."}`
    *   *For MCQ:* `{"question_text": "What is the time complexity of QuickSort?"}`

**3. `status`?**
*   Tracks the lifecycle of the question.
*   **Values:** `DRAFT` (still being written), `IN_REVIEW`, `PUBLISHED` (live and ready to be pulled into assessments), `RETIRED` / `ARCHIVED` (deprecated, shouldn't be used in new tests).

**4. `default_marks` -> Is it needed if we mark by section?**
*   **You are absolutely right in your thinking!** If your platform enforces that *all* questions in a Section have the same weight (e.g., "Section A: 10 questions, 5 marks each"), then this column is redundant.
*   *However*, if you ever want to allow a section to have "Mixed Weights" (e.g., a simple FizzBuzz coding question is naturally worth 10 marks, but a complex System Design question is naturally worth 50 marks), the `default_marks` column gives the question an inherent baseline value, which the section can either use or override.

**5. Why is `prev_task_id` maintained? (Versioning)**
*   **Immutability.** Once a question is `PUBLISHED` and candidates are taking it, you **cannot** overwrite it. If an author fixes a typo or adds a missing test case, you must create a brand-new row in the database.
*   `prev_task_id` points back to the old row, creating a version history. This ensures that active candidates finish their test on the exact version they started, while new candidates get the fixed version.

---

### B. `skills` -> How it will be used
This is the master taxonomy table of all concepts you test for (e.g., Java, React, Communication, SQL).
*   **Usage 1 (Tagging):** Authors tag Tasks with these skills.
*   **Usage 2 (Generation):** The `section_pool_rules` use these skills to randomly generate the test (e.g., "Give me 5 questions tagged with `Java`").
*   **Usage 3 (Reporting):** The backend aggregates scores based on these skills to generate the candidate's final report (e.g., "Candidate scored 95% in SQL, but 40% in React").

---

### C. `test_cases` (Specifically for Coding Tasks)
**1. `weight`?**
*   Not all test cases are equal. This allows for **Partial Scoring**.
*   *Example:* A candidate writes code that passes 2 basic test cases but fails the edge cases. Basic test cases might have a `weight` of 1 (10% of marks), while complex edge cases have a `weight` of 3 (30% of marks). The backend calculates the final score proportionally based on the weights of the passed cases.

**2. `is_performance` -> What and Why?**
*   **What it is:** A boolean flag (`TRUE`/`FALSE`) indicating this test case is designed to test the algorithm's efficiency, not just its correctness.
*   **Why it's needed:** A candidate might write a brute-force `O(N²)` solution. It will pass the basic test cases (`N=10`), but it will time-out and crash on a performance test case where `N=1,000,000`. Marking it as a performance test case allows the UI to specifically tell the candidate: *"Your logic is correct, but your code is too slow (Time Limit Exceeded)."*

---

### D. `task_options` -> What is it?
*This table stores the multiple choices for MCQ (Multiple Choice Questions).*

**1. `option_text`:**
*   The actual text of the choice (e.g., "O(N log N)", "Object Oriented", "42").

**2. `is_correct`:**
*   A boolean (`TRUE`/`FALSE`). The backend uses this to automatically grade the candidate's answer without human intervention. (You can have multiple options set to TRUE for "Select all that apply" questions).

**3. Why `display_order`?**
*   Normally, you want to randomize the order of MCQ options so candidates can't just tell their friends, "The answer to Q1 is A."
*   **BUT**, sometimes the order *must* be strictly preserved. For example:
    *   If an option is **"All of the above"**, it MUST be displayed last (Order 4).
    *   If the options are numbers (`10`, `20`, `30`, `40`), showing them in a randomized jumbled order (`40`, `10`, `30`, `20`) is bad UX.
*   `display_order` forces the UI to render them in a specific sequence when randomization is disabled for that specific question.


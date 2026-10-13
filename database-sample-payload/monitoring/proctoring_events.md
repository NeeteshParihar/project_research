# proctoring_events

Part of the **Monitoring** section. A log of every suspicious or notable action detected during a live session (browser events and AI proctoring). It records **participant** activity. Staff activity goes to `audit_logs`.

## Attributes

| Attribute | Type | Constraints | Description |
|-----------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY, NOT NULL, default `gen_random_uuid()` | Unique identifier of the event |
| `session_id` | UUID | NOT NULL, FOREIGN KEY → `assessment_sessions.id`, ON DELETE CASCADE | The session in which the event happened |
| `event_type` | VARCHAR(30) | NOT NULL, CHECK in the values below | What kind of event it was |
| `occurred_at` | TIMESTAMPTZ | NOT NULL | When it happened, by the server clock |
| `details` | JSONB | NULL | Extra data specific to the event type |

Suggested index: `(session_id, occurred_at)`. The table grows quickly, so plan for partitioning or archiving by date. Rows are append-only.

## Possible values of `event_type`

| Value | Meaning |
|-------|---------|
| `TAB_SWITCH` | The participant left the test tab. |
| `EXIT_FULLSCREEN` | The participant left fullscreen mode. |
| `WINDOW_RESIZE` | The browser window was resized (a possible split-screen). |
| `COPY_PASTE` | Text was copied or pasted in the test. |
| `FACE_NOT_DETECTED` | The webcam could not find a face. |
| `MULTIPLE_FACES` | The webcam saw more than one face. |
| `AUDIO_ANOMALY` | The microphone picked up voices or loud sounds. |

## `details` JSONB: shape by event type

| Event type | Keys |
|------------|------|
| `TAB_SWITCH` | `duration_away_seconds` |
| `EXIT_FULLSCREEN` | `duration_seconds` |
| `WINDOW_RESIZE` | `width`, `height` |
| `COPY_PASTE` | `action` (`COPY` or `PASTE`), `character_count`, `task_id` |
| `FACE_NOT_DETECTED` | `duration_seconds`, `snapshot_url` |
| `MULTIPLE_FACES` | `face_count`, `confidence`, `snapshot_url` |
| `AUDIO_ANOMALY` | `decibel_level`, `duration_seconds` |

Pasted text itself is not stored, only its length, to avoid keeping candidate data and code in the log.

## How it is used

1. **Live limits:** the server counts events against `assessments.settings` (for example `max_tab_switches`). Crossing the limit can end the session with `submit_type = PROCTORING_VIOLATION`.
2. **Summary:** when the session ends, the counts are rolled up into `results.integrity_summary` and `results.integrity_flagged`.
3. **Review:** evaluators and HR read the timeline and snapshots to decide on a flagged result.

## Sample payload

```json
{
  "id": "c1a2b3c4-d5e6-4f78-8a90-b1c2d3e4f5a1",
  "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
  "event_type": "TAB_SWITCH",
  "occurred_at": "2026-10-21T04:52:18Z",
  "details": { "duration_away_seconds": 9 }
}
```

## Sample rows

```json
[
  {
    "id": "c1a2b3c4-d5e6-4f78-8a90-b1c2d3e4f5a1",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "event_type": "TAB_SWITCH",
    "occurred_at": "2026-10-21T04:52:18Z",
    "details": { "duration_away_seconds": 9 }
  },
  {
    "id": "c1a2b3c4-d5e6-4f78-8a90-b1c2d3e4f5a2",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "event_type": "COPY_PASTE",
    "occurred_at": "2026-10-21T05:03:41Z",
    "details": { "action": "PASTE", "character_count": 84, "task_id": "6c7d8e9f-0a12-4b34-9c56-7d8e9f0a1b2c" }
  },
  {
    "id": "c1a2b3c4-d5e6-4f78-8a90-b1c2d3e4f5a3",
    "session_id": "50e5f607-1829-4a05-8ac6-d7f8091a2b3c",
    "event_type": "TAB_SWITCH",
    "occurred_at": "2026-10-21T05:15:07Z",
    "details": { "duration_away_seconds": 21 }
  },
  {
    "id": "c1a2b3c4-d5e6-4f78-8a90-b1c2d3e4f5a4",
    "session_id": "51f60718-293a-4b16-8db7-e8091a2b3c4d",
    "event_type": "MULTIPLE_FACES",
    "occurred_at": "2026-10-21T05:21:30Z",
    "details": { "face_count": 2, "confidence": 0.97, "snapshot_url": "s3://assessment-proctoring/2026/10/51f60718/evt-4.jpg" }
  }
]
```

## Relationships

- Many events belong to one session (`session_id` → `assessment_sessions.id`).
- Counts from this table feed `results.integrity_summary`.

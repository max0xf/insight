# Table: `task`

## Overview

**Purpose**: Store individual task/sub-task work items with lifecycle timestamps and authorship. Tasks represent the smallest actionable units of work, typically implementing part of a user story.

**Data Sources**:
- YouTrack: `source = "youtrack"`
- Jira: `source = "jira"`

---

## Schema Definition

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | bigint | PRIMARY KEY, AUTO INCREMENT | Internal primary key |
| `ingestion_date` | timestamp | NOT NULL | When this record was ingested |
| `source` | text | NOT NULL | Source tracker: "youtrack" or "jira" |
| `source_id` | text | NOT NULL | Identifier in source tracker system |
| `task_id` | text | NOT NULL | Human-readable task ID (e.g., "MON-305") |
| `user_story_id` | text | NULLABLE | Parent user story ID (e.g., "MON-201") |
| `deleted` | boolean | NOT NULL, DEFAULT false | Soft delete flag |
| `summary` | text | NOT NULL | Task title/summary |
| `project` | text | NULLABLE | Project key |
| `priority` | text | NULLABLE | Priority level |
| `estimation` | numeric | NULLABLE | Time estimation (hours) |
| `sprint` | text | NULLABLE | Current sprint name |
| `assignee_name` | text | NULLABLE | Current assignee name |
| `assignee_id` | text | NULLABLE | Current assignee ID in tracker |
| `labels` | text[] | NULLABLE | Labels/tags |
| `created_at` | timestamp | NOT NULL | Task creation datetime |
| `created_by_name` | text | NOT NULL | Creator name |
| `created_by_id` | text | NOT NULL | Creator ID in tracker |
| `started_at` | timestamp | NULLABLE | When work on the task began |
| `started_by_name` | text | NULLABLE | Who started the task |
| `started_by_id` | text | NULLABLE | Starter ID in tracker |
| `to_verify_at` | timestamp | NULLABLE | When task was moved to verification |
| `to_verify_by_name` | text | NULLABLE | Who moved it to verification |
| `to_verify_by_id` | text | NULLABLE | Verifier-mover ID in tracker |
| `done_at` | timestamp | NULLABLE | Task completion datetime |
| `done_by_name` | text | NULLABLE | Who completed the task |
| `done_by_id` | text | NULLABLE | Completer ID in tracker |
| `metadata` | jsonb | NULLABLE | Source-specific additional data |

**Indexes**:
- `idx_task_ingestion_date`: `(ingestion_date)` — incremental sync
- `idx_task_task_id`: `(task_id)`
- `idx_task_user_story_id`: `(user_story_id)` — parent story lookup
- `idx_task_project`: `(project)`
- `idx_task_sprint`: `(sprint)`
- `idx_task_assignee_id`: `(assignee_id)` — workload queries
- `idx_task_created_at`: `(created_at)`
- `idx_task_done_at`: `(done_at)`
- `(source, source_id)` — UNIQUE, prevents duplicates

---

## Field Semantics

### Core Identifiers

**`id`** (bigint, PRIMARY KEY)
- **Purpose**: Internal auto-increment key

**`source`** (text, NOT NULL)
- **Purpose**: Identifies the source tracker system
- **Values**: "youtrack", "jira"

**`source_id`** (text, NOT NULL)
- **Purpose**: Identifier in the source tracker system
- **Usage**: Trace back to source, deduplication

**`task_id`** (text, NOT NULL)
- **Purpose**: Human-readable task identifier
- **Format**: "PROJECT-NUMBER" (e.g., "MON-305")
- **Usage**: Display, cross-referencing with commit messages

**`user_story_id`** (text, NULLABLE)
- **Purpose**: Human-readable ID of the parent user story
- **Format**: "PROJECT-NUMBER" (e.g., "MON-201")
- **Note**: NULL if task is standalone (not linked to a story)
- **Usage**: Story→task hierarchy, rollup metrics

### Task Details

**`summary`** (text, NOT NULL)
- **Purpose**: Task title/summary
- **Examples**: "Fix login timeout bug", "Add email validation"

**`project`** (text, NULLABLE)
- **Purpose**: Project key
- **Examples**: "MON", "PLAT"

**`priority`** (text, NULLABLE)
- **Purpose**: Priority level
- **Examples**: "Critical", "Major", "Normal", "Minor"

**`estimation`** (numeric, NULLABLE)
- **Purpose**: Estimated effort in hours
- **Examples**: 2, 4, 8, 16
- **Usage**: Estimation accuracy analysis, capacity planning

**`sprint`** (text, NULLABLE)
- **Purpose**: Current or last sprint
- **Examples**: "Sprint 24.1", "2026-W08"

**`assignee_name`** / **`assignee_id`** (text, NULLABLE)
- **Purpose**: Current assignee
- **Usage**: Workload analysis, developer metrics

**`labels`** (text[], NULLABLE)
- **Purpose**: Labels/tags

### Lifecycle — Creation

**`created_at`** (timestamp, NOT NULL)
- **Purpose**: When the task was created

**`created_by_name`** (text, NOT NULL)
- **Purpose**: Creator name

**`created_by_id`** (text, NOT NULL)
- **Purpose**: Creator ID in tracker

### Lifecycle — Started (In Progress)

**`started_at`** (timestamp, NULLABLE)
- **Purpose**: When work on the task began (moved to "In Progress")
- **Note**: NULL if task hasn't been started
- **Usage**: Lead time (created → started), reaction time

**`started_by_name`** (text, NULLABLE)
- **Purpose**: Who started working on the task

**`started_by_id`** (text, NULLABLE)
- **Purpose**: Starter ID in tracker

### Lifecycle — To Verify

**`to_verify_at`** (timestamp, NULLABLE)
- **Purpose**: When the task was moved to verification/review state
- **Note**: NULL if task hasn't reached verification
- **Usage**: Development time (started → to_verify), review queue time

**`to_verify_by_name`** (text, NULLABLE)
- **Purpose**: Who moved the task to verification

**`to_verify_by_id`** (text, NULLABLE)
- **Purpose**: ID in tracker of the person who moved it to verification

### Lifecycle — Done

**`done_at`** (timestamp, NULLABLE)
- **Purpose**: When the task was completed
- **Note**: NULL if task is not done
- **Usage**: Verification time (to_verify → done), total cycle time (started → done)

**`done_by_name`** (text, NULLABLE)
- **Purpose**: Who completed/verified the task

**`done_by_id`** (text, NULLABLE)
- **Purpose**: Completer ID in tracker

### System Fields

**`ingestion_date`** (timestamp, NOT NULL)
- **Purpose**: When the record was ingested
- **Usage**: Incremental sync

**`deleted`** (boolean, NOT NULL, DEFAULT false)
- **Purpose**: Soft delete flag

**`metadata`** (jsonb, NULLABLE)
- **Purpose**: Source-specific fields
- **Examples**: `{"resolution": "Fixed", "branch": "feature/MON-305", "spent_time": "4h"}`

---

## Relationships

### Parent

**`user_story`**
- **Join**: `user_story_id` ← `task_id`
- **Cardinality**: Many tasks to one user story
- **Description**: Parent story this task implements

### Cross-Stream

- **Git commits**: `task_id` matches `raw_git.git_commit.task_id` — directly links tasks to code changes
- **Communication**: Correlate task activity with `stream_communication` events via assignee email

---

## Usage Examples

### Task cycle time breakdown

```sql
SELECT
    task_id,
    summary,
    assignee_name,
    EXTRACT(EPOCH FROM (started_at - created_at)) / 3600 as lead_time_hours,
    EXTRACT(EPOCH FROM (to_verify_at - started_at)) / 3600 as dev_time_hours,
    EXTRACT(EPOCH FROM (done_at - to_verify_at)) / 3600 as verify_time_hours,
    EXTRACT(EPOCH FROM (done_at - created_at)) / 3600 as total_time_hours
FROM task
WHERE done_at IS NOT NULL
  AND deleted = false
  AND done_at >= '2026-01-01'
ORDER BY total_time_hours DESC;
```

### Developer throughput

```sql
SELECT
    assignee_name,
    COUNT(*) as tasks_completed,
    AVG(EXTRACT(EPOCH FROM (done_at - started_at)) / 3600) as avg_cycle_hours,
    SUM(estimation) as total_estimated_hours
FROM task
WHERE done_at IS NOT NULL
  AND started_at IS NOT NULL
  AND deleted = false
  AND done_at >= '2026-01-01'
GROUP BY assignee_name
ORDER BY tasks_completed DESC;
```

### Tasks stuck in verification

```sql
SELECT
    task_id,
    summary,
    assignee_name,
    to_verify_at,
    NOW() - to_verify_at as waiting_time
FROM task
WHERE to_verify_at IS NOT NULL
  AND done_at IS NULL
  AND deleted = false
ORDER BY to_verify_at;
```

### Tasks linked to commits

```sql
SELECT
    t.task_id,
    t.summary,
    t.assignee_name,
    COUNT(c.id) as commit_count,
    SUM(ns.added) as lines_added,
    SUM(ns.removed) as lines_removed
FROM task t
JOIN raw_git.git_commit c ON t.task_id = c.task_id
LEFT JOIN raw_git.git_num_stat ns ON c.id = ns.commit_id
WHERE t.deleted = false
  AND t.done_at >= '2026-01-01'
GROUP BY t.task_id, t.summary, t.assignee_name
ORDER BY commit_count DESC
LIMIT 20;
```

### Estimation accuracy

```sql
SELECT
    assignee_name,
    COUNT(*) as tasks,
    AVG(estimation) as avg_estimated_hours,
    AVG(EXTRACT(EPOCH FROM (done_at - started_at)) / 3600) as avg_actual_hours,
    ROUND(AVG(EXTRACT(EPOCH FROM (done_at - started_at)) / 3600) /
          NULLIF(AVG(estimation), 0), 2) as actual_to_estimated_ratio
FROM task
WHERE done_at IS NOT NULL
  AND started_at IS NOT NULL
  AND estimation IS NOT NULL
  AND estimation > 0
  AND deleted = false
GROUP BY assignee_name
HAVING COUNT(*) >= 5
ORDER BY actual_to_estimated_ratio DESC;
```

### Sprint completion report

```sql
SELECT
    sprint,
    COUNT(*) as total_tasks,
    COUNT(*) FILTER (WHERE done_at IS NOT NULL) as completed,
    COUNT(*) FILTER (WHERE done_at IS NULL AND started_at IS NOT NULL) as in_progress,
    COUNT(*) FILTER (WHERE started_at IS NULL) as not_started,
    ROUND(COUNT(*) FILTER (WHERE done_at IS NOT NULL)::numeric / COUNT(*) * 100, 1) as completion_pct
FROM task
WHERE sprint IS NOT NULL
  AND deleted = false
GROUP BY sprint
ORDER BY sprint DESC;
```

---

## Notes and Considerations

### Lifecycle Flow

The task lifecycle is: **created → started → to_verify → done**

| Transition | Metric | Description |
|---|---|---|
| created → started | Lead time | Time from creation to first work |
| started → to_verify | Development time | Active development/coding time |
| to_verify → done | Verification time | Code review/QA/acceptance time |
| created → done | Total cycle time | End-to-end time |

### Skipped States

Not all tasks pass through every state:
- Simple tasks may skip verification (started → done directly)
- Tasks may be created already in progress
- Handle NULLs in duration calculations

### Commit Linkage

The `task_id` field enables direct linking to git commits via `raw_git.git_commit.task_id`. This provides code-level traceability: which commits implement which tasks.

### User Story Linkage

The `user_story_id` field references the human-readable story ID (e.g., "MON-201"), not an internal key. Join with `user_story.task_id` for hierarchy queries. Standalone tasks (not part of a story) have `user_story_id = NULL`.

### Soft Deletes

Always filter with `deleted = false`. Deleted tasks are preserved for audit trail and historical analysis.

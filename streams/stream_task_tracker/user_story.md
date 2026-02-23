# Table: `user_story`

## Overview

**Purpose**: Store user story work items with lifecycle timestamps and authorship. User stories represent units of functionality from the user perspective, progressing through states from creation to delivery.

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
| `task_id` | text | NOT NULL | Human-readable story ID (e.g., "MON-201") |
| `epic_id` | text | NULLABLE | Parent epic ID (e.g., "MON-100") |
| `deleted` | boolean | NOT NULL, DEFAULT false | Soft delete flag |
| `summary` | text | NOT NULL | User story title/summary |
| `project` | text | NULLABLE | Project key |
| `priority` | text | NULLABLE | Priority level |
| `story_points` | numeric | NULLABLE | Story point estimate |
| `sprint` | text | NULLABLE | Current sprint name |
| `assignee_name` | text | NULLABLE | Current assignee name |
| `assignee_id` | text | NULLABLE | Current assignee ID in tracker |
| `labels` | text[] | NULLABLE | Labels/tags |
| `created_at` | timestamp | NOT NULL | Story creation datetime |
| `created_by_name` | text | NOT NULL | Creator name |
| `created_by_id` | text | NOT NULL | Creator ID in tracker |
| `ready_for_dev_at` | timestamp | NULLABLE | When story became ready for development |
| `ready_for_dev_by_name` | text | NULLABLE | Who marked it ready for dev |
| `ready_for_dev_by_id` | text | NULLABLE | Ready-for-dev author ID in tracker |
| `sprint_started_at` | timestamp | NULLABLE | When story entered its first sprint |
| `sprint_started_by_name` | text | NULLABLE | Who added it to sprint |
| `sprint_started_by_id` | text | NULLABLE | Sprint-add author ID in tracker |
| `done_at` | timestamp | NULLABLE | Story completion datetime |
| `done_by_name` | text | NULLABLE | Who completed the story |
| `done_by_id` | text | NULLABLE | Completer ID in tracker |
| `metadata` | jsonb | NULLABLE | Source-specific additional data |

**Indexes**:
- `idx_user_story_ingestion_date`: `(ingestion_date)` — incremental sync
- `idx_user_story_task_id`: `(task_id)`
- `idx_user_story_epic_id`: `(epic_id)` — parent epic lookup
- `idx_user_story_project`: `(project)`
- `idx_user_story_sprint`: `(sprint)`
- `idx_user_story_created_at`: `(created_at)`
- `idx_user_story_done_at`: `(done_at)`
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
- **Purpose**: Human-readable story identifier
- **Format**: "PROJECT-NUMBER" (e.g., "MON-201")
- **Usage**: Display, cross-referencing, linking to child tasks

**`epic_id`** (text, NULLABLE)
- **Purpose**: Human-readable ID of the parent epic
- **Format**: "PROJECT-NUMBER" (e.g., "MON-100")
- **Note**: NULL if story is not linked to an epic
- **Usage**: Epic→story hierarchy, rollup metrics

### Story Details

**`summary`** (text, NOT NULL)
- **Purpose**: User story title/summary
- **Examples**: "As a user, I can reset my password", "Add export to CSV"

**`project`** (text, NULLABLE)
- **Purpose**: Project key
- **Examples**: "MON", "PLAT"

**`priority`** (text, NULLABLE)
- **Purpose**: Priority level
- **Examples**: "Critical", "Major", "Normal", "Minor"

**`story_points`** (numeric, NULLABLE)
- **Purpose**: Story point estimate for effort sizing
- **Examples**: 1, 2, 3, 5, 8, 13
- **Usage**: Velocity calculation, sprint capacity

**`sprint`** (text, NULLABLE)
- **Purpose**: Current or last sprint the story belongs to
- **Examples**: "Sprint 24.1", "2026-W08"
- **Usage**: Sprint-level reporting

**`assignee_name`** / **`assignee_id`** (text, NULLABLE)
- **Purpose**: Current assignee
- **Usage**: Workload analysis

**`labels`** (text[], NULLABLE)
- **Purpose**: Labels/tags

### Lifecycle — Creation

**`created_at`** (timestamp, NOT NULL)
- **Purpose**: When the story was created

**`created_by_name`** (text, NOT NULL)
- **Purpose**: Creator name

**`created_by_id`** (text, NOT NULL)
- **Purpose**: Creator ID in tracker

### Lifecycle — Ready for Development

**`ready_for_dev_at`** (timestamp, NULLABLE)
- **Purpose**: When the story was groomed and marked ready for development
- **Note**: NULL if story hasn't been triaged/groomed yet
- **Usage**: Grooming lead time (created → ready_for_dev)

**`ready_for_dev_by_name`** (text, NULLABLE)
- **Purpose**: Who marked the story as ready for development

**`ready_for_dev_by_id`** (text, NULLABLE)
- **Purpose**: ID in tracker of the person who marked it ready

### Lifecycle — Sprint Start

**`sprint_started_at`** (timestamp, NULLABLE)
- **Purpose**: When the story entered its first sprint (committed to a sprint)
- **Note**: NULL if story hasn't been planned into a sprint
- **Usage**: Backlog time (ready_for_dev → sprint_started), planning metrics

**`sprint_started_by_name`** (text, NULLABLE)
- **Purpose**: Who added the story to the sprint

**`sprint_started_by_id`** (text, NULLABLE)
- **Purpose**: ID in tracker of the person who added it to sprint

### Lifecycle — Done

**`done_at`** (timestamp, NULLABLE)
- **Purpose**: When the story was completed/accepted
- **Note**: NULL if story is not yet done
- **Usage**: Cycle time (sprint_started → done), total lead time (created → done)

**`done_by_name`** (text, NULLABLE)
- **Purpose**: Who completed/accepted the story

**`done_by_id`** (text, NULLABLE)
- **Purpose**: ID in tracker of the completer

### System Fields

**`ingestion_date`** (timestamp, NOT NULL)
- **Purpose**: When the record was ingested
- **Usage**: Incremental sync

**`deleted`** (boolean, NOT NULL, DEFAULT false)
- **Purpose**: Soft delete flag

**`metadata`** (jsonb, NULLABLE)
- **Purpose**: Source-specific fields
- **Examples**: `{"resolution": "Done", "fixVersions": ["1.5"], "acceptance_criteria": "..."}`

---

## Relationships

### Parent

**`epic`**
- **Join**: `epic_id` ← `task_id`
- **Cardinality**: Many stories to one epic
- **Description**: Parent epic for this story

### Parent Of

**`task`**
- **Join**: `task_id` → `user_story_id`
- **Cardinality**: One story to many tasks
- **Description**: Tasks implementing this story

### Cross-Stream

- **Git commits**: `task_id` matches `raw_git.git_commit.task_id`
- **Communication**: Correlate story timeline with `stream_communication` events

---

## Usage Examples

### Story cycle time (sprint start to done)

```sql
SELECT
    task_id,
    summary,
    sprint,
    sprint_started_at,
    done_at,
    EXTRACT(EPOCH FROM (done_at - sprint_started_at)) / 86400 as cycle_time_days
FROM user_story
WHERE done_at IS NOT NULL
  AND sprint_started_at IS NOT NULL
  AND deleted = false
  AND done_at >= '2026-01-01'
ORDER BY cycle_time_days DESC;
```

### Total lead time breakdown

```sql
SELECT
    task_id,
    summary,
    EXTRACT(EPOCH FROM (COALESCE(ready_for_dev_at, created_at) - created_at)) / 86400 as grooming_days,
    EXTRACT(EPOCH FROM (COALESCE(sprint_started_at, ready_for_dev_at) - COALESCE(ready_for_dev_at, created_at))) / 86400 as backlog_days,
    EXTRACT(EPOCH FROM (done_at - COALESCE(sprint_started_at, created_at))) / 86400 as dev_days,
    EXTRACT(EPOCH FROM (done_at - created_at)) / 86400 as total_lead_time_days
FROM user_story
WHERE done_at IS NOT NULL
  AND deleted = false
  AND done_at >= '2026-01-01'
ORDER BY total_lead_time_days DESC;
```

### Sprint velocity

```sql
SELECT
    sprint,
    COUNT(*) as stories_completed,
    SUM(story_points) as points_delivered
FROM user_story
WHERE done_at IS NOT NULL
  AND sprint IS NOT NULL
  AND deleted = false
GROUP BY sprint
ORDER BY sprint DESC;
```

### Stories by epic with completion status

```sql
SELECT
    epic_id,
    COUNT(*) as total_stories,
    COUNT(*) FILTER (WHERE done_at IS NOT NULL) as done,
    COUNT(*) FILTER (WHERE done_at IS NULL AND sprint_started_at IS NOT NULL) as in_progress,
    COUNT(*) FILTER (WHERE sprint_started_at IS NULL) as backlog,
    ROUND(SUM(CASE WHEN done_at IS NOT NULL THEN story_points ELSE 0 END) /
          NULLIF(SUM(story_points), 0) * 100, 1) as completion_pct
FROM user_story
WHERE epic_id IS NOT NULL
  AND deleted = false
GROUP BY epic_id
ORDER BY epic_id;
```

### Stories without commits

```sql
SELECT
    us.task_id,
    us.summary,
    us.sprint,
    us.done_at
FROM user_story us
LEFT JOIN raw_git.git_commit c ON us.task_id = c.task_id
WHERE c.id IS NULL
  AND us.sprint_started_at IS NOT NULL
  AND us.deleted = false
ORDER BY us.sprint_started_at DESC;
```

---

## Notes and Considerations

### Lifecycle Flow

The user story lifecycle is: **created → ready_for_dev → sprint_started → done**

| Transition | Metric | Description |
|---|---|---|
| created → ready_for_dev | Grooming time | Time to triage and refine the story |
| ready_for_dev → sprint_started | Backlog time | Time waiting in backlog before being planned |
| sprint_started → done | Cycle time | Active development time within sprint(s) |
| created → done | Total lead time | End-to-end time from idea to delivery |

### Skipped States

Not all stories go through every state:
- A story may go directly from created to sprint (skipping ready_for_dev)
- A story may be created already in a sprint
- Handle NULLs appropriately in duration calculations

### Story Points

Story points are optional and may not be used consistently across teams. Use `COUNT(*)` for throughput metrics when story points are unreliable.

### Epic Linkage

The `epic_id` field references the human-readable epic ID (e.g., "MON-100"), not an internal primary key. Join with `epic.task_id` for hierarchy queries.

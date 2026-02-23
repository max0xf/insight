# Table: `epic`

## Overview

**Purpose**: Store epic-level work items with lifecycle timestamps and authorship. Epics represent large bodies of work that can be broken down into user stories and tasks.

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
| `task_id` | text | NOT NULL | Human-readable epic ID (e.g., "MON-100") |
| `deleted` | boolean | NOT NULL, DEFAULT false | Soft delete flag |
| `summary` | text | NOT NULL | Epic title/summary |
| `project` | text | NULLABLE | Project key (e.g., "MON", "PLAT") |
| `priority` | text | NULLABLE | Priority level |
| `assignee_name` | text | NULLABLE | Current assignee name |
| `assignee_id` | text | NULLABLE | Current assignee ID in tracker |
| `labels` | text[] | NULLABLE | Labels/tags |
| `created_at` | timestamp | NOT NULL | Epic creation datetime |
| `created_by_name` | text | NOT NULL | Creator name |
| `created_by_id` | text | NOT NULL | Creator ID in tracker |
| `closed_at` | timestamp | NULLABLE | Epic closure datetime |
| `closed_by_name` | text | NULLABLE | Who closed the epic |
| `closed_by_id` | text | NULLABLE | Closer ID in tracker |
| `metadata` | jsonb | NULLABLE | Source-specific additional data |

**Indexes**:
- `idx_epic_ingestion_date`: `(ingestion_date)` — incremental sync
- `idx_epic_task_id`: `(task_id)`
- `idx_epic_project`: `(project)`
- `idx_epic_created_at`: `(created_at)`
- `idx_epic_closed_at`: `(closed_at)`
- `(source, source_id)` — UNIQUE, prevents duplicates

---

## Field Semantics

### Core Identifiers

**`id`** (bigint, PRIMARY KEY)
- **Purpose**: Internal auto-increment key

**`source`** (text, NOT NULL)
- **Purpose**: Identifies the source tracker system
- **Values**: "youtrack", "jira"
- **Usage**: Multi-source filtering

**`source_id`** (text, NOT NULL)
- **Purpose**: Identifier in the source tracker system
- **Format**: Source-specific (YouTrack internal ID or Jira issue key)
- **Usage**: Trace back to source, deduplication

**`task_id`** (text, NOT NULL)
- **Purpose**: Human-readable epic identifier
- **Format**: "PROJECT-NUMBER" (e.g., "MON-100")
- **Usage**: Display, cross-referencing, linking to child user stories

### Epic Details

**`summary`** (text, NOT NULL)
- **Purpose**: Epic title/summary text
- **Examples**: "User identity resolution system", "Q1 reporting dashboard"
- **Usage**: Display, search

**`project`** (text, NULLABLE)
- **Purpose**: Project key from the tracker
- **Examples**: "MON", "PLAT", "INFRA"
- **Usage**: Project-level filtering and grouping

**`priority`** (text, NULLABLE)
- **Purpose**: Priority level
- **Examples**: "Critical", "Major", "Normal", "Minor"
- **Usage**: Prioritization analysis

**`assignee_name`** / **`assignee_id`** (text, NULLABLE)
- **Purpose**: Current epic owner/assignee
- **Usage**: Ownership tracking, workload analysis

**`labels`** (text[], NULLABLE)
- **Purpose**: Labels or tags assigned to the epic
- **Format**: PostgreSQL text array
- **Usage**: Categorization, filtering

### Lifecycle — Creation

**`created_at`** (timestamp, NOT NULL)
- **Purpose**: When the epic was created
- **Usage**: Epic age, creation rate metrics

**`created_by_name`** (text, NOT NULL)
- **Purpose**: Name of the person who created the epic

**`created_by_id`** (text, NOT NULL)
- **Purpose**: ID of the creator in the source tracker

### Lifecycle — Closure

**`closed_at`** (timestamp, NULLABLE)
- **Purpose**: When the epic was closed/completed
- **Note**: NULL if epic is still open
- **Usage**: Epic duration (created → closed), throughput

**`closed_by_name`** (text, NULLABLE)
- **Purpose**: Name of the person who closed the epic

**`closed_by_id`** (text, NULLABLE)
- **Purpose**: ID of the closer in the source tracker

### System Fields

**`ingestion_date`** (timestamp, NOT NULL)
- **Purpose**: When the record was ingested into the stream
- **Usage**: Incremental sync

**`deleted`** (boolean, NOT NULL, DEFAULT false)
- **Purpose**: Soft delete flag
- **Usage**: Filter active records with `deleted = false`

**`metadata`** (jsonb, NULLABLE)
- **Purpose**: Source-specific fields not in the normalized schema
- **Examples**: `{"resolution": "Done", "fixVersions": ["1.5"], "components": ["backend"]}`

---

## Relationships

### Parent Of

**`user_story`**
- **Join**: `task_id` → `epic_id`
- **Cardinality**: One epic to many user stories
- **Description**: User stories that belong to this epic

### Cross-Stream

- **Git commits**: `task_id` matches `raw_git.git_commit.task_id`
- **Communication**: Correlate epic timeline with `stream_communication` events

---

## Usage Examples

### Open epics with age

```sql
SELECT
    task_id,
    summary,
    project,
    assignee_name,
    created_at,
    NOW() - created_at as age
FROM epic
WHERE closed_at IS NULL
  AND deleted = false
ORDER BY created_at;
```

### Epic duration (completed epics)

```sql
SELECT
    task_id,
    summary,
    created_at,
    closed_at,
    EXTRACT(EPOCH FROM (closed_at - created_at)) / 86400 as duration_days
FROM epic
WHERE closed_at IS NOT NULL
  AND deleted = false
  AND created_at >= '2026-01-01'
ORDER BY duration_days DESC;
```

### Epic throughput by month

```sql
SELECT
    DATE_TRUNC('month', closed_at) as month,
    COUNT(*) as epics_closed
FROM epic
WHERE closed_at IS NOT NULL
  AND deleted = false
GROUP BY month
ORDER BY month DESC;
```

### Epics with child story count

```sql
SELECT
    e.task_id,
    e.summary,
    COUNT(us.id) as story_count,
    COUNT(us.id) FILTER (WHERE us.done_at IS NOT NULL) as done_stories
FROM epic e
LEFT JOIN user_story us ON e.task_id = us.epic_id AND us.deleted = false
WHERE e.deleted = false
GROUP BY e.task_id, e.summary
ORDER BY story_count DESC;
```

---

## Notes and Considerations

### Epic vs User Story vs Task

Epics are the highest-level work items. They are broken down into user stories, which are further broken down into tasks. The hierarchy is: **Epic → User Story → Task**.

### Lifecycle Simplicity

Epics have a simple two-phase lifecycle: **created → closed**. Unlike tasks, they don't track intermediate states (started, testing) because epic progress is measured by the completion of their child stories.

### Soft Deletes

Always filter with `deleted = false` for active epic analysis. Deleted epics are kept for historical reference.

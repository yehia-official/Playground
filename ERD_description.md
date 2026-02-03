# Entity Relationship Diagram - Coding Practice Platform

## Core Entities & Relationships

### 1. Authentication & User Management

#### **users**
- Primary entity for all platform users
- Stores authentication credentials and profile information
- One-to-many with user_progress, challenge_attempts
- Many-to-one with roles

**Fields:**
- `id` (PK): UUID
- `email`: Unique, indexed
- `password_hash`: Encrypted password
- `role_id`: FK to roles
- `display_name`: User's display name
- `created_at`, `updated_at`: Timestamps
- `last_login_at`: Track activity
- `is_active`: Soft delete flag

#### **roles**
- Defines user permissions (user, admin, moderator)
- One-to-many with users

**Fields:**
- `id` (PK): Integer
- `name`: VARCHAR (e.g., 'user', 'admin')
- `description`: Role description

---

### 2. Content Hierarchy

#### **tracks**
- Top-level learning paths (HTML, CSS, JavaScript)
- One-to-many with modules
- Supports versioning via track_versions

**Fields:**
- `id` (PK): UUID
- `title`: Track name
- `description`: Markdown text
- `slug`: URL-friendly identifier (unique, indexed)
- `icon_url`: Display icon
- `display_order`: Sort order
- `is_published`: Publication status
- `created_by`: FK to users (admin)
- `created_at`, `updated_at`

#### **modules**
- Groups of related lessons within a track
- Many-to-one with tracks
- One-to-many with lessons

**Fields:**
- `id` (PK): UUID
- `track_id`: FK to tracks
- `title`: Module name
- `description`: Markdown
- `display_order`: Position in track
- `estimated_duration_minutes`: Total time estimate
- `is_published`: Status
- `created_at`, `updated_at`

#### **lessons**
- Individual learning units within modules
- Many-to-one with modules
- One-to-many with challenges

**Fields:**
- `id` (PK): UUID
- `module_id`: FK to modules
- `title`: Lesson name
- `description`: Markdown instructions
- `display_order`: Position in module
- `is_published`: Status
- `created_at`, `updated_at`

#### **challenges**
- Individual coding exercises
- Many-to-one with lessons
- One-to-many with challenge_attempts, user_progress

**Fields:**
- `id` (PK): UUID
- `lesson_id`: FK to lessons
- `title`: Challenge name
- `instructions`: Markdown (problem description)
- `starter_code_html`: Initial HTML
- `starter_code_css`: Initial CSS
- `starter_code_js`: Initial JavaScript
- `solution_code_html`: Reference solution HTML
- `solution_code_css`: Reference solution CSS
- `solution_code_js`: Reference solution JavaScript
- `tests`: JSONB (array of test cases)
- `difficulty`: ENUM ('beginner', 'intermediate', 'advanced')
- `estimated_time_minutes`: Expected completion time
- `points`: Score value
- `display_order`: Position in lesson
- `is_published`: Status
- `version`: Integer (for content versioning)
- `created_by`: FK to users (admin)
- `created_at`, `updated_at`, `published_at`

---

### 3. User Progress & Attempts

#### **user_progress**
- Tracks user completion status per challenge
- Composite unique constraint (user_id, challenge_id)
- Many-to-one with users, challenges

**Fields:**
- `id` (PK): UUID
- `user_id`: FK to users
- `challenge_id`: FK to challenges
- `status`: ENUM ('not_started', 'in_progress', 'completed')
- `best_score`: Percentage (0-100)
- `total_attempts`: Counter
- `completed_at`: Timestamp of first completion
- `last_attempt_at`: Most recent attempt
- `created_at`, `updated_at`

#### **challenge_attempts**
- Stores each submission attempt by users
- Many-to-one with users, challenges
- Retains full history for analytics

**Fields:**
- `id` (PK): UUID
- `user_id`: FK to users
- `challenge_id`: FK to challenges
- `user_code_html`: Submitted HTML
- `user_code_css`: Submitted CSS
- `user_code_js`: Submitted JavaScript
- `status`: ENUM ('pass', 'fail', 'error')
- `test_results`: JSONB (array of test outcomes)
- `runtime_logs`: TEXT (console output, errors)
- `execution_time_ms`: Performance metric
- `score`: Percentage (0-100)
- `created_at`: Submission timestamp

---

### 4. Enrollment & Tracking

#### **user_enrollments**
- Tracks which users are enrolled in which tracks
- Many-to-many relationship table
- Composite unique (user_id, track_id)

**Fields:**
- `id` (PK): UUID
- `user_id`: FK to users
- `track_id`: FK to tracks
- `enrolled_at`: Enrollment timestamp
- `progress_percentage`: Calculated field (0-100)
- `last_accessed_at`: Recent activity
- `completed_at`: Track completion timestamp

---

### 5. Content Versioning (Optional Advanced Feature)

#### **challenge_versions**
- Maintains version history for challenges
- Allows admins to publish/rollback content
- Many-to-one with challenges (via challenge_id reference)

**Fields:**
- `id` (PK): UUID
- `challenge_id`: FK to challenges
- `version`: Integer
- `title`, `instructions`, `starter_code_*`, `solution_code_*`, `tests`: Versioned content
- `published_by`: FK to users (admin)
- `published_at`: Publication timestamp
- `is_active`: Current version flag
- `created_at`

---

## Relationship Summary

```
roles (1) ──────< (M) users
users (1) ──────< (M) user_enrollments >──────── (1) tracks
users (1) ──────< (M) user_progress >──────────── (1) challenges
users (1) ──────< (M) challenge_attempts >─────── (1) challenges
users (1) ──────< (M) challenges (created_by)

tracks (1) ─────< (M) modules
modules (1) ────< (M) lessons
lessons (1) ────< (M) challenges

challenges (1) ─< (M) challenge_versions (optional)
```

---

## Key Design Decisions

### 1. **Normalized Structure**
- Content hierarchy is fully normalized (tracks → modules → lessons → challenges)
- Prevents duplication and ensures data integrity

### 2. **UUID Primary Keys**
- Used for all entities to support distributed systems and avoid ID conflicts
- Better for public-facing APIs

### 3. **JSONB for Flexible Data**
- `tests` and `test_results` stored as JSONB for flexible schema
- Allows complex test definitions without rigid structure

### 4. **Soft Deletes**
- `is_active`, `is_published` flags instead of hard deletes
- Preserves historical data and allows content restoration

### 5. **Attempt History**
- Complete retention of all attempts in `challenge_attempts`
- Enables analytics, progress visualization, and debugging

### 6. **Composite Uniqueness**
- `(user_id, challenge_id)` unique in user_progress
- `(user_id, track_id)` unique in user_enrollments
- Prevents duplicate records

### 7. **Versioning Strategy**
- `challenge_versions` table stores published versions
- Active challenges reference current version
- Admins can draft changes without affecting users

### 8. **Performance Optimization**
- Denormalized counters (`total_attempts`, `best_score`) for quick access
- Indexed foreign keys and frequently queried columns

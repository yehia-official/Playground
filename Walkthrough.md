# Database Design Walkthrough
## Coding Practice Platform

---

## Overview

I've created a **production-grade database schema** for your web-based coding practice platform. The design supports the full content hierarchy (Tracks → Modules → Lessons → Challenges), user progress tracking, multiple attempt submissions, and admin content management with versioning.

---

## Deliverables

### 1. Entity Relationship Diagram (ERD)

![ERD Visualization](C:/Users/LOQ/.gemini/antigravity/brain/cc21e3e9-78ab-42c5-92fc-c1c58ab13976/database_erd_diagram_1770122814861.png)

The ERD shows all 10 core tables with their relationships:

**Authentication Layer:**
- `roles` → `users` (1:M)

**Content Hierarchy:**
- `tracks` → `modules` → `lessons` → `challenges` (cascading 1:M)

**User Engagement:**
- `users` ↔ `tracks` via `user_enrollments` (M:M)
- `users` + `challenges` → `user_progress` (tracks completion status)
- `users` + `challenges` → `challenge_attempts` (stores all submissions)

**Versioning:**
- `challenges` → `challenge_versions` (1:M for content history)

---

## Key Features

### ✅ Normalized Structure
- **3rd Normal Form** compliance prevents data duplication
- Clear separation of concerns across 10 tables
- Referential integrity via foreign key constraints

### ✅ Comprehensive Progress Tracking
```sql
-- user_progress tracks:
- Completion status (not_started, in_progress, completed)
- Best score achieved
- Total attempts count
- First completion timestamp
- Last attempt timestamp
```

### ✅ Full Attempt History
Every code submission is preserved in `challenge_attempts`:
- User's HTML/CSS/JS code
- Pass/fail status
- Test results (JSONB)
- Runtime logs
- Execution time metrics

### ✅ Flexible Test System
Tests stored as JSONB arrays allow any test framework:
```json
[
  {
    "name": "Should have h1 tag",
    "test": "expect(doc.querySelector('h1')).toBeTruthy()",
    "points": 10
  }
]
```

### ✅ Content Versioning
Two-tier system for content management:
1. **Draft/Published**: Simple `is_published` flag (MVP)
2. **Version Control**: `challenge_versions` table (advanced)

### ✅ Performance Optimized
- **27 strategic indexes** for common queries
- **GIN indexes** on JSONB columns for fast searches
- **Denormalized counters** (`total_attempts`, `best_score`)
- **Composite indexes** for complex queries

---

## Schema Highlights

### Authentication & Authorization

```sql
CREATE TYPE user_role AS ENUM ('user', 'admin', 'moderator');

CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role_id INTEGER REFERENCES roles(id),
    display_name VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    -- ... timestamps
);
```

### Challenge Definition

```sql
CREATE TABLE challenges (
    id UUID PRIMARY KEY,
    lesson_id UUID REFERENCES lessons(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    instructions TEXT NOT NULL,
    
    -- Starter code (what users see initially)
    starter_code_html TEXT,
    starter_code_css TEXT,
    starter_code_js TEXT,
    
    -- Solution code (reference for admins)
    solution_code_html TEXT,
    solution_code_css TEXT,
    solution_code_js TEXT,
    
    -- Test cases as JSONB array
    tests JSONB DEFAULT '[]'::jsonb,
    
    difficulty challenge_difficulty,
    estimated_time_minutes INTEGER,
    points INTEGER DEFAULT 10,
    
    version INTEGER DEFAULT 1,
    is_published BOOLEAN DEFAULT false
);
```

### User Progress Tracking

```sql
CREATE TABLE user_progress (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    challenge_id UUID REFERENCES challenges(id),
    status progress_status DEFAULT 'not_started',
    best_score DECIMAL(5,2) DEFAULT 0.00,
    total_attempts INTEGER DEFAULT 0,
    completed_at TIMESTAMPTZ,
    last_attempt_at TIMESTAMPTZ,
    
    UNIQUE(user_id, challenge_id)  -- One record per user per challenge
);
```

### Attempt Storage

```sql
CREATE TABLE challenge_attempts (
    id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    challenge_id UUID REFERENCES challenges(id),
    
    -- Submitted code
    user_code_html TEXT,
    user_code_css TEXT,
    user_code_js TEXT,
    
    -- Results
    status attempt_status NOT NULL,  -- pass/fail/error
    test_results JSONB DEFAULT '[]'::jsonb,
    runtime_logs TEXT,
    execution_time_ms INTEGER,
    score DECIMAL(5,2),
    
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

---

## Versioning & Publishing Strategy

### Option 1: Simple (MVP)
Use `is_published` flag only:
- Admins create challenges in draft mode (`is_published = false`)
- Preview and test drafts
- Publish when ready (`is_published = true`)
- Users only see published content

> [!TIP]
> **Start with this approach** for faster development and easier testing.

### Option 2: Full Version Control (Production)
Use `challenge_versions` table:
- Every publish creates a new version record
- Users see the latest **active** version
- Admins can rollback to previous versions
- Complete audit trail

> [!IMPORTANT]
> Implement version control after MVP validation to avoid premature complexity.

**Migration Path:**
```sql
-- When ready to add versioning, populate historical versions
INSERT INTO challenge_versions (challenge_id, version, title, instructions, ...)
SELECT id, version, title, instructions, ...
FROM challenges
WHERE is_published = true;
```

---

## Performance Optimizations

### Critical Indexes

| Index | Purpose |
|-------|---------|
| `idx_users_email` | Fast login lookups |
| `idx_challenges_lesson_id` | Load challenges by lesson |
| `idx_progress_user_id` | User dashboard queries |
| `idx_attempts_user_challenge` | Attempt history pagination |
| `idx_challenges_tests (GIN)` | JSONB test searches |

### Denormalized Counters

To avoid expensive `COUNT(*)` queries:

```sql
-- Stored in user_progress
total_attempts INTEGER  -- Incremented on each attempt
best_score DECIMAL      -- Updated when beaten

-- Stored in user_enrollments
progress_percentage DECIMAL  -- Calculated periodically
```

**Update via triggers:**
```sql
CREATE TRIGGER increment_attempts
AFTER INSERT ON challenge_attempts
FOR EACH ROW EXECUTE FUNCTION increment_attempt_counter();
```

### Query Optimization Examples

**Bad (N+1 problem):**
```javascript
// Loads tracks, then queries modules separately for each track
const tracks = await db.query('SELECT * FROM tracks');
for (const track of tracks) {
  const modules = await db.query('SELECT * FROM modules WHERE track_id = $1', [track.id]);
}
```

**Good (Single JOIN):**
```sql
SELECT t.*, m.* 
FROM tracks t
LEFT JOIN modules m ON m.track_id = t.id
WHERE t.is_published = true;
```

---

## Common Use Cases

### 1. User Dashboard

Get all enrolled tracks with progress:

```sql
SELECT 
    t.title,
    ue.progress_percentage,
    COUNT(DISTINCT c.id) as total_challenges,
    COUNT(DISTINCT CASE WHEN up.status = 'completed' THEN c.id END) as completed
FROM user_enrollments ue
JOIN tracks t ON ue.track_id = t.id
JOIN modules m ON m.track_id = t.id
JOIN lessons l ON l.module_id = m.id
JOIN challenges c ON c.lesson_id = l.id
LEFT JOIN user_progress up ON up.challenge_id = c.id AND up.user_id = ue.user_id
WHERE ue.user_id = 'user-uuid'
GROUP BY t.id, t.title, ue.progress_percentage;
```

### 2. Load Challenge Page

```sql
SELECT 
    c.*,
    up.status,
    up.best_score,
    up.total_attempts,
    l.title as lesson_title
FROM challenges c
JOIN lessons l ON c.lesson_id = l.id
LEFT JOIN user_progress up ON up.challenge_id = c.id AND up.user_id = $1
WHERE c.id = $2 AND c.is_published = true;
```

### 3. Submit Attempt

```sql
-- Insert attempt
INSERT INTO challenge_attempts (
    user_id, challenge_id, 
    user_code_html, user_code_css, user_code_js,
    status, test_results, score
) VALUES ($1, $2, $3, $4, $5, $6, $7, $8);

-- Update user_progress (upsert)
INSERT INTO user_progress (user_id, challenge_id, status, best_score, total_attempts)
VALUES ($1, $2, 'in_progress', $3, 1)
ON CONFLICT (user_id, challenge_id) DO UPDATE SET
    best_score = GREATEST(user_progress.best_score, EXCLUDED.best_score),
    total_attempts = user_progress.total_attempts + 1,
    last_attempt_at = NOW();
```

### 4. Leaderboard

```sql
SELECT 
    u.display_name,
    up.best_score,
    up.completed_at,
    RANK() OVER (ORDER BY up.best_score DESC, up.completed_at ASC) as rank
FROM user_progress up
JOIN users u ON up.user_id = u.id
WHERE up.challenge_id = $1 AND up.status = 'completed'
ORDER BY rank
LIMIT 10;
```

---

## Security Considerations

> [!CAUTION]
> **Never store plain passwords!** Always use bcrypt or argon2:
> ```javascript
> const hash = await bcrypt.hash(password, 12);
> ```

> [!WARNING]
> **Prevent SQL injection** by using parameterized queries:
> ```javascript
> // ✅ Good
> db.query('SELECT * FROM users WHERE email = $1', [userEmail]);
> 
> // ❌ Bad
> db.query(`SELECT * FROM users WHERE email = '${userEmail}'`);
> ```

### Row-Level Security (Optional)

For multi-tenant isolation:
```sql
ALTER TABLE user_progress ENABLE ROW LEVEL SECURITY;

CREATE POLICY user_progress_isolation ON user_progress
    USING (user_id = current_setting('app.current_user_id')::uuid);
```

---

## Scalability Strategies

### Partitioning (for millions of attempts)

```sql
-- Partition by month
CREATE TABLE challenge_attempts (...)
PARTITION BY RANGE (created_at);

CREATE TABLE attempts_2026_01 PARTITION OF challenge_attempts
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

Benefits:
- Faster queries (smaller table scans)
- Easy archival (drop old partitions)
- Better vacuum performance

### Caching Strategy

**What to cache:**
- Published tracks/modules/lessons (TTL: 1 hour)
- Challenge content (invalidate on publish)
- User progress summaries (TTL: 5 minutes)

**What NOT to cache:**
- Challenge attempts (too granular)
- Real-time leaderboards (stale data issue)

### Read Replicas

For production:
- **Primary:** Write operations (attempts, progress)
- **Replica 1:** Read queries (dashboards, content)
- **Replica 2:** Analytics and reports

---

## Files Delivered

1. **[erd_description.md](file:///C:/Users/LOQ/.gemini/antigravity/brain/cc21e3e9-78ab-42c5-92fc-c1c58ab13976/erd_description.md)**
   - Complete description of all entities
   - Relationship mappings
   - Design decision rationale

2. **[schema.sql](file:///C:/Users/LOQ/.gemini/antigravity/brain/cc21e3e9-78ab-42c5-92fc-c1c58ab13976/schema.sql)**
   - Full PostgreSQL schema
   - All tables, enums, indexes
   - Triggers and constraints
   - Sample queries commented

3. **[database_documentation.md](file:///C:/Users/LOQ/.gemini/antigravity/brain/cc21e3e9-78ab-42c5-92fc-c1c58ab13976/database_documentation.md)**
   - Versioning strategies
   - Publishing workflows
   - Performance optimization guide
   - Security best practices
   - Common use cases

4. **ERD Diagram**
   - Visual representation of all entities
   - Relationship indicators (1:M, M:M)
   - Primary/foreign key markers

---

## Next Steps

### Immediate Actions

1. **Review the schema**
   - Verify it meets all requirements
   - Check for missing fields/relationships

2. **Set up PostgreSQL**
   ```bash
   # Create database
   createdb coding_practice_platform
   
   # Run schema
   psql coding_practice_platform < schema.sql
   ```

3. **Seed initial data**
   ```sql
   -- Create admin user
   INSERT INTO users (email, password_hash, role_id, display_name)
   VALUES ('admin@example.com', '$2b$12$...', 2, 'Admin');
   
   -- Create sample track
   INSERT INTO tracks (title, slug, description, is_published, created_by)
   VALUES ('HTML Basics', 'html-basics', 'Learn HTML from scratch', true, 'admin-uuid');
   ```

### Future Enhancements

- **Hints system**: Progressive hints for stuck users
- **Discussion forums**: Comments on challenges
- **Achievements**: Badges for milestones
- **Social features**: Follow users, share solutions
- **Analytics dashboard**: Track engagement metrics

---

## Summary

✅ **10 normalized tables** covering all requirements  
✅ **27 strategic indexes** for optimal performance  
✅ **JSONB flexibility** for tests and results  
✅ **Version control** support for content management  
✅ **Production-ready** with constraints, triggers, and security  
✅ **Scalable architecture** with partitioning and caching strategies  

The schema is ready for immediate implementation! 🚀

# PostgreSQL Database Schema

Complete PostgreSQL schema for the Coding Practice Platform.

---

## Overview

This schema supports:
- User authentication with role-based access control
- Content hierarchy: Tracks → Modules → Lessons → Challenges
- User progress tracking and attempt history
- Content versioning for admin workflow
- Performance-optimized with strategic indexes

---

## Database Setup

```sql
-- Coding Practice Platform - PostgreSQL Schema
-- Database: coding_practice_platform
-- PostgreSQL 14+

-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

---

## Enums

```sql
CREATE TYPE user_role AS ENUM ('user', 'admin', 'moderator');
CREATE TYPE challenge_difficulty AS ENUM ('beginner', 'intermediate', 'advanced');
CREATE TYPE progress_status AS ENUM ('not_started', 'in_progress', 'completed');
CREATE TYPE attempt_status AS ENUM ('pass', 'fail', 'error');
```

---

## Authentication & User Management

### Roles Table

```sql
CREATE TABLE roles (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role_id INTEGER NOT NULL DEFAULT 1 REFERENCES roles(id),
    display_name VARCHAR(100) NOT NULL,
    avatar_url VARCHAR(500),
    is_active BOOLEAN DEFAULT true,
    email_verified BOOLEAN DEFAULT false,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

---

## Content Hierarchy

### Tracks Table

```sql
CREATE TABLE tracks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    icon_url VARCHAR(500),
    display_order INTEGER NOT NULL DEFAULT 0,
    is_published BOOLEAN DEFAULT false,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### Modules Table

```sql
CREATE TABLE modules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    track_id UUID NOT NULL REFERENCES tracks(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    estimated_duration_minutes INTEGER,
    is_published BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### Lessons Table

```sql
CREATE TABLE lessons (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    module_id UUID NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    display_order INTEGER NOT NULL DEFAULT 0,
    is_published BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

### Challenges Table

```sql
CREATE TABLE challenges (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    lesson_id UUID NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,
    title VARCHAR(200) NOT NULL,
    instructions TEXT NOT NULL,
    
    -- Starter code
    starter_code_html TEXT DEFAULT '',
    starter_code_css TEXT DEFAULT '',
    starter_code_js TEXT DEFAULT '',
    
    -- Solution code (for admins/reference)
    solution_code_html TEXT DEFAULT '',
    solution_code_css TEXT DEFAULT '',
    solution_code_js TEXT DEFAULT '',
    
    -- Test cases stored as JSONB array
    -- Example: [{"name": "Should have h1 tag", "test": "expect(doc.querySelector('h1')).toBeTruthy()"}]
    tests JSONB DEFAULT '[]'::jsonb,
    
    difficulty challenge_difficulty DEFAULT 'beginner',
    estimated_time_minutes INTEGER,
    points INTEGER DEFAULT 10,
    display_order INTEGER NOT NULL DEFAULT 0,
    
    -- Version control
    version INTEGER DEFAULT 1,
    is_published BOOLEAN DEFAULT false,
    
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    published_at TIMESTAMPTZ
);
```

---

## User Progress & Attempts

### User Enrollments Table

```sql
CREATE TABLE user_enrollments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    track_id UUID NOT NULL REFERENCES tracks(id) ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    progress_percentage DECIMAL(5,2) DEFAULT 0.00 CHECK (progress_percentage >= 0 AND progress_percentage <= 100),
    last_accessed_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    
    UNIQUE(user_id, track_id)
);
```

### User Progress Table

```sql
CREATE TABLE user_progress (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
    status progress_status DEFAULT 'not_started',
    best_score DECIMAL(5,2) DEFAULT 0.00 CHECK (best_score >= 0 AND best_score <= 100),
    total_attempts INTEGER DEFAULT 0,
    completed_at TIMESTAMPTZ,
    last_attempt_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(user_id, challenge_id)
);
```

### Challenge Attempts Table

```sql
CREATE TABLE challenge_attempts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
    
    -- User's submitted code
    user_code_html TEXT DEFAULT '',
    user_code_css TEXT DEFAULT '',
    user_code_js TEXT DEFAULT '',
    
    -- Attempt results
    status attempt_status NOT NULL,
    
    -- Test results stored as JSONB array
    -- Example: [{"test_name": "Has h1", "passed": true, "message": "Found h1 element"}]
    test_results JSONB DEFAULT '[]'::jsonb,
    
    runtime_logs TEXT,
    execution_time_ms INTEGER,
    score DECIMAL(5,2) DEFAULT 0.00 CHECK (score >= 0 AND score <= 100),
    
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

---

## Content Versioning

### Challenge Versions Table

```sql
CREATE TABLE challenge_versions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    challenge_id UUID NOT NULL REFERENCES challenges(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    
    -- Versioned content
    title VARCHAR(200) NOT NULL,
    instructions TEXT NOT NULL,
    starter_code_html TEXT DEFAULT '',
    starter_code_css TEXT DEFAULT '',
    starter_code_js TEXT DEFAULT '',
    solution_code_html TEXT DEFAULT '',
    solution_code_css TEXT DEFAULT '',
    solution_code_js TEXT DEFAULT '',
    tests JSONB DEFAULT '[]'::jsonb,
    difficulty challenge_difficulty DEFAULT 'beginner',
    estimated_time_minutes INTEGER,
    points INTEGER DEFAULT 10,
    
    is_active BOOLEAN DEFAULT false,
    published_by UUID REFERENCES users(id),
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(challenge_id, version)
);
```

---

## Performance Indexes

### Users Indexes

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role_id ON users(role_id);
CREATE INDEX idx_users_is_active ON users(is_active);
```

### Tracks Indexes

```sql
CREATE INDEX idx_tracks_slug ON tracks(slug);
CREATE INDEX idx_tracks_is_published ON tracks(is_published);
CREATE INDEX idx_tracks_display_order ON tracks(display_order);
```

### Modules Indexes

```sql
CREATE INDEX idx_modules_track_id ON modules(track_id);
CREATE INDEX idx_modules_is_published ON modules(is_published);
CREATE INDEX idx_modules_display_order ON modules(display_order);
```

### Lessons Indexes

```sql
CREATE INDEX idx_lessons_module_id ON lessons(module_id);
CREATE INDEX idx_lessons_is_published ON lessons(is_published);
CREATE INDEX idx_lessons_display_order ON lessons(display_order);
```

### Challenges Indexes

```sql
CREATE INDEX idx_challenges_lesson_id ON challenges(lesson_id);
CREATE INDEX idx_challenges_difficulty ON challenges(difficulty);
CREATE INDEX idx_challenges_is_published ON challenges(is_published);
CREATE INDEX idx_challenges_display_order ON challenges(display_order);
CREATE INDEX idx_challenges_created_by ON challenges(created_by);
```

### User Enrollments Indexes

```sql
CREATE INDEX idx_enrollments_user_id ON user_enrollments(user_id);
CREATE INDEX idx_enrollments_track_id ON user_enrollments(track_id);
CREATE INDEX idx_enrollments_enrolled_at ON user_enrollments(enrolled_at);
```

### User Progress Indexes

```sql
CREATE INDEX idx_progress_user_id ON user_progress(user_id);
CREATE INDEX idx_progress_challenge_id ON user_progress(challenge_id);
CREATE INDEX idx_progress_status ON user_progress(status);
CREATE INDEX idx_progress_last_attempt ON user_progress(last_attempt_at);
```

### Challenge Attempts Indexes

```sql
CREATE INDEX idx_attempts_user_id ON challenge_attempts(user_id);
CREATE INDEX idx_attempts_challenge_id ON challenge_attempts(challenge_id);
CREATE INDEX idx_attempts_status ON challenge_attempts(status);
CREATE INDEX idx_attempts_created_at ON challenge_attempts(created_at DESC);
CREATE INDEX idx_attempts_user_challenge ON challenge_attempts(user_id, challenge_id, created_at DESC);
```

### Challenge Versions Indexes

```sql
CREATE INDEX idx_versions_challenge_id ON challenge_versions(challenge_id);
CREATE INDEX idx_versions_is_active ON challenge_versions(is_active);
```

### JSONB Indexes

```sql
-- GIN indexes for fast JSONB queries
CREATE INDEX idx_challenges_tests ON challenges USING GIN (tests);
CREATE INDEX idx_attempts_test_results ON challenge_attempts USING GIN (test_results);
```

---

## Triggers

### Auto-Update Timestamps

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_tracks_updated_at BEFORE UPDATE ON tracks
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_modules_updated_at BEFORE UPDATE ON modules
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_lessons_updated_at BEFORE UPDATE ON lessons
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_challenges_updated_at BEFORE UPDATE ON challenges
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_user_progress_updated_at BEFORE UPDATE ON user_progress
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

## Seed Data

```sql
INSERT INTO roles (name, description) VALUES
    ('user', 'Regular platform user'),
    ('admin', 'Platform administrator with full access'),
    ('moderator', 'Content moderator');
```

---

## Sample Queries

### Get User's Track Progress

```sql
SELECT 
    t.title,
    ue.progress_percentage,
    COUNT(DISTINCT c.id) as total_challenges,
    COUNT(DISTINCT CASE WHEN up.status = 'completed' THEN c.id END) as completed_challenges
FROM user_enrollments ue
JOIN tracks t ON ue.track_id = t.id
JOIN modules m ON m.track_id = t.id
JOIN lessons l ON l.module_id = m.id
JOIN challenges c ON c.lesson_id = l.id
LEFT JOIN user_progress up ON up.challenge_id = c.id AND up.user_id = ue.user_id
WHERE ue.user_id = 'user-uuid'
GROUP BY t.id, t.title, ue.progress_percentage;
```

### Get User's Recent Attempts

```sql
SELECT 
    c.title,
    ca.status,
    ca.score,
    ca.created_at
FROM challenge_attempts ca
JOIN challenges c ON ca.challenge_id = c.id
WHERE ca.user_id = 'user-uuid'
ORDER BY ca.created_at DESC
LIMIT 10;
```

### Get Challenge Leaderboard

```sql
SELECT 
    u.display_name,
    up.best_score,
    up.total_attempts,
    up.completed_at
FROM user_progress up
JOIN users u ON up.user_id = u.id
WHERE up.challenge_id = 'challenge-uuid' AND up.status = 'completed'
ORDER BY up.best_score DESC, up.completed_at ASC
LIMIT 10;
```

---

## Usage

To create the database:

```bash
# Create database
createdb coding_practice_platform

# Run this schema
psql coding_practice_platform < schema.sql
```

---

**Next Steps:** Review the [Database Documentation](./database_documentation.md) for detailed information about versioning strategies, performance optimization, and security best practices.

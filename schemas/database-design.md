# Database Schemas & Data Models

## Database Architecture

### Primary Database: PostgreSQL 15+
- **Purpose:** Transactional data, user management, job metadata
- **Deployment:** Multi-AZ with read replicas
- **Backup:** Point-in-time recovery with 30-day retention

### Cache Layer: Redis Cluster
- **Purpose:** Session management, rate limiting, job queues
- **Deployment:** 3-node cluster with persistence
- **Eviction:** LRU policy with TTL

### Time Series: InfluxDB
- **Purpose:** Metrics, monitoring, analytics
- **Retention:** 30 days detailed, 1 year aggregated

### Search: Elasticsearch
- **Purpose:** Generation search, user discovery
- **Deployment:** 3-node cluster with replicas

---

## PostgreSQL Schema

### 1. Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(50) UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255),
    avatar_url TEXT,
    
    -- Subscription & Billing
    tier VARCHAR(20) DEFAULT 'free' CHECK (tier IN ('free', 'standard', 'premium', 'enterprise')),
    credits INTEGER DEFAULT 5,
    subscription_id VARCHAR(255),
    subscription_status VARCHAR(20) DEFAULT 'inactive',
    subscription_starts_at TIMESTAMP,
    subscription_ends_at TIMESTAMP,
    
    -- Preferences
    preferred_language VARCHAR(5) DEFAULT 'en',
    timezone VARCHAR(50) DEFAULT 'UTC',
    email_notifications BOOLEAN DEFAULT true,
    
    -- Status & Metadata
    is_active BOOLEAN DEFAULT true,
    is_verified BOOLEAN DEFAULT false,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_tier ON users(tier);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_subscription_status ON users(subscription_status);
```

### 2. Generations Table
```sql
CREATE TABLE generations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Input Data
    input_image_url TEXT NOT NULL,
    input_image_metadata JSONB,
    original_prompt TEXT NOT NULL,
    enhanced_prompt TEXT,
    language_code VARCHAR(5) DEFAULT 'en',
    
    -- Generation Settings
    settings JSONB DEFAULT '{
        "duration": 10,
        "quality": "standard",
        "fps": 25,
        "aspect_ratio": "16:9",
        "model_version": "svd-v1.2"
    }'::jsonb,
    
    -- Processing Data
    status VARCHAR(20) DEFAULT 'pending' CHECK (
        status IN ('pending', 'queued', 'processing', 'completed', 'failed', 'cancelled')
    ),
    progress INTEGER DEFAULT 0 CHECK (progress >= 0 AND progress <= 100),
    queue_position INTEGER,
    assigned_gpu_node VARCHAR(100),
    assigned_gpu_id VARCHAR(20),
    
    -- Output Data
    output_video_url TEXT,
    output_thumbnail_url TEXT,
    output_metadata JSONB,
    
    -- Performance Metrics
    queue_wait_time_seconds INTEGER,
    inference_time_seconds REAL,
    total_processing_time_seconds REAL,
    
    -- Business Logic
    credits_cost INTEGER DEFAULT 5,
    is_public BOOLEAN DEFAULT false,
    is_featured BOOLEAN DEFAULT false,
    
    -- Error Handling
    error_code VARCHAR(50),
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    
    -- Timestamps
    queued_at TIMESTAMP,
    started_processing_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_generations_user_id ON generations(user_id);
CREATE INDEX idx_generations_status ON generations(status);
CREATE INDEX idx_generations_created_at ON generations(created_at DESC);
CREATE INDEX idx_generations_is_public ON generations(is_public) WHERE is_public = true;
CREATE INDEX idx_generations_processing ON generations(status, created_at) 
    WHERE status IN ('queued', 'processing');

-- Partial index for active jobs
CREATE INDEX idx_generations_active_jobs ON generations(created_at, status) 
    WHERE status IN ('pending', 'queued', 'processing');
```

### 3. Credit Transactions Table
```sql
CREATE TABLE credit_transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    generation_id UUID REFERENCES generations(id),
    
    -- Transaction Details
    amount INTEGER NOT NULL, -- Positive for credits, negative for debits
    transaction_type VARCHAR(20) NOT NULL CHECK (
        transaction_type IN ('purchase', 'subscription', 'generation', 'refund', 'bonus', 'penalty')
    ),
    description TEXT,
    
    -- Reference Data
    stripe_payment_intent_id VARCHAR(255),
    subscription_period_start DATE,
    subscription_period_end DATE,
    
    -- Balance Tracking
    balance_before INTEGER NOT NULL,
    balance_after INTEGER NOT NULL,
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_credit_transactions_user_id ON credit_transactions(user_id);
CREATE INDEX idx_credit_transactions_generation_id ON credit_transactions(generation_id);
CREATE INDEX idx_credit_transactions_created_at ON credit_transactions(created_at DESC);
CREATE INDEX idx_credit_transactions_type ON credit_transactions(transaction_type);
```

### 4. User Sessions Table
```sql
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Session Data
    session_token VARCHAR(255) NOT NULL UNIQUE,
    refresh_token VARCHAR(255) NOT NULL UNIQUE,
    
    -- Device & Location
    device_type VARCHAR(50), -- 'web', 'mobile_ios', 'mobile_android'
    user_agent TEXT,
    ip_address INET,
    country_code VARCHAR(2),
    city VARCHAR(100),
    
    -- Session Status
    is_active BOOLEAN DEFAULT true,
    last_activity_at TIMESTAMP DEFAULT NOW(),
    
    -- Security
    login_method VARCHAR(20) DEFAULT 'email', -- 'email', 'google', 'apple'
    is_suspicious BOOLEAN DEFAULT false,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP DEFAULT NOW() + INTERVAL '30 days'
);

-- Indexes
CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_token ON user_sessions(session_token);
CREATE INDEX idx_sessions_active ON user_sessions(user_id, is_active) WHERE is_active = true;
CREATE INDEX idx_sessions_expires ON user_sessions(expires_at);
```

### 5. Generation Analytics Table
```sql
CREATE TABLE generation_analytics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generation_id UUID NOT NULL REFERENCES generations(id) ON DELETE CASCADE,
    
    -- Engagement Metrics
    views_count INTEGER DEFAULT 0,
    likes_count INTEGER DEFAULT 0,
    shares_count INTEGER DEFAULT 0,
    downloads_count INTEGER DEFAULT 0,
    
    -- Performance Metrics
    avg_watch_time_seconds REAL,
    completion_rate REAL, -- Percentage who watched to end
    
    -- Content Metrics
    safety_score REAL,
    quality_score REAL,
    similarity_score REAL, -- How well output matches prompt
    
    -- System Metrics
    bandwidth_usage_bytes BIGINT DEFAULT 0,
    cdn_cache_hit_ratio REAL,
    
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_analytics_generation_id ON generation_analytics(generation_id);
CREATE INDEX idx_analytics_views ON generation_analytics(views_count DESC);
CREATE INDEX idx_analytics_likes ON generation_analytics(likes_count DESC);
```

### 6. GPU Nodes Table
```sql
CREATE TABLE gpu_nodes (
    id VARCHAR(100) PRIMARY KEY, -- Node name/ID from Kubernetes
    
    -- Hardware Specs
    gpu_type VARCHAR(20) NOT NULL CHECK (gpu_type IN ('A100', 'V100', 'T4', 'H100')),
    gpu_count INTEGER NOT NULL,
    gpu_memory_gb INTEGER NOT NULL,
    cpu_cores INTEGER,
    memory_gb INTEGER,
    storage_gb INTEGER,
    
    -- Network & Location
    region VARCHAR(20) NOT NULL,
    availability_zone VARCHAR(30),
    instance_type VARCHAR(50),
    network_bandwidth_gbps REAL,
    
    -- Status & Health
    status VARCHAR(20) DEFAULT 'healthy' CHECK (
        status IN ('healthy', 'degraded', 'unhealthy', 'maintenance')
    ),
    is_schedulable BOOLEAN DEFAULT true,
    
    -- Pricing
    cost_per_hour DECIMAL(10,4),
    is_spot_instance BOOLEAN DEFAULT false,
    
    -- Metrics (updated by monitoring)
    current_utilization REAL DEFAULT 0.0,
    average_utilization_24h REAL,
    jobs_completed_24h INTEGER DEFAULT 0,
    
    -- Timestamps
    last_heartbeat_at TIMESTAMP DEFAULT NOW(),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_gpu_nodes_type ON gpu_nodes(gpu_type);
CREATE INDEX idx_gpu_nodes_status ON gpu_nodes(status, is_schedulable);
CREATE INDEX idx_gpu_nodes_region ON gpu_nodes(region);
CREATE INDEX idx_gpu_nodes_utilization ON gpu_nodes(current_utilization);
```

### 7. Job Queue Table
```sql
CREATE TABLE job_queue (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generation_id UUID NOT NULL REFERENCES generations(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    
    -- Queue Management
    priority_score INTEGER NOT NULL, -- Higher = more priority
    user_tier VARCHAR(20) NOT NULL,
    required_gpu_type VARCHAR(20),
    estimated_duration_seconds INTEGER,
    
    -- Resource Requirements
    gpu_memory_required_gb INTEGER DEFAULT 24,
    cpu_cores_required INTEGER DEFAULT 4,
    memory_required_gb INTEGER DEFAULT 16,
    
    -- Queue Status
    status VARCHAR(20) DEFAULT 'queued' CHECK (
        status IN ('queued', 'assigned', 'cancelled', 'expired')
    ),
    assigned_node_id VARCHAR(100) REFERENCES gpu_nodes(id),
    
    -- Timing
    queued_at TIMESTAMP DEFAULT NOW(),
    assigned_at TIMESTAMP,
    max_wait_time_seconds INTEGER DEFAULT 3600, -- 1 hour timeout
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes for efficient queue processing
CREATE INDEX idx_job_queue_priority ON job_queue(priority_score DESC, queued_at ASC) 
    WHERE status = 'queued';
CREATE INDEX idx_job_queue_gpu_type ON job_queue(required_gpu_type, priority_score DESC) 
    WHERE status = 'queued';
CREATE INDEX idx_job_queue_user_tier ON job_queue(user_tier, priority_score DESC) 
    WHERE status = 'queued';
CREATE INDEX idx_job_queue_timeout ON job_queue(queued_at) 
    WHERE status = 'queued' AND queued_at < NOW() - INTERVAL '1 hour';
```

### 8. Content Moderation Table
```sql
CREATE TABLE content_moderation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    generation_id UUID REFERENCES generations(id),
    
    -- Content Analysis
    content_type VARCHAR(20) NOT NULL CHECK (content_type IN ('prompt', 'image', 'video')),
    content_url TEXT,
    content_text TEXT,
    
    -- Moderation Results
    is_safe BOOLEAN DEFAULT true,
    confidence_score REAL,
    flagged_categories TEXT[], -- ['nsfw', 'violence', 'hate_speech', etc.]
    
    -- Moderation Methods
    automated_result JSONB, -- Results from automated systems
    human_review_result JSONB, -- Results from human review
    human_reviewer_id UUID,
    
    -- Actions Taken
    action_taken VARCHAR(20), -- 'approved', 'rejected', 'requires_review'
    rejection_reason TEXT,
    
    -- Appeals
    appeal_submitted_at TIMESTAMP,
    appeal_resolved_at TIMESTAMP,
    appeal_result VARCHAR(20),
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_moderation_generation_id ON content_moderation(generation_id);
CREATE INDEX idx_moderation_action ON content_moderation(action_taken);
CREATE INDEX idx_moderation_flagged ON content_moderation(is_safe) WHERE is_safe = false;
```

---

## Redis Schema

### 1. Session Management
```redis
# User session data
SET session:{user_id} '{
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "tier": "premium",
    "credits": 150,
    "last_activity": "2024-02-25T10:30:00Z"
}' EX 86400  # 24 hour TTL

# Session token lookup
SET token:{jwt_token} {user_id} EX 86400

# Refresh token
SET refresh_token:{refresh_token} {user_id} EX 2592000  # 30 days
```

### 2. Rate Limiting
```redis
# API rate limits
INCR ratelimit:api:{user_id}:hourly EX 3600
INCR ratelimit:api:{user_id}:daily EX 86400

# Generation rate limits
INCR ratelimit:generation:{user_id}:daily EX 86400
INCR ratelimit:generation:{user_id}:concurrent

# IP-based rate limiting
INCR ratelimit:ip:{ip_address}:minute EX 60
```

### 3. Job Queues (Using Redis Sorted Sets)
```redis
# Priority queue for generations
ZADD generation_queue {priority_score} {job_id}

# Separate queues by GPU type
ZADD queue:gpu_a100 {priority_score} {job_id}
ZADD queue:gpu_v100 {priority_score} {job_id}
ZADD queue:gpu_t4 {priority_score} {job_id}

# Processing jobs (temporary storage)
HSET processing_jobs {job_id} '{
    "node_id": "gpu-node-001",
    "gpu_id": "0",
    "started_at": "2024-02-25T10:30:00Z",
    "estimated_completion": "2024-02-25T10:32:00Z"
}'
```

### 4. Real-time Job Status
```redis
# Job progress tracking
HSET job_status:{job_id} progress 45
HSET job_status:{job_id} status "processing"
HSET job_status:{job_id} eta_seconds 120
EXPIRE job_status:{job_id} 259200  # 3 days

# WebSocket connection mapping
SET websocket:{job_id} {connection_id} EX 7200  # 2 hours
```

### 5. Cache Layers
```redis
# User credits cache
SET credits:{user_id} 150 EX 3600  # 1 hour cache

# Generation metadata cache
SET generation_meta:{job_id} '{
    "status": "completed",
    "video_url": "https://cdn.example.com/video.mp4",
    "thumbnail_url": "https://cdn.example.com/thumb.jpg"
}' EX 3600

# Prompt translation cache
SET translate:{prompt_hash}:{lang_pair} '{
    "original": "Hello world",
    "translated": "Hola mundo",
    "confidence": 0.95
}' EX 86400
```

### 6. Analytics & Metrics
```redis
# Daily active users
SADD dau:2024-02-25 {user_id}
EXPIRE dau:2024-02-25 172800  # 2 days

# Generation counts by type
INCR stats:generations:total:daily:2024-02-25
INCR stats:generations:premium:daily:2024-02-25
INCR stats:generations:gpu_a100:daily:2024-02-25

# GPU utilization metrics
HSET gpu_metrics:gpu-node-001 utilization 85.5
HSET gpu_metrics:gpu-node-001 jobs_running 3
HSET gpu_metrics:gpu-node-001 last_update 1708859400
```

---

## Time Series Schema (InfluxDB)

### 1. Generation Metrics
```sql
-- Measurement: generation_metrics
SELECT 
  mean(inference_time_seconds) as avg_inference_time,
  count(*) as generation_count
FROM generation_metrics 
WHERE time >= now() - 1h 
GROUP BY time(5m), gpu_type;

-- Fields:
-- inference_time_seconds (float)
-- queue_wait_time_seconds (float)
-- total_processing_time_seconds (float)
-- credits_cost (int)
-- file_size_bytes (int)

-- Tags:
-- user_tier, gpu_type, model_version, region, status
```

### 2. System Metrics
```sql
-- Measurement: gpu_utilization
SELECT 
  mean(utilization_percent) as avg_utilization
FROM gpu_utilization 
WHERE time >= now() - 1h 
GROUP BY time(1m), node_id, gpu_type;

-- Fields:
-- utilization_percent (float)
-- memory_used_gb (float)
-- temperature_celsius (float)
-- power_usage_watts (float)

-- Tags:
-- node_id, gpu_id, gpu_type, region
```

### 3. Business Metrics
```sql
-- Measurement: user_activity
SELECT 
  count(distinct(user_id)) as active_users
FROM user_activity 
WHERE time >= now() - 1d 
GROUP BY time(1h);

-- Fields:
-- session_duration_seconds (int)
-- api_calls_count (int)
-- generations_created (int)
-- credits_spent (int)

-- Tags:
-- user_id, user_tier, device_type, country
```

---

## Database Triggers & Functions

### 1. Update Timestamps
```sql
-- Function to update updated_at column
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to all tables that need it
CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_generations_updated_at 
    BEFORE UPDATE ON generations 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```

### 2. Credit Balance Validation
```sql
-- Ensure credit balance never goes negative
CREATE OR REPLACE FUNCTION validate_credit_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.credits < 0 THEN
        RAISE EXCEPTION 'Credit balance cannot be negative. Current: %, Attempted: %', 
            OLD.credits, NEW.credits;
    END IF;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER validate_user_credits 
    BEFORE UPDATE ON users 
    FOR EACH ROW 
    EXECUTE FUNCTION validate_credit_balance();
```

### 3. Job Queue Management
```sql
-- Automatically calculate priority score
CREATE OR REPLACE FUNCTION calculate_priority_score()
RETURNS TRIGGER AS $$
DECLARE
    base_priority INTEGER;
    tier_multiplier INTEGER;
    wait_time_bonus INTEGER;
BEGIN
    -- Base priority by tier
    base_priority := CASE NEW.user_tier
        WHEN 'premium' THEN 1000
        WHEN 'standard' THEN 500
        WHEN 'free' THEN 100
        ELSE 100
    END;
    
    -- Time-based priority boost
    wait_time_bonus := EXTRACT(EPOCH FROM (NOW() - NEW.queued_at)) / 60; -- 1 point per minute
    
    NEW.priority_score := base_priority + wait_time_bonus;
    
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER calculate_job_priority 
    BEFORE INSERT OR UPDATE ON job_queue 
    FOR EACH ROW 
    EXECUTE FUNCTION calculate_priority_score();
```

---

## Data Migration Scripts

### 1. Initial Schema Setup
```sql
-- Run this script to set up the complete database schema
\i 001_create_users_table.sql
\i 002_create_generations_table.sql
\i 003_create_credit_transactions_table.sql
\i 004_create_sessions_table.sql
\i 005_create_analytics_table.sql
\i 006_create_gpu_nodes_table.sql
\i 007_create_job_queue_table.sql
\i 008_create_moderation_table.sql
\i 009_create_triggers.sql
\i 010_create_indexes.sql
\i 011_insert_initial_data.sql
```

### 2. Performance Optimization
```sql
-- Partition large tables by date
CREATE TABLE generations_y2024m02 PARTITION OF generations
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create materialized views for analytics
CREATE MATERIALIZED VIEW daily_generation_stats AS
SELECT 
    DATE(created_at) as date,
    status,
    COUNT(*) as count,
    AVG(inference_time_seconds) as avg_inference_time,
    SUM(credits_cost) as total_credits
FROM generations 
WHERE created_at >= NOW() - INTERVAL '90 days'
GROUP BY DATE(created_at), status;

-- Refresh schedule
CREATE OR REPLACE FUNCTION refresh_daily_stats()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_generation_stats;
END;
$$ LANGUAGE plpgsql;
```

This comprehensive database schema covers all aspects of the Image-to-Video platform, from user management to GPU resource tracking, with proper indexing, triggers, and optimization strategies for scale.

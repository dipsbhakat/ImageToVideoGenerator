# Component-Level Low-Level Design

## 1. Authentication Service (Go)

### Service Architecture
```
┌─────────────────────────────────────────┐
│            Auth Service                 │
├─────────────────────────────────────────┤
│  ┌─────────────┐ ┌─────────────────────┐ │
│  │   JWT       │ │    Session          │ │
│  │ Manager     │ │   Manager           │ │
│  └─────────────┘ └─────────────────────┘ │
│  ┌─────────────┐ ┌─────────────────────┐ │
│  │  Password   │ │    Rate             │ │
│  │  Manager    │ │   Limiter           │ │
│  └─────────────┘ └─────────────────────┘ │
│  ┌─────────────┐ ┌─────────────────────┐ │
│  │   User      │ │    Audit            │ │
│  │ Repository  │ │   Logger            │ │
│  └─────────────┘ └─────────────────────┘ │
└─────────────────────────────────────────┘
```

### Implementation Details

#### JWT Manager
```go
package jwt

import (
    "time"
    "crypto/rand"
    "github.com/golang-jwt/jwt/v5"
)

type JWTManager struct {
    secretKey     []byte
    tokenDuration time.Duration
    issuer        string
}

type CustomClaims struct {
    UserID    string `json:"user_id"`
    Email     string `json:"email"`
    Tier      string `json:"tier"`
    Credits   int    `json:"credits"`
    Roles     []string `json:"roles"`
    jwt.RegisteredClaims
}

func NewJWTManager(secret []byte, duration time.Duration, issuer string) *JWTManager {
    return &JWTManager{
        secretKey:     secret,
        tokenDuration: duration,
        issuer:        issuer,
    }
}

func (manager *JWTManager) GenerateToken(user *User) (*TokenPair, error) {
    // Generate access token
    accessClaims := &CustomClaims{
        UserID:  user.ID,
        Email:   user.Email,
        Tier:    user.Tier,
        Credits: user.Credits,
        Roles:   user.Roles,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(manager.tokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            NotBefore: jwt.NewNumericDate(time.Now()),
            Issuer:    manager.issuer,
            Subject:   user.ID,
        },
    }
    
    accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims)
    accessTokenString, err := accessToken.SignedString(manager.secretKey)
    if err != nil {
        return nil, fmt.Errorf("cannot sign access token: %w", err)
    }
    
    // Generate refresh token (longer duration, fewer claims)
    refreshClaims := &jwt.RegisteredClaims{
        ExpiresAt: jwt.NewNumericDate(time.Now().Add(30 * 24 * time.Hour)), // 30 days
        IssuedAt:  jwt.NewNumericDate(time.Now()),
        Issuer:    manager.issuer,
        Subject:   user.ID,
    }
    
    refreshToken := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims)
    refreshTokenString, err := refreshToken.SignedString(manager.secretKey)
    if err != nil {
        return nil, fmt.Errorf("cannot sign refresh token: %w", err)
    }
    
    return &TokenPair{
        AccessToken:  accessTokenString,
        RefreshToken: refreshTokenString,
        ExpiresIn:    int(manager.tokenDuration.Seconds()),
        TokenType:    "Bearer",
    }, nil
}

func (manager *JWTManager) ValidateToken(tokenString string) (*CustomClaims, error) {
    token, err := jwt.ParseWithClaims(
        tokenString,
        &CustomClaims{},
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected token signing method")
            }
            return manager.secretKey, nil
        },
    )
    
    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }
    
    claims, ok := token.Claims.(*CustomClaims)
    if !ok || !token.Valid {
        return nil, fmt.Errorf("invalid token claims")
    }
    
    return claims, nil
}
```

#### Password Manager
```go
package auth

import (
    "crypto/rand"
    "crypto/subtle"
    "encoding/base64"
    "fmt"
    "golang.org/x/crypto/argon2"
)

type PasswordManager struct {
    memory      uint32
    iterations  uint32
    parallelism uint8
    saltLength  uint32
    keyLength   uint32
}

type PasswordConfig struct {
    format      string
    version     int
    memory      uint32
    iterations  uint32
    parallelism uint8
    salt        []byte
    hash        []byte
}

func NewPasswordManager() *PasswordManager {
    return &PasswordManager{
        memory:      64 * 1024, // 64 MB
        iterations:  3,
        parallelism: 2,
        saltLength:  16,
        keyLength:   32,
    }
}

func (pm *PasswordManager) HashPassword(password string) (string, error) {
    salt, err := pm.generateRandomBytes(pm.saltLength)
    if err != nil {
        return "", err
    }
    
    hash := argon2.IDKey([]byte(password), salt, pm.iterations, pm.memory, pm.parallelism, pm.keyLength)
    
    // Encode to base64 for storage
    b64Salt := base64.RawStdEncoding.EncodeToString(salt)
    b64Hash := base64.RawStdEncoding.EncodeToString(hash)
    
    // Format: $argon2id$v=19$m=65536,t=3,p=2$salt$hash
    encodedHash := fmt.Sprintf("$argon2id$v=%d$m=%d,t=%d,p=%d$%s$%s",
        argon2.Version, pm.memory, pm.iterations, pm.parallelism, b64Salt, b64Hash)
    
    return encodedHash, nil
}

func (pm *PasswordManager) VerifyPassword(password, encodedHash string) (bool, error) {
    config, err := pm.decodeHash(encodedHash)
    if err != nil {
        return false, err
    }
    
    otherHash := argon2.IDKey([]byte(password), config.salt, config.iterations, 
        config.memory, config.parallelism, uint32(len(config.hash)))
    
    // Use constant-time comparison
    if subtle.ConstantTimeCompare(config.hash, otherHash) == 1 {
        return true, nil
    }
    return false, nil
}

func (pm *PasswordManager) generateRandomBytes(n uint32) ([]byte, error) {
    b := make([]byte, n)
    _, err := rand.Read(b)
    if err != nil {
        return nil, err
    }
    return b, nil
}
```

#### Rate Limiter
```go
package ratelimit

import (
    "context"
    "fmt"
    "time"
    "github.com/go-redis/redis/v8"
)

type RateLimiter struct {
    client *redis.Client
}

type RateLimitConfig struct {
    Requests int
    Window   time.Duration
    BurstAllowed int
}

type RateLimitResult struct {
    Allowed      bool
    Remaining    int
    ResetTime    time.Time
    RetryAfter   time.Duration
}

func NewRateLimiter(redisClient *redis.Client) *RateLimiter {
    return &RateLimiter{client: redisClient}
}

func (rl *RateLimiter) Check(ctx context.Context, key string, config RateLimitConfig) (*RateLimitResult, error) {
    now := time.Now()
    window := now.Truncate(config.Window)
    
    // Sliding window log implementation
    pipe := rl.client.TxPipeline()
    
    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%d", window.Unix()-int64(config.Window.Seconds())))
    
    // Count current requests
    pipe.ZCard(ctx, key)
    
    // Add current request
    pipe.ZAdd(ctx, key, &redis.Z{
        Score:  float64(now.Unix()),
        Member: fmt.Sprintf("%d-%d", now.Unix(), now.Nanosecond()),
    })
    
    // Set expiry
    pipe.Expire(ctx, key, config.Window*2)
    
    results, err := pipe.Exec(ctx)
    if err != nil {
        return nil, fmt.Errorf("redis pipeline error: %w", err)
    }
    
    currentCount := results[1].(*redis.IntCmd).Val()
    
    allowed := currentCount <= int64(config.Requests)
    remaining := max(0, config.Requests-int(currentCount))
    resetTime := window.Add(config.Window)
    
    result := &RateLimitResult{
        Allowed:   allowed,
        Remaining: remaining,
        ResetTime: resetTime,
    }
    
    if !allowed {
        result.RetryAfter = time.Until(resetTime)
    }
    
    return result, nil
}

// Token bucket rate limiter for burst handling
func (rl *RateLimiter) CheckTokenBucket(ctx context.Context, key string, config RateLimitConfig) (*RateLimitResult, error) {
    luaScript := `
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local tokens = tonumber(ARGV[2])
        local interval = tonumber(ARGV[3])
        local current_time = tonumber(ARGV[4])
        
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local current_tokens = tonumber(bucket[1])
        local last_refill = tonumber(bucket[2])
        
        if current_tokens == nil then
            current_tokens = capacity
            last_refill = current_time
        end
        
        -- Calculate tokens to add based on time passed
        local time_passed = current_time - last_refill
        local tokens_to_add = math.floor(time_passed / interval) * tokens
        current_tokens = math.min(capacity, current_tokens + tokens_to_add)
        
        -- Check if request can be served
        if current_tokens >= 1 then
            current_tokens = current_tokens - 1
            redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', current_time)
            redis.call('EXPIRE', key, interval * capacity)
            return {1, current_tokens}
        else
            redis.call('HMSET', key, 'tokens', current_tokens, 'last_refill', current_time)
            redis.call('EXPIRE', key, interval * capacity)
            return {0, current_tokens}
        end
    `
    
    now := time.Now().Unix()
    result := rl.client.Eval(ctx, luaScript, []string{key}, 
        config.BurstAllowed, 1, int64(config.Window.Seconds()), now)
    
    values, err := result.Result()
    if err != nil {
        return nil, fmt.Errorf("token bucket script error: %w", err)
    }
    
    vals := values.([]interface{})
    allowed := vals[0].(int64) == 1
    remaining := int(vals[1].(int64))
    
    return &RateLimitResult{
        Allowed:   allowed,
        Remaining: remaining,
        ResetTime: time.Unix(now+int64(config.Window.Seconds()), 0),
    }, nil
}
```

---

## 2. Generation Orchestrator (Python)

### Service Architecture
```
┌─────────────────────────────────────────┐
│        Generation Orchestrator          │
├─────────────────────────────────────────┤
│ ┌─────────────┐ ┌─────────────────────┐ │
│ │   Request   │ │     Queue           │ │
│ │ Validator   │ │    Manager          │ │
│ └─────────────┘ └─────────────────────┘ │
│ ┌─────────────┐ ┌─────────────────────┐ │
│ │    Credit   │ │      Job            │ │
│ │  Manager    │ │   Tracker           │ │
│ └─────────────┘ └─────────────────────┘ │
│ ┌─────────────┐ ┌─────────────────────┐ │
│ │ GPU Resource│ │   Notification      │ │
│ │ Allocator   │ │    Service          │ │
│ └─────────────┘ └─────────────────────┘ │
└─────────────────────────────────────────┘
```

### Implementation Details

#### Request Validator
```python
# orchestrator/validators.py
from pydantic import BaseModel, validator, Field
from typing import Optional, Dict, Any
import mimetypes
import validators
from enum import Enum

class QualityLevel(str, Enum):
    ECONOMY = "economy"
    STANDARD = "standard"
    HIGH = "high"
    PREMIUM = "premium"

class AspectRatio(str, Enum):
    SQUARE = "1:1"
    LANDSCAPE = "16:9"
    PORTRAIT = "9:16"
    VERTICAL = "4:5"

class GenerationRequest(BaseModel):
    user_id: str = Field(..., min_length=1, max_length=255)
    image_url: str = Field(..., max_length=2048)
    prompt: str = Field(..., min_length=5, max_length=2000)
    language: str = Field(default="en", min_length=2, max_length=5)
    
    # Generation settings
    duration: int = Field(default=10, ge=5, le=30)
    quality: QualityLevel = Field(default=QualityLevel.STANDARD)
    fps: int = Field(default=25, ge=15, le=60)
    aspect_ratio: AspectRatio = Field(default=AspectRatio.LANDSCAPE)
    
    # Advanced options
    seed: Optional[int] = Field(None, ge=0, le=2**32-1)
    guidance_scale: float = Field(default=7.5, ge=1.0, le=20.0)
    num_inference_steps: int = Field(default=20, ge=10, le=50)
    
    # Business settings
    is_public: bool = Field(default=False)
    enhance_prompt: bool = Field(default=True)
    priority: str = Field(default="normal")
    
    @validator('image_url')
    def validate_image_url(cls, v):
        if not validators.url(v):
            raise ValueError('Invalid image URL format')
        
        # Check if it's likely an image based on URL
        if not any(v.lower().endswith(ext) for ext in ['.jpg', '.jpeg', '.png', '.webp']):
            # Try to guess from content-type if available
            content_type, _ = mimetypes.guess_type(v)
            if not content_type or not content_type.startswith('image/'):
                raise ValueError('URL does not appear to be an image')
        
        return v
    
    @validator('prompt')
    def validate_prompt(cls, v):
        # Basic content validation
        forbidden_words = ['explicit', 'nsfw', 'violence']  # Simplified list
        if any(word in v.lower() for word in forbidden_words):
            raise ValueError('Prompt contains forbidden content')
        
        # Check for meaningful content
        words = v.split()
        if len(words) < 3:
            raise ValueError('Prompt must contain at least 3 words')
        
        return v.strip()
    
    @validator('priority')
    def validate_priority(cls, v):
        allowed_priorities = ['low', 'normal', 'high', 'urgent']
        if v not in allowed_priorities:
            raise ValueError(f'Priority must be one of {allowed_priorities}')
        return v

class RequestValidator:
    def __init__(self, credit_manager, content_moderator):
        self.credit_manager = credit_manager
        self.content_moderator = content_moderator
    
    async def validate_request(self, request: GenerationRequest) -> Dict[str, Any]:
        """Comprehensive request validation with business rules"""
        
        validation_result = {
            'valid': True,
            'errors': [],
            'warnings': [],
            'credit_cost': 0,
            'estimated_duration': 0
        }
        
        try:
            # 1. Basic schema validation (already done by Pydantic)
            
            # 2. User validation
            user_validation = await self._validate_user(request.user_id)
            if not user_validation['valid']:
                validation_result['errors'].extend(user_validation['errors'])
            
            # 3. Content moderation
            content_validation = await self._validate_content(
                request.prompt, request.image_url
            )
            if not content_validation['safe']:
                validation_result['errors'].append(
                    f"Content failed safety check: {content_validation['reason']}"
                )
            
            # 4. Credit calculation and validation
            credit_cost = self._calculate_credit_cost(request)
            validation_result['credit_cost'] = credit_cost
            
            user_credits = await self.credit_manager.get_user_credits(request.user_id)
            if user_credits < credit_cost:
                validation_result['errors'].append(
                    f"Insufficient credits. Required: {credit_cost}, Available: {user_credits}"
                )
            
            # 5. Rate limiting check
            rate_limit_check = await self._check_rate_limits(request.user_id)
            if not rate_limit_check['allowed']:
                validation_result['errors'].append(
                    f"Rate limit exceeded. Try again in {rate_limit_check['retry_after']} seconds"
                )
            
            # 6. Resource availability
            resource_check = await self._check_resource_availability(request)
            if not resource_check['available']:
                validation_result['warnings'].append(
                    f"High queue load. Estimated wait time: {resource_check['eta_seconds']}s"
                )
                validation_result['estimated_duration'] = resource_check['eta_seconds']
            
            # Set overall validity
            validation_result['valid'] = len(validation_result['errors']) == 0
            
        except Exception as e:
            validation_result['valid'] = False
            validation_result['errors'].append(f"Validation error: {str(e)}")
        
        return validation_result
    
    def _calculate_credit_cost(self, request: GenerationRequest) -> int:
        """Calculate credit cost based on generation parameters"""
        
        base_cost = {
            QualityLevel.ECONOMY: 1,
            QualityLevel.STANDARD: 3,
            QualityLevel.HIGH: 5,
            QualityLevel.PREMIUM: 8
        }
        
        cost = base_cost[request.quality]
        
        # Duration multiplier
        if request.duration > 15:
            cost *= 1.5
        elif request.duration > 20:
            cost *= 2.0
        
        # High FPS multiplier
        if request.fps > 30:
            cost *= 1.2
        
        # Premium features
        if request.num_inference_steps > 30:
            cost *= 1.3
        
        return int(cost)
    
    async def _validate_user(self, user_id: str) -> Dict[str, Any]:
        """Validate user exists and is in good standing"""
        # Implementation would check user database
        return {'valid': True, 'errors': []}
    
    async def _validate_content(self, prompt: str, image_url: str) -> Dict[str, Any]:
        """Run content moderation checks"""
        return await self.content_moderator.check_content(prompt, image_url)
    
    async def _check_rate_limits(self, user_id: str) -> Dict[str, Any]:
        """Check if user is within rate limits"""
        # Implementation would check Redis rate limit counters
        return {'allowed': True, 'retry_after': 0}
    
    async def _check_resource_availability(self, request: GenerationRequest) -> Dict[str, Any]:
        """Check GPU and queue availability"""
        # Implementation would check current queue length and GPU utilization
        return {'available': True, 'eta_seconds': 30}
```

#### Queue Manager
```python
# orchestrator/queue_manager.py
import redis
import json
import time
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, asdict
from enum import Enum
import logging

class JobStatus(str, Enum):
    PENDING = "pending"
    QUEUED = "queued"
    ASSIGNED = "assigned"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class JobPriority(str, Enum):
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"
    URGENT = "urgent"

@dataclass
class QueueJob:
    job_id: str
    user_id: str
    user_tier: str
    generation_request: Dict[str, Any]
    priority: JobPriority
    priority_score: int
    created_at: float
    gpu_requirements: Dict[str, Any]
    estimated_duration: int
    retry_count: int = 0
    max_retries: int = 3
    status: JobStatus = JobStatus.PENDING
    assigned_node: Optional[str] = None
    assigned_gpu: Optional[str] = None
    error_message: Optional[str] = None

class QueueManager:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.logger = logging.getLogger(__name__)
        
        # Queue keys
        self.MAIN_QUEUE = "generation_queue"
        self.PRIORITY_QUEUE = "priority_queue"
        self.PROCESSING_JOBS = "processing_jobs"
        self.FAILED_JOBS = "failed_jobs"
        self.JOB_DATA = "job_data"
        
        # Queue configurations
        self.PRIORITY_WEIGHTS = {
            JobPriority.LOW: 100,
            JobPriority.NORMAL: 500,
            JobPriority.HIGH: 1000,
            JobPriority.URGENT: 2000
        }
        
        self.TIER_MULTIPLIERS = {
            'free': 1.0,
            'standard': 1.5,
            'premium': 2.0,
            'enterprise': 3.0
        }
    
    async def enqueue_job(self, job: QueueJob) -> bool:
        """Add job to the queue with priority scoring"""
        
        try:
            # Calculate priority score
            job.priority_score = self._calculate_priority_score(job)
            
            # Store job data
            job_key = f"{self.JOB_DATA}:{job.job_id}"
            await self.redis.setex(
                job_key, 
                24 * 3600,  # 24 hour TTL
                json.dumps(asdict(job), default=str)
            )
            
            # Add to priority queue (sorted set)
            await self.redis.zadd(
                self.MAIN_QUEUE,
                {job.job_id: job.priority_score}
            )
            
            # Update job status
            job.status = JobStatus.QUEUED
            await self._update_job_status(job.job_id, job.status)
            
            self.logger.info(f"Job {job.job_id} enqueued with priority {job.priority_score}")
            return True
            
        except Exception as e:
            self.logger.error(f"Failed to enqueue job {job.job_id}: {str(e)}")
            return False
    
    async def dequeue_job(self, gpu_type: str = None) -> Optional[QueueJob]:
        """Get next job from queue based on priority and GPU requirements"""
        
        try:
            # Get highest priority job
            job_data = await self.redis.zpopmax(self.MAIN_QUEUE)
            
            if not job_data:
                return None
            
            job_id = job_data[0][0].decode('utf-8')
            
            # Retrieve full job data
            job_key = f"{self.JOB_DATA}:{job_id}"
            job_json = await self.redis.get(job_key)
            
            if not job_json:
                self.logger.error(f"Job data not found for {job_id}")
                return None
            
            job_dict = json.loads(job_json)
            job = QueueJob(**job_dict)
            
            # Check GPU compatibility
            if gpu_type and not self._is_gpu_compatible(job, gpu_type):
                # Re-queue the job and try next one
                await self.redis.zadd(self.MAIN_QUEUE, {job_id: job.priority_score})
                return await self.dequeue_job(gpu_type)
            
            # Mark as assigned
            job.status = JobStatus.ASSIGNED
            await self._update_job_status(job_id, job.status)
            
            return job
            
        except Exception as e:
            self.logger.error(f"Failed to dequeue job: {str(e)}")
            return None
    
    async def mark_job_processing(self, job_id: str, node_id: str, gpu_id: str) -> bool:
        """Mark job as currently processing"""
        
        try:
            # Update job data
            job_key = f"{self.JOB_DATA}:{job_id}"
            job_json = await self.redis.get(job_key)
            
            if not job_json:
                return False
            
            job_dict = json.loads(job_json)
            job_dict['status'] = JobStatus.PROCESSING
            job_dict['assigned_node'] = node_id
            job_dict['assigned_gpu'] = gpu_id
            
            await self.redis.setex(job_key, 24 * 3600, json.dumps(job_dict, default=str))
            
            # Add to processing jobs set
            processing_key = f"{self.PROCESSING_JOBS}:{node_id}:{gpu_id}"
            await self.redis.setex(processing_key, 3600, job_id)  # 1 hour timeout
            
            self.logger.info(f"Job {job_id} marked as processing on {node_id}:{gpu_id}")
            return True
            
        except Exception as e:
            self.logger.error(f"Failed to mark job {job_id} as processing: {str(e)}")
            return False
    
    async def complete_job(self, job_id: str, result: Dict[str, Any]) -> bool:
        """Mark job as completed with results"""
        
        try:
            # Update job status
            await self._update_job_status(job_id, JobStatus.COMPLETED)
            
            # Store result
            result_key = f"job_result:{job_id}"
            await self.redis.setex(
                result_key,
                7 * 24 * 3600,  # 7 days
                json.dumps(result, default=str)
            )
            
            # Remove from processing
            await self._cleanup_processing_job(job_id)
            
            self.logger.info(f"Job {job_id} completed successfully")
            return True
            
        except Exception as e:
            self.logger.error(f"Failed to complete job {job_id}: {str(e)}")
            return False
    
    async def fail_job(self, job_id: str, error_message: str, retryable: bool = True) -> bool:
        """Mark job as failed and optionally retry"""
        
        try:
            job_key = f"{self.JOB_DATA}:{job_id}"
            job_json = await self.redis.get(job_key)
            
            if not job_json:
                return False
            
            job_dict = json.loads(job_json)
            job = QueueJob(**job_dict)
            
            job.retry_count += 1
            job.error_message = error_message
            
            # Check if we should retry
            if retryable and job.retry_count < job.max_retries:
                # Re-queue with lower priority
                job.priority_score = max(1, job.priority_score * 0.8)
                job.status = JobStatus.PENDING
                
                await self.redis.setex(job_key, 24 * 3600, json.dumps(asdict(job), default=str))
                await self.redis.zadd(self.MAIN_QUEUE, {job_id: job.priority_score})
                
                self.logger.info(f"Job {job_id} re-queued for retry {job.retry_count}")
            else:
                # Mark as permanently failed
                job.status = JobStatus.FAILED
                await self._update_job_status(job_id, job.status)
                
                # Move to failed jobs
                await self.redis.zadd(self.FAILED_JOBS, {job_id: time.time()})
                
                self.logger.error(f"Job {job_id} permanently failed: {error_message}")
            
            # Cleanup processing state
            await self._cleanup_processing_job(job_id)
            
            return True
            
        except Exception as e:
            self.logger.error(f"Failed to handle job failure {job_id}: {str(e)}")
            return False
    
    def _calculate_priority_score(self, job: QueueJob) -> int:
        """Calculate job priority score for queue ordering"""
        
        base_score = self.PRIORITY_WEIGHTS[job.priority]
        tier_multiplier = self.TIER_MULTIPLIERS.get(job.user_tier, 1.0)
        
        # Time-based boost (older jobs get higher priority)
        age_boost = min(500, (time.time() - job.created_at) / 60)  # 1 point per minute, max 500
        
        # Resource requirement penalty (complex jobs get lower priority)
        complexity_penalty = 0
        if job.estimated_duration > 60:
            complexity_penalty = 100
        
        final_score = int((base_score * tier_multiplier) + age_boost - complexity_penalty)
        
        return max(1, final_score)
    
    def _is_gpu_compatible(self, job: QueueJob, gpu_type: str) -> bool:
        """Check if job can run on the given GPU type"""
        
        requirements = job.gpu_requirements
        required_memory = requirements.get('memory_gb', 16)
        
        gpu_specs = {
            'T4': {'memory': 16, 'compute': 7.5},
            'V100': {'memory': 32, 'compute': 7.0},
            'A100': {'memory': 40, 'compute': 8.0}
        }
        
        if gpu_type not in gpu_specs:
            return False
        
        return gpu_specs[gpu_type]['memory'] >= required_memory
    
    async def _update_job_status(self, job_id: str, status: JobStatus):
        """Update job status in Redis"""
        status_key = f"job_status:{job_id}"
        await self.redis.setex(status_key, 3600, status.value)
    
    async def _cleanup_processing_job(self, job_id: str):
        """Remove job from processing state"""
        # Find and remove from processing jobs
        pattern = f"{self.PROCESSING_JOBS}:*"
        keys = await self.redis.keys(pattern)
        
        for key in keys:
            value = await self.redis.get(key)
            if value and value.decode('utf-8') == job_id:
                await self.redis.delete(key)
                break
```

This Low-Level Design provides detailed implementation for the core components of the Image-to-Video AI platform, focusing on production-ready code with proper error handling, logging, and scalability considerations.

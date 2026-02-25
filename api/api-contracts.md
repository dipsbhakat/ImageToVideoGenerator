# Low-Level Design: Image-to-Video AI Platform

## Component-Level Architecture

### 1. API Gateway Implementation

#### Technology: Kong Gateway + Lua Plugins

```lua
-- Rate limiting plugin
local function rate_limit_by_user(user_id, tier)
    local key = "rate_limit:" .. user_id
    local limit = get_user_limit(tier) -- free: 5/day, premium: unlimited
    
    local current = redis.get(key) or 0
    if current >= limit then
        return kong.response.exit(429, {
            error = "Rate limit exceeded",
            retry_after = get_reset_time(key)
        })
    end
    
    redis.incr(key)
    redis.expire(key, 86400) -- 24 hours
end

-- Credit validation plugin
local function validate_credits(user_id, operation_cost)
    local credits = redis.get("credits:" .. user_id)
    if credits < operation_cost then
        return kong.response.exit(402, {
            error = "Insufficient credits",
            required = operation_cost,
            available = credits
        })
    end
end
```

#### Service Discovery & Load Balancing
```yaml
# Kong configuration
services:
  - name: auth-service
    url: http://auth-service:8080
    targets:
      - weight: 100
        target: auth-service-v1:8080
      - weight: 0
        target: auth-service-v2:8080  # Canary deployment
    
  - name: generation-service
    url: http://generation-orchestrator:8080
    load_balancing_algorithm: consistent-hashing
    health_checks:
      passive:
        unhealthy:
          http_failures: 3
          timeouts: 3
      active:
        http_path: /health
        interval: 30
```

### 2. Authentication Service

#### Technology: Go + JWT + Redis

```go
// auth/service.go
package auth

import (
    "time"
    "github.com/dgrijalva/jwt-go"
    "github.com/go-redis/redis/v8"
)

type AuthService struct {
    redisClient *redis.Client
    jwtSecret   []byte
    tokenTTL    time.Duration
}

type JWTClaims struct {
    UserID    string `json:"user_id"`
    Email     string `json:"email"`
    Tier      string `json:"tier"`
    Credits   int    `json:"credits"`
    jwt.StandardClaims
}

func (s *AuthService) GenerateToken(user *User) (string, error) {
    claims := &JWTClaims{
        UserID:  user.ID,
        Email:   user.Email,
        Tier:    user.Tier,
        Credits: user.Credits,
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(s.tokenTTL).Unix(),
            IssuedAt:  time.Now().Unix(),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    tokenString, err := token.SignedString(s.jwtSecret)
    
    // Store in Redis for invalidation capability
    key := fmt.Sprintf("session:%s", user.ID)
    s.redisClient.Set(ctx, key, tokenString, s.tokenTTL)
    
    return tokenString, err
}

func (s *AuthService) ValidateToken(tokenString string) (*JWTClaims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &JWTClaims{}, 
        func(token *jwt.Token) (interface{}, error) {
            return s.jwtSecret, nil
        })
    
    if err != nil || !token.Valid {
        return nil, ErrInvalidToken
    }
    
    claims := token.Claims.(*JWTClaims)
    
    // Check if session exists in Redis (not revoked)
    key := fmt.Sprintf("session:%s", claims.UserID)
    exists := s.redisClient.Exists(ctx, key).Val()
    if exists == 0 {
        return nil, ErrSessionExpired
    }
    
    return claims, nil
}
```

#### Database Schema Implementation
```sql
-- Users table with indexes
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    tier VARCHAR(20) DEFAULT 'free' CHECK (tier IN ('free', 'standard', 'premium')),
    credits INTEGER DEFAULT 5,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_tier ON users(tier);
CREATE INDEX idx_users_active ON users(is_active);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at 
    BEFORE UPDATE ON users 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();
```

### 3. Prompt Service Implementation

#### Technology: Python + FastAPI + spaCy

```python
# prompt/service.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import spacy
import openai
from transformers import pipeline

app = FastAPI()

class PromptRequest(BaseModel):
    text: str
    language: str = "en"
    enhance: bool = True
    safety_check: bool = True

class PromptResponse(BaseModel):
    original_text: str
    enhanced_text: Optional[str]
    language_detected: str
    safety_score: float
    is_safe: bool
    suggestions: list[str]

class PromptService:
    def __init__(self):
        self.nlp = spacy.load("en_core_web_sm")
        self.safety_classifier = pipeline(
            "text-classification", 
            model="unitary/toxic-bert"
        )
        self.enhancement_model = openai.OpenAI()
        
    async def process_prompt(self, request: PromptRequest) -> PromptResponse:
        # Language detection
        detected_lang = self._detect_language(request.text)
        
        # Safety check
        safety_result = self._check_safety(request.text)
        
        # Prompt enhancement
        enhanced_text = None
        if request.enhance and safety_result['is_safe']:
            enhanced_text = await self._enhance_prompt(request.text)
        
        return PromptResponse(
            original_text=request.text,
            enhanced_text=enhanced_text,
            language_detected=detected_lang,
            safety_score=safety_result['score'],
            is_safe=safety_result['is_safe'],
            suggestions=safety_result['suggestions']
        )
    
    def _check_safety(self, text: str) -> dict:
        """Multi-layer safety checking"""
        
        # 1. Keyword blacklist check
        blacklist_keywords = self._load_blacklist()
        for keyword in blacklist_keywords:
            if keyword.lower() in text.lower():
                return {
                    'is_safe': False,
                    'score': 0.0,
                    'suggestions': [f"Remove inappropriate content: '{keyword}'"]
                }
        
        # 2. ML-based toxicity detection
        toxicity_result = self.safety_classifier(text)[0]
        toxicity_score = toxicity_result['score'] if toxicity_result['label'] == 'TOXIC' else 1 - toxicity_result['score']
        
        # 3. OpenAI moderation
        moderation_result = openai.Moderation.create(input=text)
        openai_flagged = moderation_result.results[0].flagged
        
        # Combined safety score
        is_safe = toxicity_score < 0.7 and not openai_flagged
        
        return {
            'is_safe': is_safe,
            'score': min(toxicity_score, 0.9 if not openai_flagged else 0.1),
            'suggestions': self._generate_safety_suggestions(text) if not is_safe else []
        }
    
    async def _enhance_prompt(self, text: str) -> str:
        """Enhance prompt for better video generation"""
        
        enhancement_prompt = f"""
        Enhance this image-to-video prompt for better AI generation results.
        Add cinematic details, camera movements, lighting, and motion descriptions.
        Keep the core concept but make it more descriptive and specific.
        
        Original: {text}
        Enhanced:"""
        
        response = await self.enhancement_model.chat.completions.create(
            model="gpt-3.5-turbo",
            messages=[
                {"role": "system", "content": "You are an expert at creating detailed prompts for AI video generation."},
                {"role": "user", "content": enhancement_prompt}
            ],
            max_tokens=200,
            temperature=0.7
        )
        
        return response.choices[0].message.content.strip()

@app.post("/process", response_model=PromptResponse)
async def process_prompt_endpoint(request: PromptRequest):
    service = PromptService()
    return await service.process_prompt(request)
```

### 4. Translation Service Implementation

#### Technology: Python + Google Translate API + Custom Models

```python
# translation/service.py
from google.cloud import translate_v2 as translate
from langdetect import detect
import redis
import json

class TranslationService:
    def __init__(self):
        self.translate_client = translate.Client()
        self.redis_client = redis.Redis(host='redis-cache')
        self.supported_languages = ['en', 'hi', 'zh', 'es', 'fr', 'de', 'ja']
        
    async def translate_prompt(self, text: str, target_lang: str = 'en') -> dict:
        """Translate prompt to target language with caching"""
        
        # Detect source language
        source_lang = detect(text)
        
        # If already in target language, return as-is
        if source_lang == target_lang:
            return {
                'original_text': text,
                'translated_text': text,
                'source_language': source_lang,
                'target_language': target_lang,
                'confidence': 1.0
            }
        
        # Check cache
        cache_key = f"translate:{hash(text)}:{source_lang}:{target_lang}"
        cached_result = self.redis_client.get(cache_key)
        
        if cached_result:
            return json.loads(cached_result)
        
        # Perform translation
        try:
            result = self.translate_client.translate(
                text,
                source_language=source_lang,
                target_language=target_lang
            )
            
            translation_result = {
                'original_text': text,
                'translated_text': result['translatedText'],
                'source_language': source_lang,
                'target_language': target_lang,
                'confidence': result.get('confidence', 0.95)
            }
            
            # Cache for 24 hours
            self.redis_client.setex(
                cache_key, 
                86400, 
                json.dumps(translation_result)
            )
            
            return translation_result
            
        except Exception as e:
            raise TranslationError(f"Translation failed: {str(e)}")
    
    def get_supported_languages(self) -> list:
        return self.supported_languages
```

### 5. Generation Orchestrator Implementation

#### Technology: Python + Celery + Redis + AsyncIO

```python
# orchestrator/service.py
import asyncio
import uuid
from celery import Celery
from sqlalchemy.orm import Session
from typing import Dict, Any
import boto3
from datetime import datetime

# Celery configuration
celery_app = Celery(
    'generation_orchestrator',
    broker='redis://redis:6379/0',
    backend='redis://redis:6379/0'
)

class GenerationRequest:
    def __init__(self, user_id: str, image_url: str, prompt: str, 
                 language: str = 'en', priority: str = 'normal'):
        self.job_id = str(uuid.uuid4())
        self.user_id = user_id
        self.image_url = image_url
        self.prompt = prompt
        self.language = language
        self.priority = priority
        self.created_at = datetime.utcnow()
        self.status = 'pending'
        self.progress = 0

class GenerationOrchestrator:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.gpu_scheduler = GPUScheduler()
        
    async def submit_generation(self, request: GenerationRequest) -> str:
        """Submit generation job and return job_id"""
        
        # 1. Validate user credits
        await self._validate_credits(request.user_id)
        
        # 2. Store job metadata
        await self._store_job_metadata(request)
        
        # 3. Deduct credits
        await self._deduct_credits(request.user_id, request.priority)
        
        # 4. Schedule job based on priority and GPU availability
        gpu_allocation = await self.gpu_scheduler.allocate_gpu(
            user_tier=await self._get_user_tier(request.user_id),
            job_priority=request.priority
        )
        
        if gpu_allocation:
            # Immediate processing
            await self._start_processing(request, gpu_allocation)
        else:
            # Queue for later processing
            await self._queue_job(request)
        
        return request.job_id
    
    async def _start_processing(self, request: GenerationRequest, 
                               gpu_allocation: Dict[str, Any]):
        """Start actual video generation"""
        
        # Update job status
        await self._update_job_status(request.job_id, 'processing', 0)
        
        # Submit to inference service
        inference_task = celery_app.send_task(
            'inference.generate_video',
            args=[
                request.job_id,
                request.image_url,
                request.prompt,
                gpu_allocation['node_id'],
                gpu_allocation['gpu_id']
            ],
            queue=f"gpu_{gpu_allocation['gpu_type']}",
            routing_key=f"gpu.{gpu_allocation['node_id']}"
        )
        
        # Store task info for tracking
        await self._store_task_info(request.job_id, inference_task.id)
    
    async def _queue_job(self, request: GenerationRequest):
        """Queue job for later processing"""
        
        priority_score = self._calculate_priority_score(request)
        
        # Add to Redis sorted set (priority queue)
        redis_client.zadd(
            'generation_queue',
            {request.job_id: priority_score}
        )
        
        await self._update_job_status(request.job_id, 'queued', 0)
        
        # Estimate wait time
        queue_position = redis_client.zrank('generation_queue', request.job_id)
        eta = self._calculate_eta(queue_position, request.priority)
        
        # Send ETA to user
        await self._notify_user_eta(request.user_id, request.job_id, eta)

# Celery task for inference
@celery_app.task(bind=True)
def generate_video_task(self, job_id, image_url, prompt, node_id, gpu_id):
    """Main video generation task"""
    
    try:
        # Update progress callback
        def progress_callback(step, total_steps):
            progress = int((step / total_steps) * 100)
            self.update_state(
                state='PROGRESS',
                meta={'current': step, 'total': total_steps, 'progress': progress}
            )
        
        # Initialize inference service
        inference_service = InferenceService(node_id, gpu_id)
        
        # Download input image
        input_image_path = inference_service.download_image(image_url)
        
        # Generate video
        output_video_path = inference_service.generate_video(
            input_image_path, 
            prompt,
            progress_callback=progress_callback
        )
        
        # Upload result to S3
        video_url = inference_service.upload_video(output_video_path, job_id)
        
        # Update job completion
        update_job_completion(job_id, video_url)
        
        return {'video_url': video_url, 'status': 'completed'}
        
    except Exception as exc:
        # Handle failure
        handle_job_failure(job_id, str(exc))
        raise self.retry(exc=exc, countdown=60, max_retries=3)
```

### 6. GPU Scheduler Implementation

#### Technology: Go + Kubernetes API + Custom Logic

```go
// gpu/scheduler.go
package gpu

import (
    "context"
    "fmt"
    "time"
    
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "github.com/prometheus/client_golang/api"
)

type GPUType string

const (
    GPU_A100 GPUType = "a100"
    GPU_V100 GPUType = "v100" 
    GPU_T4   GPUType = "t4"
)

type GPUAllocation struct {
    NodeID    string    `json:"node_id"`
    GPUID     string    `json:"gpu_id"`
    GPUType   GPUType   `json:"gpu_type"`
    AllocatedAt time.Time `json:"allocated_at"`
}

type GPUScheduler struct {
    k8sClient   kubernetes.Interface
    promClient  api.Client
    nodeStates  map[string]*NodeState
}

type NodeState struct {
    NodeID       string
    GPUType      GPUType
    TotalGPUs    int
    AvailableGPUs int
    RunningJobs  []string
    LastUpdate   time.Time
}

func NewGPUScheduler(k8sClient kubernetes.Interface, promClient api.Client) *GPUScheduler {
    scheduler := &GPUScheduler{
        k8sClient:  k8sClient,
        promClient: promClient,
        nodeStates: make(map[string]*NodeState),
    }
    
    // Start monitoring goroutine
    go scheduler.monitorNodes()
    go scheduler.scheduleQueuedJobs()
    
    return scheduler
}

func (s *GPUScheduler) AllocateGPU(userTier string, jobPriority string) (*GPUAllocation, error) {
    targetGPUType := s.getTargetGPUType(userTier)
    
    // Find best available node
    bestNode := s.findBestNode(targetGPUType, jobPriority)
    if bestNode == nil {
        return nil, fmt.Errorf("no available GPU of type %s", targetGPUType)
    }
    
    // Allocate GPU on the node
    allocation, err := s.allocateGPUOnNode(bestNode, targetGPUType)
    if err != nil {
        return nil, err
    }
    
    // Update node state
    s.updateNodeAllocation(bestNode.NodeID, allocation.GPUID)
    
    return allocation, nil
}

func (s *GPUScheduler) getTargetGPUType(userTier string) GPUType {
    switch userTier {
    case "premium":
        return GPU_A100
    case "standard":
        return GPU_V100
    default:
        return GPU_T4
    }
}

func (s *GPUScheduler) findBestNode(gpuType GPUType, priority string) *NodeState {
    var candidates []*NodeState
    
    // Filter nodes by GPU type and availability
    for _, node := range s.nodeStates {
        if node.GPUType == gpuType && node.AvailableGPUs > 0 {
            candidates = append(candidates, node)
        }
    }
    
    if len(candidates) == 0 {
        return nil
    }
    
    // Sort by load and prefer less loaded nodes
    sort.Slice(candidates, func(i, j int) bool {
        loadI := float64(len(candidates[i].RunningJobs)) / float64(candidates[i].TotalGPUs)
        loadJ := float64(len(candidates[j].RunningJobs)) / float64(candidates[j].TotalGPUs)
        return loadI < loadJ
    })
    
    return candidates[0]
}

func (s *GPUScheduler) monitorNodes() {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            s.updateNodeStates()
        }
    }
}

func (s *GPUScheduler) updateNodeStates() {
    // Get nodes from Kubernetes
    nodes, err := s.k8sClient.CoreV1().Nodes().List(context.TODO(), metav1.ListOptions{
        LabelSelector: "node-type=gpu-worker",
    })
    
    if err != nil {
        log.Printf("Error fetching nodes: %v", err)
        return
    }
    
    for _, node := range nodes.Items {
        nodeState := s.getNodeGPUState(node.Name)
        s.nodeStates[node.Name] = nodeState
    }
}

func (s *GPUScheduler) getNodeGPUState(nodeName string) *NodeState {
    // Query Prometheus for GPU utilization
    query := fmt.Sprintf(`DCGM_FI_DEV_GPU_UTIL{instance=~"%s.*"}`, nodeName)
    
    result, _, err := s.promClient.Query(context.Background(), query, time.Now())
    if err != nil {
        log.Printf("Error querying Prometheus: %v", err)
        return nil
    }
    
    // Parse results and determine available GPUs
    // Implementation details for parsing Prometheus metrics
    
    return &NodeState{
        NodeID:       nodeName,
        GPUType:      s.detectGPUType(nodeName),
        TotalGPUs:    s.getTotalGPUs(nodeName),
        AvailableGPUs: s.calculateAvailableGPUs(result),
        RunningJobs:  s.getRunningJobs(nodeName),
        LastUpdate:   time.Now(),
    }
}
```

### 7. Model Inference Service Implementation

#### Technology: Python + PyTorch + TensorRT + NVIDIA Triton

```python
# inference/service.py
import torch
import numpy as np
from diffusers import StableVideoDiffusionPipeline
import cv2
from PIL import Image
import tritonclient.http as httpclient
import asyncio

class InferenceService:
    def __init__(self, node_id: str, gpu_id: str):
        self.node_id = node_id
        self.gpu_id = gpu_id
        self.device = f"cuda:{gpu_id}"
        
        # Initialize model pipeline
        self.pipeline = self._load_model()
        self.triton_client = self._init_triton_client()
        
    def _load_model(self):
        """Load and optimize the video generation model"""
        
        pipeline = StableVideoDiffusionPipeline.from_pretrained(
            "stabilityai/stable-video-diffusion-img2vid",
            torch_dtype=torch.float16,
            variant="fp16"
        )
        
        pipeline = pipeline.to(self.device)
        
        # Enable memory efficient attention
        pipeline.enable_attention_slicing()
        pipeline.enable_vae_slicing()
        
        # Compile model for better performance (PyTorch 2.0+)
        if hasattr(torch, 'compile'):
            pipeline.unet = torch.compile(pipeline.unet, mode="reduce-overhead")
        
        return pipeline
    
    def _init_triton_client(self):
        """Initialize Triton Inference Server client for ensemble models"""
        try:
            triton_url = f"http://triton-{self.node_id}:8000"
            client = httpclient.InferenceServerClient(url=triton_url)
            return client
        except Exception as e:
            print(f"Failed to initialize Triton client: {e}")
            return None
    
    async def generate_video(self, image_path: str, prompt: str, 
                           progress_callback=None) -> str:
        """Generate video from image and prompt"""
        
        try:
            # Load and preprocess image
            image = Image.open(image_path)
            image = self._preprocess_image(image)
            
            # Progress update: 10%
            if progress_callback:
                progress_callback(1, 10)
            
            # Generate video frames
            frames = await self._generate_frames(image, prompt, progress_callback)
            
            # Progress update: 80%
            if progress_callback:
                progress_callback(8, 10)
            
            # Post-process and create video
            video_path = self._create_video_from_frames(frames)
            
            # Progress update: 100%
            if progress_callback:
                progress_callback(10, 10)
            
            return video_path
            
        except Exception as e:
            raise InferenceError(f"Video generation failed: {str(e)}")
    
    def _preprocess_image(self, image: Image) -> Image:
        """Preprocess input image for optimal generation"""
        
        # Resize to optimal dimensions (maintains aspect ratio)
        target_size = (1024, 576)  # 16:9 aspect ratio
        image = image.resize(target_size, Image.LANCZOS)
        
        # Enhance image if needed
        image = self._enhance_image_quality(image)
        
        return image
    
    async def _generate_frames(self, image: Image, prompt: str, 
                              progress_callback=None) -> list:
        """Generate video frames using the diffusion model"""
        
        # Configure generation parameters
        generator = torch.manual_seed(42)  # For reproducible results
        
        with torch.no_grad():
            frames = self.pipeline(
                image=image,
                prompt=prompt,
                num_frames=25,  # 1 second at 25fps
                num_inference_steps=20,
                guidance_scale=7.5,
                generator=generator,
                decode_chunk_size=8,  # Memory optimization
                callback=self._create_progress_callback(progress_callback, 1, 7)
            ).frames[0]
        
        return frames
    
    def _create_video_from_frames(self, frames: list) -> str:
        """Convert frames to video file"""
        
        output_path = f"/tmp/output_{self.node_id}_{int(time.time())}.mp4"
        
        # Video configuration
        fourcc = cv2.VideoWriter_fourcc(*'mp4v')
        fps = 25
        frame_size = (frames[0].width, frames[0].height)
        
        out = cv2.VideoWriter(output_path, fourcc, fps, frame_size)
        
        for frame in frames:
            # Convert PIL to OpenCV format
            frame_array = np.array(frame)
            frame_bgr = cv2.cvtColor(frame_array, cv2.COLOR_RGB2BGR)
            out.write(frame_bgr)
        
        out.release()
        
        # Optimize video for web streaming
        optimized_path = self._optimize_video(output_path)
        
        return optimized_path
    
    def _optimize_video(self, input_path: str) -> str:
        """Optimize video for web delivery"""
        
        output_path = input_path.replace('.mp4', '_optimized.mp4')
        
        # Use FFmpeg for optimization
        cmd = [
            'ffmpeg', '-i', input_path,
            '-c:v', 'libx264',
            '-preset', 'fast',
            '-crf', '23',
            '-c:a', 'aac',
            '-b:a', '128k',
            '-movflags', '+faststart',  # Enable progressive download
            '-y', output_path
        ]
        
        subprocess.run(cmd, check=True)
        
        # Remove original file
        os.remove(input_path)
        
        return output_path
    
    def _create_progress_callback(self, user_callback, start_step, end_step):
        """Create progress callback for diffusion process"""
        
        def callback(step, timestep, latents):
            if user_callback:
                progress_step = start_step + (step / 20) * (end_step - start_step)
                user_callback(int(progress_step), 10)
        
        return callback

# Model serving with batch processing
class BatchInferenceService:
    def __init__(self, max_batch_size: int = 4):
        self.max_batch_size = max_batch_size
        self.pending_requests = []
        self.processing_lock = asyncio.Lock()
        
    async def add_request(self, request: InferenceRequest) -> str:
        """Add request to batch processing queue"""
        
        async with self.processing_lock:
            self.pending_requests.append(request)
            
            # Process batch if full or timeout reached
            if (len(self.pending_requests) >= self.max_batch_size or 
                self._should_process_batch()):
                await self._process_batch()
        
        return request.job_id
    
    async def _process_batch(self):
        """Process a batch of requests together"""
        
        if not self.pending_requests:
            return
        
        batch = self.pending_requests[:self.max_batch_size]
        self.pending_requests = self.pending_requests[self.max_batch_size:]
        
        # Process batch in parallel
        tasks = []
        for request in batch:
            task = asyncio.create_task(
                self._process_single_request(request)
            )
            tasks.append(task)
        
        await asyncio.gather(*tasks)
```

This completes the detailed Low-Level Design. The implementation covers all major components with production-ready code examples, error handling, performance optimizations, and scalability considerations.

Would you like me to continue with the API contracts, database schemas, and deployment configurations?

<function_calls>
<invoke name="create_file">
<parameter name="content"># API Contracts & Specifications

## REST API Endpoints

### Base URL
```
Production: https://api.videogen.ai/v1
Staging: https://api-staging.videogen.ai/v1
```

### Authentication
All endpoints require Bearer token authentication except public endpoints.

```http
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

---

## 1. Authentication Endpoints

### POST /auth/login
**Description:** User login with email/password

**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword123"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "rt_abc123...",
    "expires_in": 86400,
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "user@example.com",
      "tier": "premium",
      "credits": 150
    }
  }
}
```

**Error Responses:**
```json
// 401 Unauthorized
{
  "success": false,
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password"
  }
}

// 429 Rate Limited
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many login attempts",
    "retry_after": 300
  }
}
```

### POST /auth/refresh
**Description:** Refresh access token

**Request:**
```json
{
  "refresh_token": "rt_abc123..."
}
```

### POST /auth/logout
**Description:** Invalidate token and logout

**Request:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## 2. User Management Endpoints

### GET /users/profile
**Description:** Get current user profile

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "tier": "premium",
    "credits": 150,
    "total_generations": 45,
    "created_at": "2024-01-15T10:30:00Z",
    "subscription": {
      "plan": "premium",
      "status": "active",
      "next_billing_date": "2024-03-15T00:00:00Z"
    }
  }
}
```

### GET /users/credits
**Description:** Get current credit balance and usage

**Response:**
```json
{
  "success": true,
  "data": {
    "current_balance": 150,
    "monthly_allowance": 500,
    "used_this_month": 350,
    "credit_history": [
      {
        "id": "tx_001",
        "amount": -5,
        "type": "generation",
        "description": "Video generation - Premium quality",
        "created_at": "2024-02-20T14:30:00Z",
        "generation_id": "gen_abc123"
      }
    ]
  }
}
```

---

## 3. Generation Endpoints

### POST /generations/upload-image
**Description:** Upload image and get pre-signed URL

**Request (Multipart Form Data):**
```
Content-Type: multipart/form-data

image: [binary file data]
filename: "image.jpg"
```

**Response:**
```json
{
  "success": true,
  "data": {
    "upload_id": "upload_abc123",
    "image_url": "https://storage.videogen.ai/inputs/user123/upload_abc123/image.jpg",
    "thumbnail_url": "https://storage.videogen.ai/thumbnails/upload_abc123/thumb.jpg",
    "metadata": {
      "width": 1024,
      "height": 576,
      "format": "JPEG",
      "file_size": 245760
    }
  }
}
```

### POST /generations/create
**Description:** Create new video generation job

**Request:**
```json
{
  "image_url": "https://storage.videogen.ai/inputs/user123/upload_abc123/image.jpg",
  "prompt": "A beautiful sunset over the ocean with gentle waves",
  "language": "en",
  "settings": {
    "duration": 10,
    "quality": "high",
    "fps": 25,
    "aspect_ratio": "16:9"
  },
  "is_public": false,
  "enhance_prompt": true
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "status": "queued",
    "estimated_wait_time": 45,
    "credits_deducted": 5,
    "remaining_credits": 145,
    "websocket_url": "wss://api.videogen.ai/v1/generations/gen_550e8400e29b41d4a716446655440000/ws",
    "created_at": "2024-02-25T10:30:00Z"
  }
}
```

### GET /generations/{job_id}
**Description:** Get generation job status and result

**Response:**
```json
{
  "success": true,
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "status": "completed",
    "progress": 100,
    "created_at": "2024-02-25T10:30:00Z",
    "completed_at": "2024-02-25T10:31:45Z",
    "input": {
      "image_url": "https://storage.videogen.ai/inputs/user123/upload_abc123/image.jpg",
      "original_prompt": "A beautiful sunset over the ocean",
      "enhanced_prompt": "A cinematic beautiful sunset over the ocean with gentle waves, golden hour lighting, camera slowly panning across the horizon, peaceful and serene atmosphere, 4K quality",
      "language": "en"
    },
    "output": {
      "video_url": "https://cdn.videogen.ai/outputs/gen_550e8400e29b41d4a716446655440000/video.mp4",
      "thumbnail_url": "https://cdn.videogen.ai/outputs/gen_550e8400e29b41d4a716446655440000/thumbnail.jpg",
      "duration": 10.5,
      "file_size": 15728640,
      "resolution": "1024x576"
    },
    "metadata": {
      "model_version": "svd-v1.2",
      "inference_time": 87.3,
      "gpu_type": "A100"
    }
  }
}
```

**Status Values:**
- `pending` - Job created, waiting for processing
- `queued` - Job in queue, waiting for GPU
- `processing` - Currently generating video
- `completed` - Successfully completed
- `failed` - Generation failed
- `cancelled` - User cancelled job

### GET /generations
**Description:** List user's generations with pagination

**Query Parameters:**
- `page` (int): Page number (default: 1)
- `limit` (int): Items per page (default: 20, max: 100)
- `status` (string): Filter by status
- `start_date` (ISO date): Filter from date
- `end_date` (ISO date): Filter to date

**Response:**
```json
{
  "success": true,
  "data": {
    "generations": [
      {
        "job_id": "gen_550e8400e29b41d4a716446655440000",
        "status": "completed",
        "original_prompt": "A beautiful sunset over the ocean",
        "thumbnail_url": "https://cdn.videogen.ai/thumbnails/gen_550e8400e29b41d4a716446655440000/thumb.jpg",
        "created_at": "2024-02-25T10:30:00Z",
        "is_public": false
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 45,
      "total_pages": 3,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

### DELETE /generations/{job_id}
**Description:** Cancel or delete generation

**Response:**
```json
{
  "success": true,
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "status": "cancelled",
    "credits_refunded": 5
  }
}
```

---

## 4. WebSocket API for Real-time Updates

### Connection URL
```
wss://api.videogen.ai/v1/generations/{job_id}/ws?token={jwt_token}
```

### Message Types

#### Progress Update
```json
{
  "type": "progress",
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "progress": 45,
    "status": "processing",
    "current_step": "generating_frames",
    "eta_seconds": 42
  }
}
```

#### Status Change
```json
{
  "type": "status_change",
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "old_status": "queued",
    "new_status": "processing",
    "timestamp": "2024-02-25T10:31:00Z"
  }
}
```

#### Job Completed
```json
{
  "type": "completed",
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "status": "completed",
    "video_url": "https://cdn.videogen.ai/outputs/gen_550e8400e29b41d4a716446655440000/video.mp4",
    "thumbnail_url": "https://cdn.videogen.ai/outputs/gen_550e8400e29b41d4a716446655440000/thumbnail.jpg",
    "metadata": {
      "duration": 10.5,
      "file_size": 15728640
    }
  }
}
```

#### Error
```json
{
  "type": "error",
  "data": {
    "job_id": "gen_550e8400e29b41d4a716446655440000",
    "error_code": "INFERENCE_FAILED",
    "error_message": "Model inference failed due to GPU memory error",
    "is_retryable": true,
    "credits_refunded": 5
  }
}
```

---

## 5. Translation & Localization Endpoints

### POST /translate/prompt
**Description:** Translate prompt to target language

**Request:**
```json
{
  "text": "A beautiful sunset over the ocean",
  "source_language": "auto",
  "target_language": "hi",
  "context": "video_generation"
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "original_text": "A beautiful sunset over the ocean",
    "translated_text": "समुद्र के ऊपर एक सुंदर सूर्यास्त",
    "source_language": "en",
    "target_language": "hi",
    "confidence": 0.95
  }
}
```

### GET /translate/languages
**Description:** Get supported languages

**Response:**
```json
{
  "success": true,
  "data": {
    "supported_languages": [
      {"code": "en", "name": "English", "native_name": "English"},
      {"code": "hi", "name": "Hindi", "native_name": "हिन्दी"},
      {"code": "zh", "name": "Chinese", "native_name": "中文"},
      {"code": "es", "name": "Spanish", "native_name": "Español"},
      {"code": "fr", "name": "French", "native_name": "Français"}
    ]
  }
}
```

---

## 6. Public/Discovery Endpoints

### GET /public/generations
**Description:** Get public generations feed

**Query Parameters:**
- `page` (int): Page number
- `limit` (int): Items per page
- `sort` (string): Sort by 'recent', 'popular', 'trending'
- `category` (string): Filter by category

**Response:**
```json
{
  "success": true,
  "data": {
    "generations": [
      {
        "id": "gen_abc123",
        "thumbnail_url": "https://cdn.videogen.ai/thumbnails/gen_abc123/thumb.jpg",
        "video_url": "https://cdn.videogen.ai/outputs/gen_abc123/video.mp4",
        "prompt": "A majestic eagle soaring through mountain peaks",
        "likes_count": 245,
        "views_count": 1580,
        "created_at": "2024-02-24T15:30:00Z",
        "user": {
          "username": "artist123",
          "avatar_url": "https://cdn.videogen.ai/avatars/artist123.jpg"
        }
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1250,
      "has_next": true
    }
  }
}
```

---

## 7. Admin/Monitoring Endpoints

### GET /admin/stats
**Description:** Get system statistics (Admin only)

**Response:**
```json
{
  "success": true,
  "data": {
    "active_users": 2340,
    "generations_today": 8450,
    "gpu_utilization": {
      "a100": {"total": 8, "active": 6, "utilization": 75.0},
      "v100": {"total": 16, "active": 12, "utilization": 82.5},
      "t4": {"total": 32, "active": 18, "utilization": 56.3}
    },
    "queue_stats": {
      "pending": 23,
      "processing": 15,
      "average_wait_time": 42
    }
  }
}
```

---

## Error Handling

### Standard Error Response
```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable error message",
    "details": {
      "field": "specific field that caused error",
      "value": "invalid value"
    },
    "request_id": "req_abc123",
    "timestamp": "2024-02-25T10:30:00Z"
  }
}
```

### Common Error Codes
- `INVALID_REQUEST` - Malformed request body
- `UNAUTHORIZED` - Invalid or missing authentication
- `FORBIDDEN` - Insufficient permissions
- `RESOURCE_NOT_FOUND` - Requested resource doesn't exist
- `RATE_LIMITED` - Rate limit exceeded
- `INSUFFICIENT_CREDITS` - Not enough credits for operation
- `QUOTA_EXCEEDED` - Monthly quota exceeded
- `VALIDATION_ERROR` - Request validation failed
- `INTERNAL_ERROR` - Internal server error
- `SERVICE_UNAVAILABLE` - Service temporarily unavailable

### Rate Limiting Headers
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640995200
X-RateLimit-Retry-After: 3600
```

---

## API Versioning

### Version Header
```http
API-Version: v1
Accept: application/json
```

### Backward Compatibility
- Minor version changes maintain backward compatibility
- Major version changes may introduce breaking changes
- Deprecated endpoints return `X-Deprecated` header
- Minimum 6 months notice before version retirement

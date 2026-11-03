# Image-to-Video AI Platform: System Design

A system design for a multilingual image-to-video generation service: a user uploads an
image and a prompt, and the platform returns a generated video clip.

**This repository is design documentation, not an implementation.** It covers the
architecture, component design, API contracts, data model, infrastructure and cost model,
and a phased path from an MVP to production scale.

## Design targets

| | MVP | Production |
|---|---|---|
| Daily active users | 1k | 100k |
| Concurrent generations | 100 | 10k |
| Generation latency | | P95 under 60 s |
| Deployment | single region | multi-region |

## Architecture

```
Clients        Web (React) · Mobile (React Native) · Partner APIs
Edge           CDN · API gateway · load balancer
Application    Auth · Prompt · Translation · Content moderation · Credits · Generation orchestrator
Inference      GPU scheduler · model registry · inference workers · queue · quality checker
Data           PostgreSQL · Redis · object storage · vector DB · time-series DB · search
```

The request path: the image goes to object storage through a pre-signed URL; the prompt is
validated, translated to English if needed, enhanced by an LLM and safety-checked; credits
are checked; the orchestrator queues the job (`PENDING → QUEUED → PROCESSING →
COMPLETED/FAILED`); a GPU worker runs inference (Stable Video Diffusion as the primary
model); progress streams back over WebSocket/SSE.

## Key design problems addressed

- **GPU scheduling and cost.** Tiered GPU pools mapped to user tiers, a 70/30 spot to
  on-demand mix with preemption handling, request batching, and model optimisation
  (quantisation, TensorRT).
- **Multilingual prompts.** Language detection, translation to a canonical English prompt
  for inference, and the original language kept in metadata for the response.
- **Safety.** Moderation on prompts and inputs before any GPU time is spent.
- **Unit economics.** Cost per generation, credit-based pricing, and a three-year model,
  so the architecture can be judged on cost as well as scale.

## Documents

| Document | Contents |
|---|---|
| [High-level design](hld/system-architecture.md) | Layers, core services, data flow, storage, GPU scheduling, deployment, observability |
| [Component design](lld/component-design.md) | Service-level design of each component |
| [API contracts](api/api-contracts.md) | Endpoints, request/response shapes, errors |
| [Database design](schemas/database-design.md) | Schema, indexes, caching |
| [Infrastructure](configs/infrastructure.md) | Kubernetes, GPU node pools, service mesh, multi-region |
| [Cost optimisation](lld/cost-optimization-strategy.md) | GPU, storage and CDN cost, pricing, financial model |
| [MVP to production roadmap](lld/mvp-production-roadmap.md) | Three phases, timeline, risks, success metrics |

# Backend Developer

## **Evaluation Criteria**

* System design clarity (services, boundaries, async workflows)
* Schema design quality (entities, constraints, indexing, migrations order)
* Database + storage selection (why, tradeoffs, scaling path)
* Security (authN/authZ, secrets, data protection, safe downloads)
* Reliability (retries, idempotency, job state machine, observability)
* Cost + scalability thinking (MVP → v1)

---

## **Problem 1: Video-to-Notes Platform (Architecture + Schema)**

**Goal:** Upload video → async processing → outputs: transcript, **Summary.md**, highlights (timestamps), screenshot/clip references. [READ MORE ABOUT THE PROJECT](./Video-summary-platform.md)

**My solution  includes**

* **System design:** API + worker(s) + storage + external AI/transcription boundaries
* **DB choice:** Postgres vs other (short justification)
* **Schema:** tables only (User, VideoAsset, Job, JobEvent, Artifact, Highlight)
* **Key constraints/indexes:** (unique keys, foreign keys, indexes you’d add)
* **Job lifecycle:** queued → processing → success/failed (+ retry/idempotency rule)
* **Storage layout:** how files/artifacts are stored + safe download strategy

**My Solution for problem 1:**
## System Design
API Service (Spring Boot) handles video upload, job creation, and artifact retrieval.  
Worker Service processes jobs asynchronously using a queue (Redis/RabbitMQ).  
PostgreSQL stores metadata.  
Object storage (S3-compatible) stores videos and generated outputs.  
External AI services handle transcription and summarization.

Supports batch folder processing, chunked upload, and streaming for large files (200MB+, 3–4 hours).

Flow:
1. Upload video → create VideoAsset + Job (QUEUED)
2. Worker picks job → PROCESSING → calls AI
3. Stores transcript, summary, highlights → SUCCESS/FAILED

## Database Choice
PostgreSQL for strong relational modeling, indexing on job status, and JSONB for AI metadata.

## Schema (Tables)
User(id, email, created_at)  
VideoAsset(id, user_id, storage_path, duration, created_at)  
Job(id, video_id, status, retry_count, idempotency_key, created_at)  
JobEvent(id, job_id, status, message, created_at)  
Artifact(id, job_id, type, storage_path, created_at)  
Highlight(id, job_id, timestamp_start, timestamp_end, text)

## Constraints & Indexes
FK: video_asset.user_id → user.id  
FK: job.video_id → video_asset.id  
Index on job(status)  
Unique job.idempotency_key  
Index on artifact.job_id

## Job Lifecycle
QUEUED → PROCESSING → SUCCESS | FAILED  
Max 3 retries with idempotency check.

## Storage Strategy
/tenant/{userId}/videos/{videoId}.mp4  
/tenant/{userId}/jobs/{jobId}/summary.md  

Secure downloads via pre-signed URLs after auth validation.

## Security
User ownership validation and private object storage.

## Reliability
Job state machine, JobEvent logging, retries with backoff.

## Cost & Scalability
MVP single worker → v1 horizontal workers + S3 + queue partitioning.


---

## **Problem 2: LinkedIn Automation Platform (Backend Architecture)**

**Goal:** Connect LinkedIn → store persona → generate drafts (handled by GenAI team) → approve → schedule → auto-post + audit logs. [READ MORE ABOUT THE PROJECT](./linkedin-automation.md)

**My solution  includes**

* **System design:** OAuth flow, token storage/refresh, scheduler/worker design
* **Schema:** User, LinkedInAccount, Persona, Draft, Schedule, PostAttempt/PostLog
* **Security:** encrypt tokens at rest, least-privilege scopes, access control
* **Reliability:** retry posting, dedupe to prevent double-post, rate limiting
* **Prompt/config storage proposal:** how backend stores “prompt versions” or “config packs” provided by GenAI team (DB vs repo vs hybrid, versioning + rollback)

**My Solution for problem 2:**

## System Design
OAuth connection to LinkedIn → store encrypted tokens.  
Persona stored per user → Draft generation → Schedule → Worker auto-post → PostLog for audit.

## Schema
User(id, email)  
LinkedInAccount(id, user_id, access_token_enc, refresh_token_enc, expires_at)  
Persona(id, user_id, tone, topics)  
Draft(id, user_id, content, status)  
Schedule(id, draft_id, scheduled_time, status)  
PostLog(id, schedule_id, status, response, created_at)

## Security
AES-256 encrypted tokens, least-privilege scopes, per-user access control.

## Reliability
Dedupe key = hash(content + scheduled_time)  
Retry with exponential backoff  
Rate limiting per user  
Automatic token refresh via worker

## Prompt Storage
PromptConfig(id, version, content, created_at) for versioning and rollback.


---

## **Problem 3: DOCX Template → Bulk DOCX/PDF Generator (Backend + Storage)**

**Goal:** Upload DOCX template → detect fields → single fill export → bulk fill via CSV/Sheet → ZIP + per-row report. [READ MORE ABOUT THE PROJECT](./docs-template-output-generation.md)

**My solution includes**

* **System design:** template ingestion, field extraction service, bulk job worker, export service
* **Schema:** Template, TemplateVersion, TemplateField, BulkRun, BulkRow, Artifact, JobEvent
* **Bulk storage strategy:** where CSV input + generated docs + ZIP live, cleanup policy
* **Reliability:** partial success handling, per-row status, retries, resumable bulk run
* **Security:** template isolation per tenant/user, safe downloads, anti-path traversal

**My Solution for problem 3:**

## System Design
Template upload → extract fields → store TemplateVersion + TemplateFields.  
CSV upload → BulkRun → Worker processes rows → generate DOCX/PDF → ZIP export.

## Schema
Template(id, user_id, name)  
TemplateVersion(id, template_id, storage_path, created_at)  
TemplateField(id, template_version_id, field_name)  
BulkRun(id, template_version_id, status, total_rows, processed_rows)  
BulkRow(id, bulk_run_id, status, output_artifact_id)  
Artifact(id, storage_path, type)  
JobEvent(id, bulk_run_id, message)

## Storage Strategy
CSV input, generated docs, and ZIP stored in object storage.  
Temporary files deleted after ZIP creation.

## Reliability
Per-row status, resumable using processed_rows, partial success supported.

## Security
Tenant isolation via user_id, signed URLs, path traversal prevention.


---

## **Problem 4: Character-Based Video Series Generator (Backend Architecture)**

**Goal:** Define characters once (image + traits + relationships). For each episode story → output episode package (script/scenes/assets plan/render plan), optionally render. [READ MORE ABOUT THE PROJECT](./char-based-video-generation.md)

**My solution includes**

* **System design:** episodic pipeline as jobs, asset management, consistency strategy storage
* **Schema:** Character, Relationship, Episode, Scene, Asset, RenderJob, Artifact
* **Consistency data:** what gets persisted to keep character continuity across episodes
* **Storage:** images/audio/video assets, versioning, dedupe strategy
* **Security + cost controls:** quotas, rate limits, large asset constraints

**My Solution for problem 4:**

## System Design
Characters defined once with traits and assets.  
Episode pipeline: Episode → scenes → assets → render jobs processed asynchronously.

## Schema
Character(id, user_id, name, traits, voice_id, appearance_ref)  
Relationship(id, character_a_id, character_b_id, type)  
Episode(id, user_id, title, character_snapshot_version)  
Scene(id, episode_id, script_text)  
Asset(id, scene_id, type, storage_path, hash)  
RenderJob(id, episode_id, status, retry_count)  
Artifact(id, render_job_id, storage_path)

## Consistency
Character snapshot version stored per episode to maintain continuity.

## Storage
Versioned assets with hash-based deduplication and per-user quotas.

## Security & Cost
Rate limits on render jobs, file size limits, per-user storage quotas.


## Problem 5: Cross-Cutting

Answer briefly for the whole platform:

1. **Multi-tenancy:** user-level vs workspace-level, how you model it in schema
2. **AuthZ model:** RBAC or simple ownership rules, and where enforced (API + DB)
3. **Observability:** what you log for jobs + correlation ids + minimal metrics
4. **Data retention:** what to delete and when (inputs, artifacts, logs)
5. **Secrets & compliance:** token encryption, key management approach, PII handling

**My Answer for problem 5:**

## Multi-Tenancy
User-level tenancy with user_id present in all tables.

## AuthZ Model
RBAC (USER, ADMIN) enforced at API layer and query filters.

## Observability
Logs include job_id, correlation_id, and status transitions.  
Metrics: job latency, failure rate, queue depth.

## Data Retention
Raw inputs: 30 days  
Logs: 14 days  
Artifacts: user-controlled deletion.

## Secrets & Compliance
Tokens encrypted with AES-256.  
Keys stored in environment/secret manager.  
Minimal PII storage.


# AI Social Media Automation Agent - Design:

## Architecture Overview:

The system implements a serverless microservices architecture on AWS, using event-driven patterns for scalability and fault tolerance. The design separates concerns into presentation, API, application, data, and integration layers.

### Architecture Principles:

- **Scalability**: Serverless compute with automatic scaling, stateless Lambda functions, unlimited DynamoDB storage.
- **Reliability**: Multi-AZ redundancy, circuit breakers, dead letter queues, graceful degradation.
- **Security**: Defense in depth, AWS KMS encryption, VPC isolation, least privilege IAM, API Gateway auth.
- **Cost Optimization**: On-demand pricing, caching strategy, S3 lifecycle policies, reserved capacity for baseline.

## System Components:

### 1. User Management Service:

Handles authentication, authorization, and social account connections.

**Components:**
- Authentication: JWT with RS256, 1-hour access tokens, 30-day refresh tokens, OAuth 2.0 social login.
- Authorization: RBAC with roles (user, premium_user, admin, super_admin), resource-level permissions, Redis caching.
- Account Connection: OAuth 2.0 flows, envelope encryption for tokens, automatic refresh, scope validation.

**Data Models:**
```
User:
  user_id (PK), email (GSI), password_hash, role, status, preferences, timestamps

SocialAccount:
  account_id (PK), user_id (GSI), platform, platform_user_id, 
  encrypted tokens, token_expiry, scopes, status, timestamps
```

### 2. Content Generation Service:

Orchestrates AI-powered content creation with context loading and platform formatting.

**Components:**
- Content Orchestrator: Workflow coordination, context loading, model selection (cloud: Claude Sonnet 4.5, GPT-5.X, Gemini 2.5/3 Pro; local: Qwen-3, GLM-4, Llama), retry management.
- LLM Client: Bedrock API abstraction, prompt templates, parameter management, response parsing, caching, local model inference support.
- Context Loader: Profile retrieval, historical performance queries, trending topics, audience data, multi-level caching.
- Platform Formatter: Platform-specific formatting, character limits, hashtag generation, thread structure, CTA insertion.
- Content Validator: Guideline validation, prohibited word checking, brand voice similarity, link validation.

**Data Models:**
```
ContentDraft:
  draft_id (PK), user_id (GSI), platform, content_type, text_content,
  media_urls, metadata, safety_scores, brand_alignment_score, status, timestamps

ContentTemplate:
  template_id (PK), user_id (GSI), template_type, prompt_template, 
  variables, platform, timestamps
```

### 3. Engagement Automation Service:

Monitors and responds to social media interactions automatically.

**Components:**
- Event Monitor: Webhook listening, API polling, event normalization, deduplication, routing.
- Event Classifier: Type analysis, entity extraction, urgency determination, spam filtering.
- Sentiment Analyzer: Transformer-based sentiment analysis, emotion detection, sarcasm identification.
- Response Generator: LLM-based responses, conversation history, tone application, template usage.
- Engagement Executor: Platform posting, retry logic, rate limiting, audit logging.

**Data Models:**
```
EngagementEvent:
  event_id (PK), user_id (GSI), platform, event_type, sender_id,
  content, sentiment, sentiment_score, urgency_score, timestamps

EngagementAction:
  action_id (PK), event_id (GSI), action_type, response_content,
  status, timestamps
```

### 4. Analytics Service:

Collects, aggregates, and generates insights from social media metrics.

**Components:**
- Metrics Collector: API ingestion, normalization, validation, interpolation, time-series storage.
- Aggregation Engine: Hourly/daily/weekly aggregations, engagement rate calculation, cohort analysis, materialized views.
- Insight Generator: Trend identification, anomaly detection, ML predictions, recommendation generation.
- Report Builder: Scheduled reports, multi-format exports, visualizations, pagination, caching.

**Data Models:**
```
PostMetric:
  metric_id (PK), post_id (GSI), user_id (GSI), timestamp (SK),
  platform, likes, comments, shares, saves, impressions, reach, engagement_rate

UserAnalytics:
  user_id (PK), date (SK), follower_count, follower_growth, total_posts,
  total_engagement, avg_engagement_rate, top_performing_content_type
```

### 5. Scheduling Service:

Manages intelligent post scheduling and publication.

**Components:**
- Scheduler: Priority queue, ML-based optimal time prediction, distribution logic, timezone handling, retry mechanism.
- Queue Manager: FIFO queues, visibility timeout, deduplication, depth monitoring, DLQ handling.
- Publisher: Platform-specific formatting, media upload, success validation, status updates, analytics triggering.

**Data Models:**
```
ScheduledPost:
  schedule_id (PK), user_id (GSI), content_id, scheduled_time,
  timezone, platform, status, retry_count, timestamps
```

### 6. Safety Service:

Ensures content safety, brand alignment, and platform compliance.

**Components:**
- Content Filter: Toxicity detection, hate speech identification, explicit content flagging, PII checking, custom word lists.
- Brand Alignment Scorer: Cosine similarity, tone analysis, keyword validation, multi-dimensional scoring.
- Compliance Checker: Community guidelines validation, copyright checking, disclosure verification, accessibility checks.

**Data Models:**
```
SafetyCheck:
  check_id (PK), content_id (GSI), toxicity_score, hate_speech_score,
  explicit_content_score, brand_alignment_score, compliance_status, flagged_issues, timestamp
```

## Data Architecture:

### Storage Strategy:

**DynamoDB Tables:**
- Users: user_id (PK), email (GSI).
- SocialAccounts: account_id (PK), user_id (GSI).
- ContentDrafts: draft_id (PK), user_id + status (composite GSI).
- ScheduledPosts: schedule_id (PK), scheduled_time (SK).
- PostMetrics: metric_id (PK), timestamp (SK).

**S3 Buckets:**
- user-generated-content: Media files with versioning.
- content-templates: Reusable templates.
- analytics-exports: Report exports.
- model-artifacts: ML model files.
- logs: Application and access logs.

**ElastiCache (Redis):**
- Session cache: 1-hour TTL.
- Content cache: 15-minute TTL.
- Context cache: 30-minute TTL.
- Metrics cache: 5-minute TTL.

### Data Flow:

**Write Path:**
API Gateway → Lambda validation → DynamoDB conditional writes → DynamoDB Streams → EventBridge fan-out → S3 for large objects.

**Read Path:**
API request → ElastiCache check → DynamoDB query (on miss) → Cache result → Return.

**Analytics Path:**
Kinesis Data Streams → Lambda enrichment → DynamoDB + S3 → Glue ETL aggregation → QuickSight/Dashboard queries.

## API Design:

### REST Endpoints:

**Authentication:**
- POST /api/v1/auth/register.
- POST /api/v1/auth/login.
- POST /api/v1/auth/refresh.
- POST /api/v1/auth/logout.

**Social Accounts:**
- GET /api/v1/accounts.
- POST /api/v1/accounts/connect.
- DELETE /api/v1/accounts/{account_id}.
- PUT /api/v1/accounts/{account_id}/refresh.

**Content:**
- POST /api/v1/content/generate.
- GET /api/v1/content/drafts.
- PUT /api/v1/content/drafts/{draft_id}.
- POST /api/v1/content/drafts/{draft_id}/approve.
- POST /api/v1/content/drafts/{draft_id}/schedule.
- DELETE /api/v1/content/drafts/{draft_id}.

**Analytics:**
- GET /api/v1/analytics/overview.
- GET /api/v1/analytics/posts/{post_id}.
- GET /api/v1/analytics/insights.
- POST /api/v1/analytics/export.

**User:**
- GET /api/v1/user/profile.
- PUT /api/v1/user/profile.
- PUT /api/v1/user/preferences.

### WebSocket API:

**Connection:** wss://api.example.com/ws with Bearer token auth.

**Events:**
- content.published.
- metrics.updated.
- engagement.received.

### API Security:

- Bearer token authentication (except auth endpoints).
- RBAC enforcement.
- Rate limiting per user and IP.
- JSON schema validation.
- Input sanitization.
- 10MB request size limit.

## ML Pipeline Design:

### Content Generation Pipeline:

**Prompt Engineering:**
- System prompts for agent behavior.
- User prompts with context.
- Few-shot examples.
- Parameterized templates.
- Prompt versioning for A/B testing.

**Model Selection:**
- Claude Sonnet 4.5: General content, professional tone.
- GPT-5.X: Creative/marketing content, conversational engagement.
- Gemini 2.5/3 Pro: Multi-modal content, visual descriptions.
- Local LLMs (Qwen-3, GLM-4, Llama): Cost-effective, privacy-focused deployments.
- Selection based on content type, user tier, and latency requirements.
- Fallback strategies across cloud and local models.
- Latency-based routing with quality thresholds.

**Post-Processing:**
- JSON response parsing.
- Field extraction and validation.
- Character limit application.
- Hashtag and mention formatting.
- URL validation.

### Recommendation Pipeline:

**Feature Engineering:**
- Temporal features (hour, day, week).
- Engagement features from history.
- Content features via NLP.
- User profile features.
- Interaction features.

**Model Training:**
- Time-series models on 90-day data.
- Cross-validation for hyperparameters.
- Early stopping.
- S3 model registry.
- Semantic versioning.

**Model Serving:**
- SageMaker endpoints.
- Auto-scaling.
- Batch transform for offline predictions.
- Redis caching.
- Performance monitoring.

**Model Monitoring:**
- Prediction accuracy tracking.
- Feature distribution monitoring.
- Performance degradation alerts.
- Drift detection triggers.
- Lineage maintenance.

### Safety Pipeline:

**Toxicity Detection:**
- Perspective API integration.
- Custom domain-specific classifier.
- Model ensembling.
- 0.8 threshold for auto-approval.
- Human review for borderline cases.

**Brand Alignment:**
- Embedding similarity to brand profile.
- Linguistic feature analysis.
- Keyword density scoring.
- Weighted average combination.
- 0.7 alignment threshold.

**Quality Assessment:**
- Grammar checking.
- Factual consistency.
- Coherence evaluation.
- Completeness validation.
- Multi-dimensional scoring.

## Infrastructure Design:

### Compute:

**Lambda Configuration:**
- Content generation: 2GB memory, 5-minute timeout.
- Engagement: 1GB memory, 30-second timeout.
- Analytics: 512MB memory, 30-second timeout.
- Concurrency limits for cost control.

**Container Configuration:**
- SageMaker: GPU instances (A10G, A100) for ML models and local LLM inference.
- ECS: Background jobs unsuitable for Lambda, local LLM serving (vLLM, Ollama).
- ECR: Docker image storage for custom inference containers.
- Health checks and graceful shutdown.
- CPU/memory-based auto-scaling for traditional workloads.
- GPU-based auto-scaling for LLM inference (scale on queue depth and GPU utilization).

### Network:

**VPC Configuration:**
- 3 availability zones.
- Public subnets: NAT gateways, load balancers.
- Private subnets: Lambda, containers.
- Database subnet: Isolated from internet.
- VPC endpoints for cost reduction.

**Security Groups:**
- API Gateway: HTTPS inbound.
- Lambda: Outbound to VPC endpoints.
- Database: Inbound from Lambda only.
- Least privilege rules.
- Documented and versioned.

**Load Balancing:**
- ALB for container traffic.
- CloudFront for static assets.
- API Gateway for rate limiting.
- Health checks.
- Connection draining.

### Storage:

**DynamoDB:**
- On-demand capacity mode.
- Point-in-time recovery.
- Global secondary indexes.
- TTL for automatic expiration.
- Streams for CDC.

**S3:**
- Versioning enabled.
- Lifecycle policies (Glacier after 90 days).
- KMS encryption.
- Cross-region replication.
- Access logging.

**ElastiCache:**
- Redis cluster mode.
- Multi-AZ.
- Sub-60-second failover.
- Reserved nodes for baseline.
- Daily backups.

### Monitoring:

**CloudWatch:**
- Custom business metrics.
- Service-organized log groups.
- Metric alarms → SNS.
- Health dashboards.
- 30-day log retention.

**X-Ray:**
- Lambda function tracing.
- 10% sampling for high-volume endpoints.
- Service map visualization.
- Bottleneck identification.
- CloudWatch integration.

## Security Architecture:

### Identity and Access Management:

**IAM Roles:**
- Lambda execution: Minimal permissions.
- S3: Separate read/write access.
- DynamoDB: Table-scoped.
- Cross-service: Trust policies.
- Quarterly reviews.

**IAM Policies:**
- Least privilege principle.
- Resource-level permissions.
- Condition keys (IP, time).
- Policy versioning.
- Automated validation.

### Data Protection:

**Encryption at Rest:**
- DynamoDB: AWS managed KMS keys.
- S3: Customer managed keys.
- RDS: Storage volume encryption.
- EBS: Default encryption.
- Annual key rotation.

**Encryption in Transit:**
- TLS 1.3 required.
- Certificate pinning (mobile).
- API Gateway HTTPS enforcement.
- VPC endpoint TLS.
- Database connection encryption.

### Application Security:

**API Security:**
- RS256 JWT signing.
- Per-request validation.
- Rate limiting (user + IP).
- Injection prevention.
- CORS restrictions.

**Secrets Management:**
- AWS Secrets Manager storage.
- 90-day automatic rotation.
- IAM role access.
- No logging/exposure.
- Audit trail.

## Scalability Design:

### Horizontal Scaling:

**Compute:**
- Lambda: Automatic scaling.
- SageMaker: Metric-based scaling.
- ECS: Target tracking.
- Per-service policies.
- Scale-in protection.

**Data:**
- DynamoDB: Automatic partitioning.
- S3: Unlimited scaling.
- ElastiCache: Cluster mode.
- Read replicas.
- Sharding strategy.

### Vertical Scaling:

- Lambda: Memory tuning per function.
- RDS: Instance type upgrades.
- ElastiCache: Node type selection.
- Monitoring-based right-sizing.
- Scheduled scaling for patterns.

### Geographic Scaling:

- Multi-region architecture.
- CloudFront edge locations.
- Database replication.
- Active-active configuration.
- Data residency compliance.

## Reliability Design:

### Fault Tolerance:

**Redundancy:**
- Multi-AZ deployment.
- Database replicas per AZ.
- Load balancer distribution.
- S3 automatic replication.
- No single point of failure.

**Retry Logic:**
- Exponential backoff.
- Maximum retry limits.
- Idempotency keys.
- Circuit breakers.
- Jitter for thundering herd prevention.

### Disaster Recovery:

**Backup Strategy:**
- DynamoDB point-in-time recovery.
- S3 versioning.
- RDS automated daily backups.
- Version-controlled configuration.
- Automated backup verification.

**Recovery Procedures:**
- Documented runbooks.
- Regular DR drills.
- Automated recovery where possible.
- RTO: 4 hours.
- RPO: 1 hour.

## Performance Optimization:

### Caching Strategy:

**Application Caching:**
- ElastiCache for frequent data.
- Cache-aside pattern.
- Write-through for consistency.
- Volatility-based TTL.
- Deployment cache warming.

**CDN Caching:**
- CloudFront for static assets.
- Update invalidation.
- Path-specific behaviors.
- Geographic distribution.
- Hit ratio monitoring.

### Database Optimization:

**Query Optimization:**
- Access pattern-designed indexes.
- Composite keys for range queries.
- Sparse indexes.
- Regular plan analysis.
- Slow query identification.

**Connection Management:**
- Connection pooling.
- Limit enforcement.
- Idle connection recycling.
- Health checks.
- Prepared statements.

### Asynchronous Processing:

- Event-driven architecture.
- SQS buffering.
- Background job processing.
- Dead letter queues.
- Batch processing.

## Technology Stack:

**Frontend:** React 18, TypeScript 5, Tailwind CSS 3, React Query, Socket.io.

**Backend:** Python 3.11 (Lambda), Node.js 20 (API Gateway), FastAPI, Pydantic.

**Infrastructure:** Lambda, API Gateway, DynamoDB, S3, ElastiCache, CloudFront, Secrets Manager, CloudWatch, X-Ray.

**AI/ML:** Amazon Bedrock, SageMaker, Anthropic Claude Sonnet 4.5, OpenAI GPT-5.X, Google Gemini 2.5/3 Pro, HuggingFace Transformers, vLLM, Ollama (local: Qwen-3, GLM-4, Llama 3.3).

**Data Processing:** Kinesis, AWS Glue, Apache Spark.

**Development:** GitHub, GitHub Actions, Terraform, Docker.

## Design Decisions:

### Decision 1: Serverless Architecture:
**Rationale:** Pay-per-use pricing, automatic scaling, reduced operational overhead.  
**Trade-offs:** Cold start latency, execution time limits, stateless design required.

### Decision 2: DynamoDB for Primary Storage:
**Rationale:** Single-digit millisecond latency, automatic scaling, managed service.  
**Trade-offs:** Query patterns must be known upfront, complex queries require GSIs.

### Decision 3: Event-Driven Architecture:
**Rationale:** Loose coupling, fault tolerance, independent scaling.  
**Trade-offs:** Eventual consistency, debugging complexity, message ordering challenges.

### Decision 4: Multi-Model LLM Strategy (Cloud + Local):
**Rationale:** Flexibility across cloud providers (Claude Sonnet 4.5, GPT-5.X, Gemini 2.5/3 Pro) and local models (Qwen-3, GLM-4, Llama 3.3) enables cost optimization, data privacy options, and vendor lock-in avoidance. Local LLMs provide zero-cost inference at scale and full data sovereignty.  
**Trade-offs:** Integration complexity, consistency challenges across models, testing overhead, infrastructure management for local deployments, GPU cost for self-hosted models.

## Capacity Planning:

**Initial Launch (1K users):**
- 10 posts/user/week.
- 100 engagement events/user/day.
- 1TB storage.
- 10M API requests/month.

**6 Months (10K users):**
- 10 posts/user/week.
- 100 engagement events/user/day.
- 10TB storage.
- 100M API requests/month.

**12 Months (50K users):**
- 10 posts/user/week.
- 100 engagement events/user/day.
- 50TB storage.
- 500M API requests/month.

## Correctness Properties:

### Property 1: Authentication Token Validity:
**Validates:** Requirements 1.1, 1.2.  
**Property:** For all authentication attempts, if credentials are valid, a JWT token is issued with correct expiration and can be validated successfully until expiration.  
**Test Strategy:** Property-based test generating random valid/invalid credentials and verifying token lifecycle.

### Property 2: Content Safety Filtering:
**Validates:** Requirements 3.1, 3.2.  
**Property:** For all generated content, if toxicity score > 0.8 OR hate speech detected OR brand alignment < 0.7, content must be flagged for review and not auto-published.  
**Test Strategy:** Property-based test with generated content samples across safety thresholds.

### Property 3: Rate Limit Enforcement:
**Validates:** Requirements 4.8, Platform Compliance.  
**Property:** For all engagement actions, the system never exceeds platform-specific rate limits within any time window.  
**Test Strategy:** Property-based test simulating high-volume engagement scenarios and verifying rate limit compliance.

### Property 4: Scheduled Post Timing:
**Validates:** Requirements 8.1, 8.4.  
**Property:** For all scheduled posts, publication occurs within 30 seconds of scheduled time OR retry is attempted with exponential backoff.  
**Test Strategy:** Property-based test with various scheduling scenarios and time zones.

### Property 5: Data Encryption:
**Validates:** Security Requirements 2.4.2.  
**Property:** For all OAuth tokens and sensitive data, storage is encrypted using AES-256 and retrieval returns decrypted data only to authorized principals.  
**Test Strategy:** Property-based test verifying encryption at rest and access control.

### Property 6: Engagement Response Appropriateness:
**Validates:** Requirements 4.2, 4.3.  
**Property:** For all incoming engagements, if sentiment is negative, response tone is empathetic; if positive, response is appreciative; if spam, no response generated.  
**Test Strategy:** Property-based test with sentiment-labeled engagement samples.

### Property 7: Analytics Aggregation Accuracy:
**Validates:** Requirements 6.1, 6.2.  
**Property:** For all time periods, aggregated metrics equal the sum of individual metric events within that period.  
**Test Strategy:** Property-based test comparing raw events to aggregated results.

### Property 8: Multi-Platform Content Formatting:
**Validates:** Requirements 2.3.  
**Property:** For all generated content, platform-specific constraints are satisfied (Instagram ≤ 2200 chars, X ≤ 280 chars per tweet, LinkedIn ≤ 3000 chars).  
**Test Strategy:** Property-based test generating content for each platform and verifying length constraints.

## Model Selection Strategy:

### Default LLM Models by Content Type:

**Instagram Posts:**
- Primary: Claude Sonnet 4.5 (creative, visual-focused captions with strong emoji and hashtag generation).
- Fallback: Gemini 2.5 Pro (multi-modal understanding for image-text alignment).
- Local Option: Qwen-3 (72B+ parameters, strong multilingual support).
- Rationale: Claude Sonnet 4.5 excels at concise, engaging social content with natural emoji placement and cultural awareness.

**X (Twitter) Threads:**
- Primary: GPT-5.X (superior thread coherence, conversational tone, real-time awareness).
- Fallback: Claude Sonnet 4.5.
- Local Option: Llama 3.3 (70B+, optimized for dialogue).
- Rationale: GPT-5.X's advanced reasoning and platform-native voice understanding provides better thread structure.

**LinkedIn Articles:**
- Primary: Claude Sonnet 4.5 (professional tone, longer-form content, thought leadership, extended context).
- Fallback: Gemini 3 Pro (research-backed content, fact-checking capabilities).
- Local Option: GLM-4 (strong reasoning, professional Chinese/English content).
- Rationale: Claude Sonnet 4.5's extended context window (200K+ tokens) and formal writing style suits professional long-form content.

**Short-form Engagement Responses:**
- Primary: Gemini 2.5 Flash (cost-effective, ultra-fast, sufficient for brief replies).
- Fallback: Claude Sonnet 4.5 (quality fallback).
- Local Option: Qwen-3 (14B variant, optimized for inference speed).
- Rationale: Lower latency (<500ms) and cost for high-volume automated responses while maintaining quality.

**Marketing/Promotional Content:**
- Primary: GPT-5.X (creative copywriting, persuasive language, brand voice adaptation).
- Fallback: Claude Sonnet 4.5.
- Local Option: Llama 3.3 (fine-tuned on marketing copy).
- Rationale: GPT-5.X's advanced creative capabilities and training on diverse marketing materials.

**Multi-lingual Content:**
- Primary: Qwen-3 (exceptional multilingual performance, 29+ languages).
- Fallback: Gemini 2.5 Pro (strong translation and cultural adaptation).
- Cloud Option: Claude Sonnet 4.5.
- Rationale: Qwen-3's native multilingual training provides superior non-English content quality.

**Privacy-Sensitive/On-Premise Deployments:**
- Primary: Llama 3.3 (70B+, fully open-source, enterprise-ready).
- Secondary: Qwen-3 (72B, strong performance-to-size ratio).
- Tertiary: GLM-4 (bilingual excellence, efficient inference).
- Rationale: Local deployment ensures data sovereignty and eliminates API costs for high-volume users.

## High-Value Account Identification:

### Scoring Algorithm:

Accounts are scored on a 0-100 scale using weighted factors:

**Engagement Potential (40%):**
- Follower count relative to user's niche (0-25 points).
- Average engagement rate on posts (0-15 points).

**Relevance (30%):**
- Content topic alignment with user's niche (0-20 points).
- Shared audience overlap (0-10 points).

**Influence (20%):**
- Verified status (+10 points).
- Industry authority indicators (0-10 points).

**Reciprocity Likelihood (10%):**
- Historical engagement with similar accounts (0-10 points).

**Threshold:** Accounts scoring ≥ 70 are classified as high-value and prioritized for proactive engagement.

**Dynamic Adjustment:** Thresholds adjust based on user's account size (smaller accounts have lower thresholds to ensure sufficient engagement opportunities).

## A/B Testing Configuration:

### Hybrid Approach:

**Automatic A/B Testing (Default):**
- System automatically generates 2 variants for each content piece.
- Variants differ in: hook/opening line, CTA placement, emoji usage, hashtag strategy.
- 20% of content budget allocated to testing variants.
- Winner determined after 24 hours based on engagement rate.
- Winning patterns incorporated into future generation.

**User Configuration Options:**
- Users can disable automatic A/B testing.
- Users can specify which elements to test (tone, length, hashtags, CTAs).
- Users can set custom test duration and sample size.
- Premium users access advanced multivariate testing (3+ variants).

**Rationale:** Automatic testing provides value out-of-box while allowing power users to customize.

## LLM Provider Fallback Strategy:

### Multi-Tier Failover:

**Tier 1 - Primary Cloud Providers (Amazon Bedrock):**
- Claude Sonnet 4.5, GPT-5.X, and Gemini 2.5/3 Pro via Bedrock.
- Health check every 30 seconds.
- Load balancing across available models based on latency and cost.

**Tier 2 - Direct Provider APIs:**
- On Bedrock failure, switch to direct Anthropic API (Claude Sonnet 4.5).
- Parallel fallback to OpenAI API (GPT-5.X).
- Google AI Studio fallback (Gemini 2.5/3 Pro).
- Automatic retry with exponential backoff (3 attempts, max 10s delay).

**Tier 3 - Local LLM Deployment:**
- Qwen-3 (72B) via vLLM or TGI inference server.
- Llama 3.3 (70B) via Ollama or vLLM.
- GLM-4 (9B) for lightweight, fast inference.
- Deployed on ECS with GPU instances (A10G/A100).
- Zero external API dependency, full data privacy.

**Tier 4 - Degraded Mode:**
- Template-based content generation using pre-approved templates.
- Smaller local models (Qwen-3 14B, Llama 3.2 8B) for basic generation.
- User notification of degraded service.
- Queue content for generation when service restores.

**Tier 5 - Manual Mode:**
- After 15 minutes of all provider failures, system enters manual mode.
- Users notified to create content manually.
- Draft queue preserved for later AI enhancement.

**Circuit Breaker:** After 5 consecutive failures, provider marked unhealthy for 5 minutes before retry.

**Local LLM Benefits:**
- Cost reduction: $0 per token after infrastructure costs.
- Data privacy: No data leaves infrastructure.
- Latency: Sub-second inference with proper GPU allocation.
- Customization: Fine-tuning on user-specific data.

## Team Brand Voice Conflict Resolution:

### Hierarchical Approval System:

**Role-Based Priority:**
- Super Admin: Can set organization-wide brand voice (highest priority).
- Brand Manager: Can set team/department brand voice.
- Content Creator: Can set personal preferences (lowest priority).

**Conflict Resolution Rules:**
1. Organization-wide settings override all others.
2. Team settings override individual preferences.
3. Individual preferences apply only when no higher-level settings exist.

**Consensus Mode (Optional):**
- Teams can enable "consensus mode" requiring 2+ approvers for brand voice changes.
- Changes require majority approval from designated brand stakeholders.
- Voting period: 48 hours, then defaults to current settings.

**Version Control:**
- All brand voice changes tracked with timestamps and authors.
- Ability to rollback to previous brand voice versions.
- A/B testing between old and new brand voice before full adoption.

**Workspace Isolation:**
- Each workspace (client/brand) has independent brand voice settings.
- No cross-contamination between workspaces.

**Conflict Indicators:**
- Dashboard shows when content deviates from brand voice guidelines.
- Flagged content requires additional approval from brand manager.
- Monthly brand voice consistency reports generated.

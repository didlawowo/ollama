# External Integrations

**Analysis Date:** 2026-03-22

## APIs & External Services

**Model Registry / Container Registry:**
- Container registry compatibility for model distribution
- Authentication flow: Challenge-response digest auth using ED25519 signatures
- Location: `server/auth.go` - Registry authentication handler

**OpenAI API Compatibility Layer:**
- Provides partial OpenAI API compatibility for client libraries
- Location: `openai/openai.go`
- Supported endpoints: Chat completions, embeddings
- Feature: Streaming responses with Server-Sent Events (SSE)
- No direct external API calls; middleware adapter for protocol translation

## Data Storage

**Databases:**
- None detected - No SQL database integration
- Model metadata stored in local filesystem

**File Storage:**
- Local filesystem only
- Location: `~/.ollama/models` (configurable via `OLLAMA_MODELS`)
- Blob storage: Multi-part download support for large model files
- Location: `server/download.go` - Blob download with parallel part transfers
- Features:
  - Up to 16 parallel download parts
  - Part size range: 100MB to 1GB
  - Automatic retry with max 6 attempts
  - Stall detection with timeout

**Model Format Support:**
- GGUF format (native llama.cpp models)
- Format converters: `convert/convert_*.go`
  - BERT → GGUF
  - Gemma/Gemma2 → GGUF
  - LLaMA/LLaMA Adapter → GGUF
  - Mixtral → GGUF
  - Phi3 → GGUF
  - Pixtral (multimodal) → GGUF
- Tokenizer support:
  - SentencePiece (via protobuf: `convert/sentencepiece/`)
  - PyTorch format handling: `convert/reader_torch.go`
  - SafeTensors format: `convert/reader_safetensors.go`

**Caching:**
- In-memory model caching (KV cache management)
- Image embedding cache: `llama/runner/image.go`
  - LRU-style cache for processed image embeddings
  - Multi-user prompt cache optimization flag: `OLLAMA_MULTIUSER_CACHE`
- No external cache service (Redis, Memcached, etc.)

## Authentication & Identity

**Auth Provider:**
- Custom local authentication for model registry
- Implementation: ED25519 public/private key cryptography
- Location: `auth/auth.go`

**Key Management:**
- Private key storage: `~/.ollama/id_ed25519` (SSH format)
- Challenge nonce generation for registry auth
- No external OAuth/OIDC provider

## Monitoring & Observability

**Error Tracking:**
- None detected - No Sentry, DataDog, or similar integration

**Logs:**
- `log/slog` (Go standard structured logging)
- Log output to stdout/stderr
- Environment variable: `OLLAMA_DEBUG` for debug-level logging
- Locations: Throughout codebase (e.g., `app/lifecycle/logging.go`)

**Metrics:**
- Not detected - No Prometheus, StatsD, or metrics export

## CI/CD & Deployment

**Hosting:**
- Not a hosted service - Ollama is self-hosted software
- Docker container support (multi-stage builds in `Dockerfile`)

**CI Pipeline:**
- GitHub Actions (`.github/workflows/`)
- Main workflows:
  - `test.yaml` - Unit and integration tests on PR changes
  - `release.yaml` - Release build distribution
  - `latest.yaml` - Latest image publication
- GPU runner compilation: Conditional compilation for CUDA/ROCm on PR changes

**Build Targets:**
- Native binary distribution for: Linux (amd64/arm64), macOS, Windows
- Docker images for: CPU-only, CUDA 11/12, ROCm, Jetson (JetPack 5/6)
- Container registry: DockerHub (implied from release workflows)

## Environment Configuration

**Required env vars:**
- `OLLAMA_HOST` - Server listen address (default: `http://127.0.0.1:11434`)
- Optional GPU device visibility:
  - `CUDA_VISIBLE_DEVICES` (NVIDIA)
  - `HIP_VISIBLE_DEVICES` (AMD)
  - `ROCR_VISIBLE_DEVICES` (AMD alternative)

**Secrets location:**
- Private key: `~/.ollama/id_ed25519` (local file system)
- No .env files or secret management system detected
- Auth credentials are locally generated (no external secrets)

## Webhooks & Callbacks

**Incoming:**
- None detected - No incoming webhook support

**Outgoing:**
- None detected - No outgoing webhook/notification system

## Model Distribution

**Model Pull/Push:**
- Blob-based distribution with digest verification
- Multi-part parallel download capability (up to 16 parts)
- Automatic checksum validation (digest matching)
- Location: `server/download.go`, `server/images.go`

**Model Metadata:**
- Manifest-based model definitions
- Location: `server/manifest.go`, `server/model.go`
- Contains: Model name, tags, digest references, layer info

## Server API

**REST Endpoints:**
- Chat completions: `/api/chat`
- Generate: `/api/generate`
- Embeddings: `/api/embed`
- Models: `/api/tags`, `/api/show`, `/api/list`
- Model management: `/api/pull`, `/api/push`, `/api/delete`
- System: `/api/ps` (process list)
- Streaming: Server-Sent Events (SSE) for chat/generate endpoints

**Response Formats:**
- JSON for API responses
- NDJSON (newline-delimited JSON) for streaming responses
- OpenAI-compatible format support via `openai/openai.go`

## Multimodal Support

**Image Processing:**
- Image embedding generation for vision models
- Location: `llama/runner/image.go`, `model/pixtral/imageproc.go`
- Formats: Image processing via standard image library
- Encoding: Base64 encoding for API transmission

## No External Integrations Detected

**Not used:**
- SQL databases (PostgreSQL, MySQL, SQLite)
- Message queues (RabbitMQ, Kafka)
- Object storage services (S3, GCS, Azure Blob)
- Vector databases (Pinecone, Weaviate)
- LLM APIs (OpenAI API client, Anthropic API)
- Search engines (Elasticsearch)
- Payment processors
- Email services
- Notification services (Slack, Discord)

---

*Integration audit: 2026-03-22*

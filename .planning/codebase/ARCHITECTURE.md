# Architecture

**Analysis Date:** 2026-03-22

## Pattern Overview

**Overall:** Distributed service architecture with request-driven scheduler and pluggable compute backends

**Key Characteristics:**
- HTTP REST API for client communication (Gin framework)
- Central scheduler managing model loading and runner lifecycle
- Pluggable LLM backends (C++ inference engine via cgo)
- GPU-aware resource scheduling with automatic offloading
- OpenAI API compatibility layer for chat operations
- Model manifest/blob storage system for versioning

## Layers

**HTTP API Layer:**
- Purpose: Expose REST endpoints for model generation, chat, embeddings, and management
- Location: `server/routes.go`
- Contains: Gin route handlers (GenerateHandler, ChatHandler, EmbedHandler, etc.)
- Depends on: Server scheduler, model management, OpenAI middleware
- Used by: Client CLI, external API consumers

**Scheduler Layer:**
- Purpose: Orchestrate model loading, runner allocation, and resource constraints
- Location: `server/sched.go`
- Contains: Request queuing, GPU/CPU assignment, context window management, runner lifecycle
- Depends on: LLM backend server factory, GPU discovery, environment config
- Used by: Route handlers via scheduleRunner()

**LLM Backend Layer:**
- Purpose: Load and execute inference on GGML format models
- Location: `llm/server.go`, `llm/memory.go`, `llm/ggml.go`
- Contains: Model loading, memory estimation, GPU VRAM allocation, backend selection
- Depends on: C++ llama.cpp bindings, GPU discovery, GGML format parsing
- Used by: Scheduler for runner creation

**Go Runner Layer:**
- Purpose: Native Go server for sequence-based inference (replacing C++ server)
- Location: `llama/runner/runner.go`, `llama/runner/cache.go`
- Contains: Sequence management, batching, prompt processing, token generation
- Depends on: llama.go cgo bindings, sampling context, image processing
- Used by: New-style LLM backend instances

**Model Management Layer:**
- Purpose: Handle model manifest parsing, blob storage, conversion, and layer management
- Location: `server/model.go`, `server/manifest.go`, `server/layer.go`
- Contains: Model metadata, layer handling (model, projector, adapter), blob operations
- Depends on: Convert package for format conversion, types/model for parsing
- Used by: Route handlers and scheduler for model lookup and configuration

**Command Layer:**
- Purpose: CLI interface for model operations (create, pull, push, run, etc.)
- Location: `cmd/cmd.go`
- Contains: Cobra CLI definitions, interactive shell, model build/pull/push logic
- Depends on: API client, model management, parser
- Used by: Direct user invocation via main.go

**Supporting Packages:**
- `api/`: Type definitions for requests/responses (GenerateRequest, ChatRequest, Options)
- `parser/`: Modelfile parsing (FROM, ADAPTER, PARAMETER directives)
- `template/`: Prompt template system for different model formats (llama2-chat, phi-3, etc.)
- `convert/`: Model format conversion (Safetensors, PyTorch → GGML)
- `types/model/`: Model name parsing and validation
- `discover/`: GPU/CPU capability detection (CUDA, Metal, ROCm, CPU)

## Data Flow

**Inference Request:**

1. Client sends POST `/api/generate` or `/api/chat` with model name and prompt
2. Route handler (GenerateHandler, ChatHandler) validates request and parses model name
3. Handler calls `scheduleRunner()` to obtain runner for model
4. Scheduler checks if model already loaded in `s.loaded` map
5. If not loaded:
   - Scheduler calls `s.load()` which creates new LlamaServer via `llm.NewLlamaServer()`
   - LlamaServer loads GGML file, estimates VRAM, determines GPU layers
   - C++ backend (llama.cpp) or Go runner spins up and listens on local port
   - Scheduler tracks runner reference with expiration time
6. Handler receives runner and sends completion request via `runner.Completion()`
7. Runner processes prompt tokens through LLM layers
8. Tokens streamed back to handler, formatted into API response
9. Response sent to client (streaming or batched)
10. On keep_alive timeout, runner unloaded from `s.loaded` and process terminated

**Model Loading:**

1. User runs `ollama create mymodel -f Modelfile`
2. Parser reads Modelfile (FROM base_model, ADAPTER adapter, PARAMETER options)
3. If base model not found locally, PullModel downloads from registry
4. Model converted to GGML format if needed (via convert package)
5. Layer blobs created and stored in blob directory
6. Manifest created and stored alongside model path

**State Management:**
- **Loaded runners:** `Scheduler.loaded[modelPath] → *runnerRef` map, protected by mutex
- **Pending requests:** `Scheduler.pendingReqCh` channel queue, max size configurable
- **Model cache:** `GetModel()` reads from manifest files on disk, no in-memory cache
- **GPU state:** Captured at runner startup in `llmServer.gpus`, updated on status queries

## Key Abstractions

**Model:**
- Purpose: Represents a versioned model with layers and configuration
- Examples: `server/model.go` Model struct, contains Layers []Layer, Options map
- Pattern: Lazy-loaded from disk via ParseNamedManifest, manifest format is tar + config

**LlamaServer interface:**
- Purpose: Abstraction for inference backend (C++ or Go)
- Examples: `llm/server.go` LlamaServer interface with Completion, Embedding, Tokenize methods
- Pattern: Implemented by `llmServer` (C++ executable wrapper) and `runner.Server` (Go native)

**Runner (runnerRef):**
- Purpose: Reference to an active LlamaServer with lifecycle management
- Examples: `server/sched.go` runnerRef tracks LlamaServer, ref count, expiration
- Pattern: Ref-counted, expires after inactivity (default: 5 minutes via KeepAlive)

**Sequence:**
- Purpose: Tracks a single inference request through token generation
- Examples: `llama/runner/runner.go` Sequence with inputs, responses, sampling context
- Pattern: Created per request, holds prompt/response channels, manages batching

**CompletionRequest/CompletionResponse:**
- Purpose: Wire format for streaming token generation
- Examples: `llm/server.go` structs with prompt, context, options, tokens, timing
- Pattern: Streamed from runner to scheduler to route handler to client

## Entry Points

**Main CLI:**
- Location: `main.go`
- Triggers: `ollama [command] [args]`
- Responsibilities: Creates CLI via cmd.NewCLI() and executes

**Server Entry:**
- Location: `cmd/cmd.go` in cmd.ExecuteContext()
- Triggers: `ollama serve` command (default if no command)
- Responsibilities: Starts HTTP server on configured port, initializes scheduler, listens for OS signals

**HTTP Routes:**
- Location: `server/routes.go` NewServer()
- Triggers: HTTP requests to `/api/*` endpoints
- Responsibilities: Route mounting, CORS, OpenAI compatibility middleware

## Error Handling

**Strategy:** Context propagation with structured logging, HTTP status codes, and graceful degradation

**Patterns:**

- **Model not found:** Returns 404 with JSON error message, suggests closest model name
- **Capability mismatch:** Returns 400 if model doesn't support requested capability (e.g., chat on completion-only model)
- **Resource exhausted:** Returns 503 with `ErrMaxQueue` if pending request queue full
- **Scheduler timeout:** Returns 504 if runner fails to become ready within deadline
- **Inference error:** Returns 500 with detailed error from backend, logged at ERROR level
- **Cancellation:** Request context cancellation triggers cleanup, pending request dropped from queue

## Cross-Cutting Concerns

**Logging:** Structured logging via `log/slog`, with DEBUG level for scheduler details, WARN for fallbacks, ERROR for failures. Environment variable controls level.

**Validation:**
- Model name validation in `types/model/name.go` (strict format checking)
- Options validation in `api/types.go` via Options.FromMap() (type coercion and bounds)
- Request body validation via Gin `ShouldBindJSON()` (schema-based)

**Authentication:** Optional token-based auth in `auth/` package, enforced via middleware if configured. Defaults to no auth.

**Resource Management:**
- Semaphore-based concurrency control per runner (`llmServer.sem`)
- Manual context cancellation for cleanup (no defers, explicit Close() calls)
- GPU VRAM tracking via `MemoryEstimate` structs, respects NumGPU option
- CPU fallback when VRAM insufficient or NumGPU=0

---

*Architecture analysis: 2026-03-22*

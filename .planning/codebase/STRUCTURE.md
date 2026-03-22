# Codebase Structure

**Analysis Date:** 2026-03-22

## Directory Layout

```
ollama/
├── cmd/                    # CLI and server startup
├── server/                 # HTTP API routes and scheduler
├── api/                    # Request/response types
├── llm/                    # LLM backend abstraction and GGML parsing
├── llama/                  # C++ inference engine bindings (cgo)
├── model/                  # Model-specific components (mllama, pixtral, qwen2vl)
├── convert/                # Model format conversion (Safetensors, PyTorch)
├── template/               # Prompt template system
├── parser/                 # Modelfile parsing
├── types/                  # Type definitions (model names, errors)
├── discover/               # GPU/CPU capability detection
├── auth/                   # Optional authentication
├── openai/                 # OpenAI API compatibility middleware
├── progress/               # Progress reporting for long operations
├── format/                 # Formatting utilities (human-readable sizes, tables)
├── envconfig/              # Environment variable parsing
├── util/                   # General utilities
├── version/                # Version information
├── examples/               # Example code and usage
├── docs/                   # Documentation
├── app/                    # Desktop/platform app integration
├── macapp/                 # macOS app specific code
├── scripts/                # Build and utility scripts
├── Makefile                # Build configuration
├── Dockerfile              # Container image definition
├── go.mod, go.sum          # Go module dependencies
└── main.go                 # Entry point
```

## Directory Purposes

**cmd/**
- Purpose: CLI command definitions and server initialization
- Contains: Cobra command tree, model create/pull/push/run/list logic, interactive shell
- Key files: `cmd.go` (CLI root), `start.go` (server startup), `interactive.go` (REPL)
- Environment: Handles platform-specific startup (Darwin/Windows)

**server/**
- Purpose: HTTP server implementation and request handling
- Contains: Route handlers, scheduler, model management, authentication
- Key files:
  - `routes.go` (1661 lines) - All HTTP endpoints (generate, chat, embed, models, etc.)
  - `sched.go` (836 lines) - Request scheduler and runner lifecycle
  - `model.go` - Model loading and conversion
  - `manifest.go` - Model manifest parsing and storage
  - `download.go` - Registry model download/pull
  - `auth.go` - Authentication middleware

**api/**
- Purpose: Request/response type definitions and client
- Contains: API contracts for all endpoints
- Key files:
  - `types.go` - GenerateRequest, ChatRequest, EmbedRequest, Message, Tool structs
  - `client.go` - HTTP client for calling Ollama API

**llm/**
- Purpose: LLM backend abstraction and GGML format support
- Contains: Backend server lifecycle, memory estimation, GGML parsing
- Key files:
  - `server.go` (800+ lines) - LlamaServer interface, NewLlamaServer factory
  - `ggml.go` - GGML file format decoding and inspection
  - `memory.go` - VRAM estimation algorithms
  - `status.go` - Backend status reporting

**llama/**
- Purpose: C++ inference engine bindings via cgo
- Contains: Direct C/C++ source code and Go wrapper (only 3% of the codebase)
- Key files:
  - `llama.go` - cgo directives and Go type bindings
  - `*.c`, `*.cpp`, `*.h` - Unmodified llama.cpp source
  - `runner/` subdirectory - Go-native runner implementation
- Note: Compiled with platform-specific optimizations (AVX2, CUDA, Metal, ROCm)

**llama/runner/**
- Purpose: Native Go inference server (new, replaces C++ server)
- Contains: Sequence batching, token sampling, image embedding
- Key files:
  - `runner.go` - Server and Sequence types
  - `cache.go` - KV cache management
  - `image.go` - Vision projector/CLIP integration

**model/**
- Purpose: Model-specific implementations and processors
- Contains: Subdirectories for different architectures
- Subdirectories:
  - `mllama/` - Meta Llama implementation
  - `pixtral/` - Mistral Pixtral multimodal
  - `qwen2vl/` - Qwen vision-language
  - `imageproc/` - Image processing utilities

**convert/**
- Purpose: Model format conversion pipeline
- Contains: Readers/writers for different formats
- Key files:
  - `convert.go` - Main conversion dispatch
  - `convert_llama.go`, `convert_mixtral.go`, etc. - Architecture-specific converters
  - `reader_safetensors.go`, `reader_torch.go` - Input format readers
  - `tokenizer.go` - Tokenizer handling

**template/**
- Purpose: Prompt templating system for different model formats
- Contains: Built-in templates and template engine
- Key files:
  - `template.go` (434 lines) - Template parsing and rendering
  - `*.gotmpl` - Template definitions (llama2-chat, phi-3, mistral-instruct, etc.)
  - `*.json` - Template parameters (stop tokens, etc.)
  - `index.json` - Template registry

**parser/**
- Purpose: Modelfile syntax parsing
- Contains: Modelfile DSL parser (FROM, ADAPTER, PARAMETER, SYSTEM, TEMPLATE directives)
- Key files: `parser.go` - Recursive descent parser for Modelfile format

**types/**
- Purpose: Shared type definitions
- Contains:
  - `model/name.go` - Model name parsing and validation
  - `errtypes/` - Typed errors

**discover/**
- Purpose: GPU/CPU capability detection
- Contains: Platform-specific GPU detection (CUDA, Metal, ROCm), system info gathering
- Key files: Platform-specific files (linux.go, darwin.go, windows.go)

**auth/**
- Purpose: Optional authentication middleware
- Contains: Token-based auth implementation
- Usage: Only active if OLLAMA_AUTH env var configured

**openai/**
- Purpose: OpenAI API compatibility layer
- Contains: Request/response adapters for /v1/chat/completions compatibility
- Key files: `openai.go` (200+ lines) - Message and tool call transformation

**progress/**
- Purpose: Progress reporting for long operations
- Contains: Progress bars and status updates
- Usage: Model downloads, conversions, inference

**format/**
- Purpose: Output formatting utilities
- Contains: Human-readable file sizes, table formatting
- Key files: Uses `github.com/mattn/go-runewidth` and `github.com/olekukonko/tablewriter`

**envconfig/**
- Purpose: Environment variable configuration
- Contains: Config parsing (OLLAMA_HOST, OLLAMA_NUM_GPU, OLLAMA_MAX_QUEUE, etc.)

**util/**
- Purpose: General-purpose utilities
- Contains: Buffer utilities, file operations

**version/**
- Purpose: Version information
- Contains: Build-time version strings

**app/**
- Purpose: Platform-specific application integration
- Contains: macOS/Windows app integration logic

**examples/**
- Purpose: Example usage and integration patterns
- Contains: Sample client code, Modelfiles

**scripts/**
- Purpose: Build and deployment scripts
- Contains: Signing scripts, platform-specific build helpers

## Key File Locations

**Entry Points:**
- `main.go`: Minimal entry point, delegates to cmd.NewCLI()
- `cmd/cmd.go`: CLI root with all subcommands (serve, run, create, pull, push, etc.)
- `server/routes.go`: HTTP server initialization and route registration

**Configuration:**
- `envconfig/`: All environment variable parsing
- `.golangci.yaml`: Linting configuration

**Core Logic:**
- `server/sched.go`: Model scheduling and lifecycle
- `server/model.go`: Model manifest and metadata handling
- `llm/server.go`: LLM backend abstraction
- `api/types.go`: API request/response contracts
- `template/template.go`: Prompt template system

**Testing:**
- `server/routes_test.go` (689 lines)
- `server/routes_generate_test.go` (954 lines) - Comprehensive inference tests
- `server/sched_test.go` (793 lines)
- Multiple `*_test.go` files throughout (42221 total lines of Go code)

## Naming Conventions

**Files:**
- Implementation files: `feature.go` (lowercase)
- Test files: `feature_test.go`
- Platform-specific: `feature_platform.go` (e.g., `start_darwin.go`, `start_windows.go`)
- Generated: `*_gen.go` (build tags control usage)

**Directories:**
- Lowercase, single word when possible: `cmd`, `api`, `model`
- Multi-purpose: `types/model/`, `llama/runner/`
- Avoid single-letter directories (except `v` for versioning)

**Types:**
- Exported: PascalCase (Server, GenerateRequest, LlamaServer)
- Unexported: camelCase (llmServer, runnerRef, pendingReqCh)
- Interfaces: CapitalCase ending in suffix (LlamaServer, Capability)

**Functions:**
- Exported methods: PascalCase (GenerateHandler, NewLlamaServer)
- Unexported methods: camelCase (scheduleRunner, parseFromModel)
- Constructors: NewTypeName pattern (NewLlamaServer, NewSequence)
- Handlers: TypeHandler pattern (GenerateHandler, ChatHandler, EmbedHandler)

**Variables:**
- Constants: UPPER_SNAKE_CASE or CamelCase (defaultParallel, ErrMaxQueue)
- Config vars: descriptive camelCase (pendingReqCh, loadedMu)
- Package-level: prefixed with context (errRequired, errModelNotFound)

## Where to Add New Code

**New Inference Feature (e.g., new endpoint):**
- Primary code: `server/routes.go` (add handler function and route registration)
- Tests: `server/routes_test.go` (follow existing patterns for mocking)
- Types: Add request/response struct to `api/types.go`
- Documentation: Update `docs/` if user-facing

**New Backend/Runner Implementation:**
- Implementation: `llm/server.go` (implement LlamaServer interface) or extend `llama/runner/`
- Memory estimation: `llm/memory.go` (add EstimateGPULayers logic for new architecture)
- Tests: Parallel test file in same directory
- Discovery: Update `discover/` if new GPU type support

**New Model Architecture Support:**
- Architecture implementation: New subdirectory in `model/` (e.g., `model/mymodel/`)
- Conversion logic: New file in `convert/` following pattern of `convert_llama.go`
- Tokenizer handling: Update `convert/tokenizer.go` if needed
- Template: Add `*.gotmpl` and `*.json` to `template/` directory

**CLI Command:**
- Command definition: New method in `cmd/cmd.go` following Cobra patterns
- Logic: Extract to separate file if >100 lines (pattern: `cmd_action.go`)
- Tests: `cmd/cmd_test.go` or dedicated test file

**New Scheduler Feature:**
- Core logic: `server/sched.go` (extend Scheduler struct and processing loops)
- Model-specific: `server/model.go` (extend Model struct for metadata)
- Tests: `server/sched_test.go` with unit test coverage

**Shared Utilities:**
- String/number formatting: `format/`
- Buffer/file operations: `util/`
- API type definitions: `api/types.go` (preferred over scattered struct definitions)

**Configuration/Environment:**
- New env var parsing: `envconfig/` (follows pattern of MaxQueue(), NumParallel())

## Special Directories

**llama/:**
- Purpose: Contains unmodified llama.cpp C/C++ source + Go bindings
- Generated: Yes - compiled from C++ sources via cgo, platform-specific binaries
- Committed: Partially - *.go files committed, *.o/*.a built locally
- Note: This is vendored copy of llama.cpp project, avoid modifying C++ files

**build/ or bin/:**
- Purpose: Intermediate build artifacts (platform-specific)
- Generated: Yes - created during build process
- Committed: No - in .gitignore
- Contents: Platform binaries for runners, test artifacts

**examples/**
- Purpose: Reference implementations for users
- Committed: Yes
- Usage: Documentation, integration testing, user guidance

**make/**
- Purpose: Build system helper scripts
- Contains: Platform-specific build logic

**integration/**
- Purpose: Integration test suite
- Contents: End-to-end tests for full workflows
- Run: As part of CI/CD pipeline

---

*Structure analysis: 2026-03-22*

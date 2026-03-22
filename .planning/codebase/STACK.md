# Technology Stack

**Analysis Date:** 2026-03-22

## Languages

**Primary:**
- Go 1.23.4 - Core application, CLI, server, and API implementation

**Secondary:**
- C/C++ (via cgo) - llama.cpp bindings for inference engine
- Protocol Buffers - Sentencepiece tokenizer model definition
- Python (implicit) - Referenced in CI/CD build scripts

## Runtime

**Environment:**
- Go 1.23.4 runtime
- Supports Linux, macOS, Windows, and embedded systems (Jetson)
- Cross-platform compilation targets: amd64, arm64

**Package Manager:**
- Go modules (go.mod/go.sum)
- Lockfile: `go.sum` present and version-pinned

## Frameworks

**Core:**
- `github.com/gin-gonic/gin` v1.10.0 - REST API HTTP server framework
- `github.com/spf13/cobra` v1.7.0 - CLI command framework

**HTTP Middleware:**
- `github.com/gin-contrib/cors` v1.7.2 - CORS support for API
- `github.com/gin-contrib/sse` v0.1.0 - Server-sent events for streaming

**Testing:**
- `github.com/stretchr/testify` v1.9.0 - Testing assertions and mocks

**Build/Dev:**
- `make` - Build orchestration (`Makefile` at root)
- `golangci-lint` - Code linting (`.golangci.yaml` configuration)
- Docker - Multi-stage builds for different GPU targets (CUDA 11.3.1, CUDA 12.4.0, ROCm 6.1.2)
- Compiler toolchain: GCC (DevToolset-10 on CentOS for Linux builds)

## Key Dependencies

**Critical (Application):**
- `golang.org/x/crypto` v0.31.0 - Cryptographic operations for auth (ED25519 signing)
- `golang.org/x/sync` v0.10.0 - Synchronization primitives (errgroup for concurrent operations)
- `google.golang.org/protobuf` v1.34.1 - Protocol Buffers runtime for sentencepiece tokenizer
- `github.com/google/uuid` v1.6.0 - UUID generation

**Infrastructure:**
- `github.com/containerd/console` v1.0.3 - Terminal/console interaction
- `github.com/olekukonko/tablewriter` v0.0.5 - CLI table output formatting
- `golang.org/x/term` v0.27.0 - Terminal capabilities detection
- `golang.org/x/net` v0.25.0 - Networking utilities
- `golang.org/x/text` v0.21.0 - Text encoding and unicode support

**Optional/Experimental:**
- `golang.org/x/image` v0.22.0 - Image processing (multimodal model support)
- `github.com/emirpasic/gods` v1.18.1 - Data structures library
- `github.com/x448/float16` v0.8.4 - FP16 float support
- `github.com/nlpodyssey/gopickle` v0.3.0 - Python pickle format support (model conversion)

**ML/Tensor Processing:**
- `github.com/pdevine/tensor` v0.0.0-20240510204454 - Tensor operations (image embeddings)
- `gonum.org/v1/gonum` v0.15.0 - Numerical computing
- `gorgonia.org/vecf32`, `gorgonia.org/vecf64` - Vector operations

**Indirect/Transitive:**
- `github.com/bytedance/sonic` v1.11.6 - High-performance JSON serialization (via Gin)
- `github.com/gabriel-vasile/mimetype` v1.4.3 - MIME type detection
- `golang.org/x/exp` v0.0-20231110203233 - Experimental Go features
- `gopkg.in/yaml.v3` v3.0.1 - YAML parsing (config files)

## Configuration

**Environment:**
- Configured via environment variables (prefix: `OLLAMA_*`)
- Source: `envconfig/config.go` - Central configuration management

**Key Configuration Variables:**
- `OLLAMA_HOST` - Server address (default: `http://127.0.0.1:11434`)
- `OLLAMA_MODELS` - Model storage directory (default: `$HOME/.ollama/models`)
- `OLLAMA_KEEP_ALIVE` - Model memory retention (default: 5 minutes)
- `OLLAMA_LOAD_TIMEOUT` - Model load timeout (default: 5 minutes)
- `OLLAMA_NUM_PARALLEL` - Concurrent request limit
- `OLLAMA_MAX_LOADED_MODELS` - Maximum loaded models per GPU
- `OLLAMA_MAX_QUEUE` - Request queue size (default: 512)
- `OLLAMA_MAX_VRAM` - VRAM usage limit override
- `OLLAMA_GPU_OVERHEAD` - Per-GPU VRAM reservation
- `OLLAMA_ORIGINS` - CORS allowed origins
- `OLLAMA_DEBUG` - Debug logging flag
- `OLLAMA_FLASH_ATTENTION` - Experimental attention optimization
- `OLLAMA_KV_CACHE_TYPE` - KV cache quantization type
- GPU device variables: `CUDA_VISIBLE_DEVICES`, `HIP_VISIBLE_DEVICES`, `ROCR_VISIBLE_DEVICES`, `GPU_DEVICE_ORDINAL`
- Proxy configuration: `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`

**Build Configuration:**
- `Makefile` - Primary build orchestration
- `go.mod` - Dependency versioning
- `.golangci.yaml` - Linter configuration
- `Dockerfile` - Multi-platform container builds

## Platform Requirements

**Development:**
- Go 1.23.4+
- GCC/DevToolset-10 (for C/C++ bindings)
- CUDA 11.3.1 or 12.4.0 (optional, for NVIDIA GPU acceleration)
- ROCm 6.1.2 (optional, for AMD GPU acceleration)
- Docker (for cross-platform builds)

**Production:**
- Linux: CentOS 7+ / Rocky Linux 8+
- macOS: Apple Silicon or Intel (native binary)
- Windows: Windows Server 2022 or Windows 10/11
- Jetson: JetPack 5 (r35.4.1) or JetPack 6 (r36.2.0)
- NVIDIA GPUs: CUDA 11.3+ supported
- AMD GPUs: ROCm 6.1+ supported
- Intel GPUs: Optional experimental detection

**API Server:**
- HTTP/1.1 (via Gin framework)
- Listen ports: Default 11434, configurable via `OLLAMA_HOST`
- CORS enabled via gin-contrib/cors

---

*Stack analysis: 2026-03-22*

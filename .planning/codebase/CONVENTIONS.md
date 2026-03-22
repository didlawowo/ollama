# Coding Conventions

**Analysis Date:** 2026-03-22

## Naming Patterns

**Files:**
- Use lowercase with underscores for internal implementation details: `convert_llama.go`, `gpu_linux.go`, `reader_safetensors.go`
- Use standard Go convention: `filename_platform.go` for platform-specific implementations (e.g., `server_unix.go`, `server_windows.go`)
- Test files follow the pattern `name_test.go` (e.g., `client_test.go`, `convert_test.go`)
- Platform-specific tests use the pattern `name_platform_test.go` (e.g., `gpu_linux_test.go`)

**Functions:**
- Use PascalCase for exported functions: `ClientFromEnvironment()`, `GenerateTestHelper()`, `CreateHandler()`, `NewClient()`
- Use camelCase for unexported functions: `checkError()`, `getModelfileName()`, `createTokenizerFS()`, `readFile()`
- Descriptors in function names: Handler suffix for command handlers (`CreateHandler`, `DeleteHandler`, `PushHandler`)
- Test helpers: Helper functions explicitly marked with `t.Helper()` call

**Variables:**
- Use short, meaningful names: `err`, `buf`, `fp`, `ctx`, `req`, `resp`
- Struct field names use PascalCase: `StatusCode`, `ErrorMessage`, `Model`, `Prompt`
- Private struct fields use lowercase: `base`, `http`
- Map/slice element abbreviations: `tc` for test case, `tt` for table test, `bts` for bytes

**Types:**
- Use PascalCase for exported types: `Client`, `GenerateRequest`, `ChatResponse`, `StatusError`
- Type aliases for callbacks: `GenerateResponseFunc`, `ChatResponseFunc`, `PullProgressFunc` (Func suffix)
- Interfaces follow standard Go patterns: small, focused interfaces

## Code Style

**Formatting:**
- Go standard formatting via `gofmt` (enforced by the project)
- Line length: Follow Go conventions (typically 80-120 characters, no hard limit specified)
- Indentation: Tabs (Go standard)

**Linting:**
- No explicit linter configuration in repository (relies on Go standard tooling)
- Code demonstrates adherence to Go conventions without custom ESLint/Prettier configs

**Imports:**
- Organized in groups: standard library, then external packages, then internal packages
- Example from `api/client.go`:
  ```go
  import (
      "bufio"                    // standard library
      "bytes"
      "context"
      // ... other stdlib
      "github.com/ollama/ollama/envconfig"  // internal packages
      "github.com/ollama/ollama/format"
  )
  ```
- Use blank lines between import groups

## Import Organization

**Order:**
1. Standard library imports (e.g., `"context"`, `"io"`, `"net/http"`)
2. Third-party packages (e.g., `"github.com/spf13/cobra"`, `"github.com/stretchr/testify"`)
3. Internal packages (e.g., `"github.com/ollama/ollama/api"`, `"github.com/ollama/ollama/format"`)

**Path Aliases:**
- No path aliases used in codebase (direct package paths)
- Package imports match directory structure

## Error Handling

**Patterns:**

1. **Immediate error check:**
   ```go
   err := someFunction()
   if err != nil {
       return err
   }
   ```

2. **Error wrapping with context:**
   ```go
   if err := json.Unmarshal(bts, &errorResponse); err != nil {
       return fmt.Errorf("unmarshal: %w", err)
   }
   ```

3. **Error type assertion:**
   ```go
   var statusError api.StatusError
   switch {
   case errors.As(err, &statusError) && statusError.StatusCode == http.StatusNotFound:
       // Handle not found
   case err != nil:
       return err
   }
   ```

4. **Deferred cleanup with error handling:**
   ```go
   defer respObj.Body.Close()
   respBody, err := io.ReadAll(respObj.Body)
   if err != nil {
       return err
   }
   ```

5. **Custom error types:**
   - Define error variables at package level: `var errModelNotFound = errors.New("...")`
   - Custom error structs implement `Error()` method: `StatusError` with formatted output

**Error Checking Priority:**
- Check errors immediately after operations
- Always defer Close/cleanup before checking read errors
- Use `errors.Is()` and `errors.As()` for type-aware error checking

## Logging

**Framework:** `log/slog` (Go 1.21+ structured logging)

**Patterns:**

1. **Info level - general information:**
   ```go
   slog.Info("test pass", "model", genReq.Model, "prompt", genReq.Prompt, "contains", anyResp, "response", response)
   slog.Info("server connection", "host", host, "port", port)
   ```

2. **Warn level - non-critical issues:**
   ```go
   slog.Warn("invalid port, using default", "port", port, "default", defaultPort)
   slog.Warn("invalid option provided", "option", key)
   slog.Warn("error looking up nvidia GPU memory", "error", C.GoString(memInfo.err))
   ```

3. **Error level - error conditions:**
   ```go
   slog.Error("invalid CudaComputeMajorMin setting", "value", CudaComputeMajorMin, "error", err)
   slog.Error("failed to open server log", "logfile", lifecycle.ServerLogFile, "error", err)
   ```

4. **Debug level - detailed diagnostic info:**
   ```go
   slog.Debug("searching for GPU discovery libraries for NVIDIA")
   slog.Debug("nvidia-ml loaded", "library", libPath)
   slog.Debug("vocabulary", "size", len(t.Vocabulary.Tokens))
   ```

**Key-value pairs:** Structured logging with alternating keys and values - keys should be lowercase, concise

## Comments

**When to Comment:**
- Package-level documentation: Explain what the package does and provide usage examples
- Function documentation: Describe function purpose, parameters, and return values (doc comments)
- Complex logic: Explain the "why" rather than the "what"
- Non-obvious workarounds: Document reasons for unusual code patterns

**Doc Comments (GoDoc):**

Use `//` comments immediately before package, type, and function declarations:

```go
// Package api implements the client-side API for code wishing to interact
// with the ollama service. The methods of the [Client] type correspond to
// the ollama REST API as described in [the API documentation].
package api

// Client encapsulates client state for interacting with the ollama
// service. Use [ClientFromEnvironment] to create new Clients.
type Client struct {
    base *url.URL
    http *http.Client
}

// ClientFromEnvironment creates a new [Client] using configuration from the
// environment variable OLLAMA_HOST, which points to the network host and
// port on which the ollama service is listening.
func ClientFromEnvironment() (*Client, error) {
    // implementation
}
```

**Patterns observed:**
- Multi-line comments for complex descriptions
- Reference other functions/types with square bracket notation: `[Client.Generate]`, `[Modelfile]`
- Include external documentation links in package comments
- Example sections in package comments for common use cases

**Inline Comments:**
- Brief explanations of buffer size rationale:
  ```go
  // increase the buffer size to avoid running out of space
  scanBuf := make([]byte, 0, maxBufferSize)
  ```
- Document assumptions or constraints in error messages

## Function Design

**Size Guidelines:**
- Keep functions focused on a single responsibility
- Example: `checkError()` in `api/client.go` (5-15 lines) - single responsibility
- Example: `stream()` in `api/client.go` (50+ lines) - allows for complexity in streaming logic

**Parameters:**
- Use typed function parameters, avoid variadic where structure is better
- Use context.Context as first parameter when async/cancellation matters
- For callbacks, define function types at package level: `GenerateResponseFunc`, `ChatResponseFunc`
- Structs for request data when >2-3 parameters: `GenerateRequest`, `ChatRequest`

**Return Values:**
- Single return value for simple functions: `Host() *url.URL`, `Version(ctx context.Context) (string, error)`
- Pointer returns when zero value isn't valid: `ClientFromEnvironment() (*Client, error)`
- Error as last return value: `(result, error)` pattern throughout
- Multiple returns for structured data: `(ListResponse, error)`

## Module Design

**Exports:**
- Exported symbols (PascalCase) are primarily types and their constructors/methods
- Keep exported surface minimal; unexported helpers for internal logic
- Constructor functions follow `New*` pattern: `NewClient()`, or context-aware: `ClientFromEnvironment()`

**Barrel Files:**
- No barrel file exports observed in codebase
- Each package has focused purpose: `api/` exports API types, `cmd/` exports handlers, `format/` exports formatting utilities

**Package Boundaries:**
- `api/` - Client types, request/response structs, status errors
- `cmd/` - CLI command handlers for user-facing operations
- `convert/` - Model format conversion logic
- `discover/` - GPU/hardware discovery
- `format/` - Formatting utilities (numbers, time, bytes)
- `envconfig/` - Environment variable configuration
- `server/` - HTTP server routes and request handling
- `llm/` - Low-level model loading and GGML format
- `integration/` - Integration tests (marked with `//go:build integration` build tag)

## Constants and Variables

**Constants:**
- Use UPPERCASE_SNAKE_CASE for module-level constants: `Thousand`, `Million`, `Billion`, `maxBufferSize`
- Group related constants: numeric constants for format conversion
- Define constants at top of file before usage

**Package-level Variables:**
- Unexported test fixtures: `var stream = true` for reusable boolean in tests
- Error variables: `var errModelNotFound = errors.New("no Modelfile or safetensors files found")`
- Stateful test synchronization: `var serverMutex sync.Mutex`, `var serverReady bool`

## Testing Patterns

**Naming:**
- Test functions: `Test*` with clear test names: `TestHumanNumber`, `TestClientFromEnvironment`, `TestShowInfo`
- Table tests: `func Test*(t *testing.T)` with `cases` or `tests` slice
- Sub-tests: `t.Run(description, func(t *testing.T) { ... })`
- Test case struct fields: lowercase descriptive names (`name`, `value`, `expected`, `ok`, `fileExists`)

---

*Convention analysis: 2026-03-22*

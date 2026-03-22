# Testing Patterns

**Analysis Date:** 2026-03-22

## Test Framework

**Runner:**
- Go built-in `testing` package (no external test runner)
- Version: Go 1.23.4 (from `go.mod`)
- Config: No separate config file (uses Go standard)

**Assertion Library:**
- Built-in `testing.T` with `.Error()`, `.Fatal()`, `.Fatalf()`, `.Errorf()`
- `github.com/google/go-cmp/cmp` for deep equality assertions with diffs
- `github.com/stretchr/testify` for additional assertions and mocks

**Run Commands:**
```bash
go test ./...                 # Run all tests
go test ./package/...         # Run specific package tests
go test -run TestName ./...   # Run specific test function
go test -v ./...              # Verbose output with test names
go test -cover ./...          # Show coverage percentage
go test -coverprofile=cover.out ./...  # Generate coverage report
```

**Integration Tests:**
- Marked with build tag: `//go:build integration`
- Run with: `go test -tags=integration ./integration/...`
- Location: `integration/` directory contains integration tests

## Test File Organization

**Location:**
- Co-located with implementation: `filename.go` and `filename_test.go` in same directory
- Platform-specific tests: `filename_platform_test.go` (e.g., `gpu_linux_test.go`, `gpu_windows_test.go`)
- Integration tests: Separate `integration/` directory with `//go:build integration` tag
- Test utilities: `utils_test.go` in packages (e.g., `integration/utils_test.go`)

**Naming:**
- File pattern: `*_test.go`
- Function pattern: `Test*` (PascalCase after "Test")
- Examples: `TestClientFromEnvironment`, `TestHumanNumber`, `TestShowInfo`, `TestOrcaMiniBlueSky`

**Structure:**
```
api/
├── client.go
├── client_test.go           # Tests for client.go
├── types.go
└── types_test.go            # Tests for types.go

integration/
├── basic_test.go            # Integration tests (//go:build integration)
├── utils_test.go            # Shared test utilities
├── concurrency_test.go
└── context_test.go

discover/
├── gpu.go
├── gpu_test.go              # Main tests
├── gpu_linux_test.go        # Platform-specific tests
├── gpu_windows_test.go
└── gpu_linux.go
```

## Test Structure

**Suite Organization (Table-driven tests):**

```go
func TestHumanNumber(t *testing.T) {
    type testCase struct {
        input    uint64
        expected string
    }

    testCases := []testCase{
        {0, "0"},
        {1000000, "1M"},
        {125000000, "125M"},
    }

    for _, tc := range testCases {
        t.Run(tc.expected, func(t *testing.T) {
            result := HumanNumber(tc.input)
            if result != tc.expected {
                t.Errorf("Expected %s, got %s", tc.expected, result)
            }
        })
    }
}
```

**Patterns:**

1. **Table-driven tests with map of test cases:**
   - Location: `api/client_test.go`
   ```go
   testCases := map[string]*testCase{
       "empty":           {value: "", expect: "http://127.0.0.1:11434"},
       "only address":    {value: "1.2.3.4", expect: "http://1.2.3.4:11434"},
       "scheme and port": {value: "https://1.2.3.4:1234", expect: "https://1.2.3.4:1234"},
   }

   for k, v := range testCases {
       t.Run(k, func(t *testing.T) {
           // test implementation
       })
   }
   ```

2. **Slice-based table-driven tests:**
   - Location: `format/format_test.go`
   ```go
   cases := []struct {
       input    uint64
       expected string
   }{
       {0, "0"},
       {1000000, "1M"},
   }

   for _, tc := range cases {
       t.Run(tc.expected, func(t *testing.T) {
           // test implementation
       })
   }
   ```

3. **Sub-test organization with named tests:**
   - Location: `cmd/cmd_test.go`
   ```go
   t.Run("bare details", func(t *testing.T) {
       // nested test
   })

   t.Run("parameters", func(t *testing.T) {
       // another nested test
   })
   ```

4. **Setup and cleanup with helper functions:**
   - Location: `integration/utils_test.go`
   ```go
   func InitServerConnection(ctx context.Context, t *testing.T) (*api.Client, string, func()) {
       client, testEndpoint := GetTestEndpoint()
       // setup code
       return client, testEndpoint, func() {
           // cleanup logic
           defer serverProcMutex.Unlock()
       }
   }
   ```

## Mocking

**Framework:** Manual test doubles using `httptest` package

**Patterns:**

1. **Mock HTTP servers:**
   ```go
   // Location: cmd/cmd_test.go - TestDeleteHandler
   mockServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
       if r.URL.Path == "/api/delete" && r.Method == http.MethodDelete {
           var req api.DeleteRequest
           if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
               http.Error(w, err.Error(), http.StatusBadRequest)
               return
           }
           if req.Name == "test-model" {
               w.WriteHeader(http.StatusOK)
           } else {
               w.WriteHeader(http.StatusNotFound)
           }
           return
       }
   }))
   defer mockServer.Close()
   ```

2. **Server mocking with custom handlers:**
   ```go
   // Location: cmd/cmd_test.go - TestPushHandler
   serverResponse: map[string]func(w http.ResponseWriter, r *http.Request){
       "/api/push": func(w http.ResponseWriter, r *http.Request) {
           // handler implementation
           responses := []api.ProgressResponse{
               {Status: "preparing manifest"},
               {Digest: "sha256:abc123456789", Total: 100, Completed: 50},
           }

           for _, resp := range responses {
               if err := json.NewEncoder(w).Encode(resp); err != nil {
                   http.Error(w, err.Error(), http.StatusInternalServerError)
                   return
               }
               w.(http.Flusher).Flush()
           }
       },
   }
   ```

3. **Environment variable mocking:**
   ```go
   // Location: api/client_test.go
   t.Setenv("OLLAMA_HOST", v.value)

   // Location: cmd/cmd_test.go
   t.Setenv("OLLAMA_HOST", mockServer.URL)
   ```

4. **File system mocking with `fs.FS`:**
   ```go
   // Location: convert/tokenizer_test.go
   func createTokenizerFS(t *testing.T, dir string, files map[string]io.Reader) fs.FS {
       t.Helper()

       for k, v := range files {
           if err := func() error {
               f, err := os.Create(filepath.Join(dir, k))
               if err != nil {
                   return err
               }
               defer f.Close()

               if _, err := io.Copy(f, v); err != nil {
                   return err
               }
               return nil
           }(); err != nil {
               t.Fatalf("unexpected error: %v", err)
           }
       }

       return os.DirFS(dir)
   }
   ```

**What to Mock:**
- HTTP endpoints (use `httptest.NewServer`)
- External services (API clients)
- File I/O (create temporary directories with `t.TempDir()`)
- Environment variables (use `t.Setenv()`)

**What NOT to Mock:**
- Core business logic (test real implementations)
- Standard library functions (unless testing error paths)
- Internal helper functions (test indirectly through public APIs)

## Fixtures and Factories

**Test Data:**

1. **Inline test structs with field initialization:**
   ```go
   // Location: cmd/cmd_test.go
   if err := showInfo(&api.ShowResponse{
       Details: api.ModelDetails{
           Family:            "test",
           ParameterSize:     "7B",
           QuantizationLevel: "FP16",
       },
   }, &b); err != nil {
       t.Fatal(err)
   }
   ```

2. **Test data generation functions:**
   ```go
   // Location: integration/utils_test.go
   func GenerateRequests() ([]api.GenerateRequest, [][]string) {
       return []api.GenerateRequest{
           {
               Model:     "orca-mini",
               Prompt:    "why is the ocean blue?",
               Stream:    &stream,
               KeepAlive: &api.Duration{Duration: 10 * time.Second},
               Options: map[string]interface{}{
                   "seed":        42,
                   "temperature": 0.0,
               },
           },
       }, [][]string{
           {"sunlight"},
       }
   }
   ```

3. **Helper functions with t.Helper():**
   ```go
   // Location: convert/tokenizer_test.go
   func createTokenizerFS(t *testing.T, dir string, files map[string]io.Reader) fs.FS {
       t.Helper()  // Mark as test helper function
       // implementation
   }

   // Location: server/model_test.go
   func readFile(t *testing.T, base, name string) *bytes.Buffer {
       t.Helper()
       // implementation
   }
   ```

**Location:**
- Test data files: `testdata/` subdirectory (e.g., `server/testdata/tools/`)
- Fixture generation: Within test functions or helper functions in `*_test.go` files
- Reusable test utilities: `utils_test.go` in each package (e.g., `integration/utils_test.go`)

## Coverage

**Requirements:** Not enforced (no minimum coverage threshold found in config)

**View Coverage:**
```bash
go test -cover ./...                              # Show coverage percentage
go test -coverprofile=coverage.out ./...          # Generate coverage profile
go tool cover -html=coverage.out -o coverage.html # View HTML coverage report
```

**Coverage Patterns Observed:**
- Unit tests cover public APIs thoroughly
- Integration tests in separate directory with build tag
- Test coverage includes error cases (StatusError scenarios, not found cases)

## Test Types

**Unit Tests:**
- **Scope:** Single function or small unit of functionality
- **Examples:**
  - `TestHumanNumber` - `format/format_test.go` - tests number formatting
  - `TestClientFromEnvironment` - `api/client_test.go` - tests client initialization
  - `TestShowInfo` - `cmd/cmd_test.go` - tests info display formatting
- **Approach:**
  - Direct function calls with known inputs
  - Assert output matches expected values
  - Use mock HTTP servers for API client testing
  - Environment variable mocking with `t.Setenv()`

**Integration Tests:**
- **Scope:** Multiple components working together
- **Examples:**
  - `TestOrcaMiniBlueSky` - `integration/basic_test.go` - tests full generate workflow
  - `TestUnicode` - `integration/basic_test.go` - tests model with unicode tokenizer
  - `TestDeleteHandler` - `cmd/cmd_test.go` - tests delete with server interaction
- **Approach:**
  - Use `//go:build integration` build tag
  - Start real or mock server instances
  - Test full request/response cycles
  - Verify actual data processing (not mocked)
  - Use realistic test data (actual model names, prompts)
  - Long timeouts for real model loading: `context.WithTimeout(context.Background(), 2*time.Minute)`

**E2E Tests:** Not formally distinguished; integration tests serve this purpose

## Common Patterns

**Async Testing:**
```go
// Location: integration/utils_test.go
done := make(chan int)
var genErr error
go func() {
    genErr = client.Generate(ctx, &genReq, fn)
    done <- 0
}()

select {
case <-stallTimer.C:
    t.Errorf("timeout after %s", initialTimeout.String())
case <-done:
    require.NoError(t, genErr)
case <-ctx.Done():
    t.Error("outer test context done while waiting")
}
```

**Error Testing:**
```go
// Location: cmd/cmd_test.go
if err == nil || !strings.Contains(err.Error(), "expected error message") {
    t.Fatalf("DeleteHandler failed: expected error about stopping non-existent model, got %v", err)
}

// Using StatusError type assertion
var statusError api.StatusError
if errors.As(err, &statusError) && statusError.StatusCode == http.StatusNotFound {
    // handle not found
}
```

**Comparison with go-cmp:**
```go
// Location: format/format_test.go, convert/tokenizer_test.go
if diff := cmp.Diff(tt.expected, result); diff != "" {
    t.Errorf("unexpected output (-want +got):\n%s", diff)
}

// For deep struct comparison
if diff := cmp.Diff(tt.want, tokenizer); diff != "" {
    t.Errorf("unexpected tokenizer (-want +got):\n%s", diff)
}
```

**Temporary Directory Management:**
```go
// Location: convert/tokenizer_test.go
f, err := os.CreateTemp(t.TempDir(), "f16")
if err != nil {
    t.Fatal(err)
}
defer f.Close()

// Or with cleanup
fp, err := os.CreateTemp("", "ollama-server-*.log")
if err != nil {
    t.Fatalf("failed to generate log file: %s", err)
}
t.Cleanup(func() { r.Close() })
```

**Server Startup for Integration Tests:**
```go
// Location: integration/utils_test.go
func InitServerConnection(ctx context.Context, t *testing.T) (*api.Client, string, func()) {
    client, testEndpoint := GetTestEndpoint()
    if os.Getenv("OLLAMA_TEST_EXISTING") == "" {
        serverProcMutex.Lock()
        require.NoError(t, startServer(t, ctx, testEndpoint))
    }

    return client, testEndpoint, func() {
        if os.Getenv("OLLAMA_TEST_EXISTING") == "" {
            defer serverProcMutex.Unlock()
            if t.Failed() {
                // dump server log on failure
            }
        }
    }
}
```

---

*Testing analysis: 2026-03-22*
